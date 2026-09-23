# Job Ledger

Every machine on the network can now hear about each other ([Address Book](./address_book.md)) and every machine ends up with a copy of every uploaded job's file ([File Share](./file_share.md)). What's still missing is *coordination*: something has to make sure exactly one machine picks up any given job, rather than every idle, ready machine on the network all trying to print the same file at once. That's the job ledger — a single list of pending jobs, passed from machine to machine like a token, where only the current holder is allowed to act on it. Stage gate: **watch your machine create and hold the ledger on its own, then, with a second machine and a real upload, watch a job actually get dry-printed.**

## The ledger and its state

`kinds.rs` defines the ledger's shape (`tasks/messages.rs`'s `Ledger` — just an owner address and a set of job UUIDs) and wraps it with a timestamp of when it was last received:

```rust
pub struct LedgerState {
    pub ledger: Ledger,
    pub received: Instant,
}

impl LedgerState {
    pub fn is_owner(&self) -> bool {
        let Some(my_addr) = my_ipaddr() else { return false; };
        my_addr == self.ledger.owner
    }
}

pub type JobLedger = Mutex<ThreadModeRawMutex, Option<LedgerState>>;
```

It's `Option`-wrapped because when a machine first boots, nobody has the ledger yet — that's the first thing `manage_ledger` has to resolve.

## Getting a job onto the ledger

Back in [File Upload](./upload.md), we left a placeholder in `handle_conn` for a genuinely new upload. [File Share](./file_share.md) already filled in half of it — pushing the file out to every peer. Here's the other half, completing the block exactly as it appears in `tasks/tcp.rs`:

```rust
if info.guid.is_none() {
    append_to_ledger(guid, address_book, ledger, udp).await;
    distribute_file(guid, address_book, udp).await;
}
```

`append_to_ledger` is the piece that actually gets a job onto the ledger — updating it directly if this machine already owns it, or alerting the network otherwise:

```rust
async fn append_to_ledger<const N1: usize, const N2: usize>(
    guid: Uuid,
    address_book: &AddressBook<N1>,
    ledger: &JobLedger,
    udp: &UdpSocket<N2>,
) {
    let address_book_is_empty = address_book.lock().await.is_empty();

    let is_owner = {
        let mut guard = ledger.lock().await;
        if let Some(state) = guard.as_mut()
            && let Some(addr) = my_ipaddr()
            && addr == state.ledger.owner
        {
            let _ = state.ledger.jobs.insert(guid);
            true
        } else {
            false
        }
    };

    if !is_owner && !address_book_is_empty {
        Message::send_new_job_alert(guid, udp).await;
    }
}
```

If the machine you uploaded to already owns the ledger, the job is simply added to it directly. Otherwise, it broadcasts a `NewJob` alert instead — and whichever machine *does* own the ledger picks that up over in `udp_receiver`:

```rust
if let Payload::NewJob(data) = msg.payload {
    let mut guard = ledger.lock().await;
    if let Some(ledger) = guard.as_mut() {
        let _ = ledger.ledger.jobs.insert(data.guid);
    }
    continue;
}

if let Payload::Ledger(sent_ledger) = msg.payload {
    let mut l = ledger.lock().await;
    *l = Some(LedgerState::new(sent_ledger));
    continue;
}
```

Receiving a full `Ledger` payload (rather than just a `NewJob` alert) is how the token itself moves from machine to machine — whoever sends one is handing ownership to whoever's listening.

## `manage_ledger`: the token-ring loop

This is the task that actually drives everything — creating the ledger if none exists, printing when it's this machine's turn, and passing ownership on:

```rust
#[embassy_executor::task(pool_size = 1)]
pub async fn manage_ledger(
    udp: &'static UdpSocket<UDP_CHANNEL_SIZE>,
    address_book: &'static AddressBook<ADDRESS_BOOK_ENTRIES>,
    ledger: &'static JobLedger,
) -> ! {
    Timer::after_secs(20).await; // let the address book populate first
    loop {
        Timer::after_secs(5).await;
        let Some(my_addr) = my_ipaddr() else { continue; };
        let book_guard = address_book.lock().await;
        let mut ledger_guard = ledger.lock().await;

        if ledger_guard.is_none() {
            log_info!("Creating new ledger");
            let ledger = Ledger { owner: my_addr, jobs: FnvIndexSet::new() };
            let state = LedgerState::new(ledger);
            *ledger_guard = Some(state);
            if let Some(ledger_state) = ledger_guard.as_mut() {
                Message::send_ledger(ledger_state.ledger.clone(), udp).await;
            };
            continue;
        }

        let Some(ledger_state) = ledger_guard.as_mut() else { continue; };

        // Take over a stale ledger whose owner has gone quiet.
        if ledger_state.received.elapsed() > Duration::from_secs(45)
            && !book_guard.contains_key(&ledger_state.ledger.owner)
        {
            if let Some(min_addr) = book_guard.keys().min() {
                if *min_addr == my_addr {
                    ledger_state.ledger.owner = my_addr;
                }
            } else {
                ledger_state.ledger.owner = my_addr;
            };
        }

        // If it's my turn, and I'm ready and idle, print something.
        if ledger_state.is_owner() && marlin::is_ready() && marlin::is_idle() {
            let printable_job = ledger_state.ledger.jobs.iter().find_map(|&guid| {
                let path = make_path(&guid, false);
                if fs::open(&path, ReadBytes).is_ok() {
                    Some((guid, path))
                } else {
                    None
                }
            });
            if let Some((guid, path)) = printable_job {
                match marlin::print(&path, true) {
                    Ok(_) => {
                        marlin::set_offline();
                        ledger_state.ledger.jobs.remove(&guid);
                        let msg = format!("Job Accepted: {guid}");
                        Message::send_log(&msg, udp).await;
                    }
                    Err(e) => log_error!("Print Error: {e}"),
                }
            }
        }

        // Pass the ledger on to the next machine (by IP), or keep it if alone.
        if ledger_state.is_owner() {
            if book_guard.is_empty() {
                log_info!("It's only me - keeping ledger - and telling everyone.");
                Message::send_ledger(ledger_state.ledger.clone(), udp).await;
                continue;
            }
            let mut next_highest = u8::MAX;
            let mut next_addr = Ipv4Addr::new(255, 255, 255, 255);
            for addr in book_guard.keys() {
                let diff = addr.octets()[3].saturating_sub(my_addr.octets()[3]);
                if diff > 0 && diff < next_highest {
                    next_addr = *addr;
                    next_highest = diff;
                }
            }
            if next_highest > 0 && next_highest < u8::MAX {
                ledger_state.ledger.owner = next_addr;
            } else {
                // No higher IP found — wrap around to the lowest IP in the book.
                let min_addr = book_guard.keys().min().unwrap();
                ledger_state.ledger.owner = *min_addr;
            }

            Message::send_ledger(ledger_state.ledger.clone(), udp).await;
        }
    }
}
```

A few things worth calling out:

- **Bootstrapping**: the *first* machine to have its `manage_ledger` timer fire with no ledger present creates one, owns it, and broadcasts it. Any other machine that later boots and sees a `Ledger` message arrive over UDP just adopts it via `udp_receiver`'s `Payload::Ledger` branch above — it never tries to create its own.
- **Passing the token**: ownership moves to the *next-highest* IP address in the address book each round, wrapping back around to the lowest once you reach the top — a simple, fully decentralised way to give every machine a turn without anyone keeping a global list of "whose turn is it".
- **Failover**: if the current owner goes quiet (missing from the address book, and the ledger hasn't been refreshed in 45s), the machine with the lowest IP in the book takes over — so one machine dropping off the network doesn't strand the ledger forever.
- **Printing is gated three ways**: owning the ledger isn't enough on its own — [Marlin](./marlin.md)'s `is_ready()` (the technician toggled the machine `ONLINE`) and `is_idle()` (it's not already printing something else) both have to hold too, and `set_offline()` is called immediately after a successful print so the machine won't grab a second job mid-print.

## Wiring it into `embassy_main`

`manage_ledger` is spawned alongside the other UDP tasks:

```rust
static JOB_LEDGER: JobLedger = AsyncMutex::new(None);

match manage_ledger(udp, address_book, ledger) {
    Ok(t) => spawner.spawn(t),
    Err(e) => log_error!("Spawn Error: {e}"),
}
```

## Did it work?

**On your own**, flash a single machine and wait. After the initial 20-second settle period, you should see:

```
[INFO  - derusting:0] Creating new ledger
[INFO  - derusting:0] It's only me - keeping ledger - and telling everyone.
```

with the second line repeating every 5 seconds after that — proof the token-ring logic is alive and correctly recognises it has no peers to hand off to yet.

**With a second machine**, both on the network and both showing each other in their address book: one of them will create the ledger first (whichever's `manage_ledger` timer fires first), broadcast it, and the other will silently adopt it via `udp_receiver`. Watch the log on both sides — every 5 seconds you should see the ledger's ownership alternate between the two IPs as `Message::send_ledger` broadcasts hand it back and forth.

**The full end-to-end test**: toggle both machines `ONLINE` on their home screens (see [Marlin](./marlin.md)), then upload a `.gcode` file through either one's web form. Watch:

1. [File Share](./file_share.md)'s log lines as the file gets distributed.
2. The job appearing in the ledger's `jobs` set (via `append_to_ledger` or a `NewJob` broadcast).
3. Whichever machine happens to own the ledger *and* is ready *and* is idle picking the job up and logging `Gcode from Rust: M32 /usb/<guid>.gcode` — the printer should visibly home and start a dry run. That machine also broadcasts `Log("Job Accepted: <guid>")` over UDP the moment the print starts successfully, visible via [Monitoring](./monitoring.md).

That's the whole system working together: an upload on one machine ends up dry-printed on whichever machine's turn it was, with no central coordinator anywhere in the loop.
