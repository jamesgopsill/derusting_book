# Address Book

This chapter builds the shared foundation the rest of [Gossiping](./gossip.md) sits on: a message envelope broadcast over UDP, a receive loop that deduplicates and dispatches whatever arrives, and the address book itself — the list of peers each machine has heard from recently. Stage gate: **see your machine log a received packet from another machine, and know it's tracking who's on the network.**

The same envelope and receive loop also carry two more message kinds — a new-job announcement and a job-ledger handoff — covered in the next two chapters, [File Share](./file_share.md) and [Job Ledger](./ledger.md), once there's an actual job (from [Submitting Jobs](./job_submission.md)) to gossip about.

## One envelope, four payloads

Every message gossiped between machines — a heartbeat, a new-job announcement, a ledger handoff, or a broadcast log line — is wrapped in the same envelope, `tasks/messages.rs`'s `Message`:

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct Message<'a> {
    pub idempotency: Uuid,
    #[serde(borrow)]
    pub payload: Payload<'a>,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum Payload<'a> {
    Heartbeat(Heartbeat),
    NewJob(NewJob),
    Ledger(Ledger),
    #[serde(borrow)]
    Log(&'a str),
}
```

`Payload::Log` is the odd one out: unlike the other three, it borrows its string straight out of the incoming UDP packet's buffer rather than owning a copy, so `Message` and `Payload` both need a lifetime parameter tying them to that buffer.

The `idempotency` field is a UUIDv7 (see [Marlin](./marlin.md)'s neighbour, `tasks/rng.rs`) generated fresh for every message — its only job is letting a receiver recognise "I've already seen this one" when the same message arrives more than once.

UDP is unreliable, so rather than trust a single broadcast to arrive, `Message::send` fires the same message several times with a short gap between each:

```rust
async fn send<const N: usize>(udp: &UdpSocket<N>, msg: Self, repeats: usize) {
    for _ in 0..repeats {
        {
            let Some(pbuf) = PacketBuffer::alloc(&msg) else { return; };
            let _ = udp.broadcast(pbuf, UDP_PORT).await;
        }
        Timer::after_millis(100).await
    }
}
```

`PacketBuffer::alloc` serialises the message with [`postcard`](https://docs.rs/postcard) (a compact, `no_std`-friendly binary format) straight into a freshly allocated lwIP `pbuf`:

```rust
pub fn alloc<T: Serialize>(msg: T) -> Option<Self> {
    let size = postcard::serialize_with_flavor(&msg, Size::default()).ok()?;
    let pbuf = blocking_lwip(|| unsafe {
        Ok(pbuf_alloc(pbuf_layer::Transport, size as u16, pbuf_type::Ram))
    }).unwrap();
    // ...
    let payload = unsafe { core::slice::from_raw_parts_mut((*pbuf).payload, size) };
    postcard::to_slice(&msg, payload).ok()?;
    Some(Self { inner: pbuf, current: pbuf })
}
```

`heartbeat` (already introduced in [LWIP UDP](./lwip_udp.md)) is the simplest caller — `Message::send_heartbeat` wraps a one-field `Heartbeat { alive: true }` and sends it once every 2 seconds. `NewJob` and `Ledger` messages ([File Share](./file_share.md) and [Job Ledger](./ledger.md) respectively) are sent 3 times in a row instead of once, since losing one of those matters more than losing a heartbeat.

A fourth sender, `Message::send_log`, broadcasts an arbitrary `&str` (repeated twice) instead of a fixed struct — it's how a machine makes one of its own internal log lines visible to the rest of the network, not just its own local serial console. [File Share](./file_share.md) and [Job Ledger](./ledger.md) both call it at a few key points; [Monitoring](./monitoring.md) is where those broadcasts actually get decoded and printed somewhere you can read them.

## Receiving, deduplicating, and updating the address book

`tasks/udp.rs`'s `udp_receiver` task is the mirror image: it loops forever, deserialises whatever arrives, drops anything it's already processed recently, and otherwise records the sender in the address book before dispatching on the payload:

```rust
#[embassy_executor::task(pool_size = 1)]
pub async fn udp_receiver(state: &'static DerustingState) {
    log_info!("Ready to receive UDP packets");
    let mut history: HistoryBuf<Uuid, 24> = HistoryBuf::new();
    loop {
        let (addr, packet) = state.udp.receive().await;
        log_info!("Received packet from: {addr}");

        let Some(msg) = packet.into_iter().next() else { continue; };
        let Ok(msg) = postcard::from_bytes::<Message>(msg) else {
            log_error!("Packet Deserialization failed");
            continue;
        };

        if history.contains(&msg.idempotency) {
            continue; // Already processed it recently
        }
        history.write(msg.idempotency);

        {
            let mut book = state.addresses.lock().await;
            if let Err(e) = book.insert(addr, Instant::now()) {
                log_error!("Address book error: {e:?}");
            };
        }

        // ...dispatch on msg.payload (NewJob/Ledger) — covered in the
        // next two chapters...
    }
}
```

Every message that passes the idempotency check — regardless of its payload — updates the address book. A `Heartbeat` doesn't need any further handling beyond that; it exists purely to keep a peer's entry alive.

The address book itself is a small fixed-capacity map from IP address to "when we last heard from them", defined in `kinds.rs`:

```rust
pub type AddressBook<const N: usize> =
    Mutex<ThreadModeRawMutex, LinearMap<Ipv4Addr, Instant, N>>;
```

and a second task, `address_book_lifetime_check`, prunes any entry that's gone quiet — if a machine stops heartbeating (powered off, unplugged, network dropped), it falls out of everyone else's address book within 20 seconds:

```rust
#[embassy_executor::task(pool_size = 1)]
pub async fn address_book_lifetime_check(state: &'static DerustingState) {
    loop {
        Timer::after_secs(10).await;
        let mut book = state.addresses.lock().await;
        book.retain(|_k, instant| instant.elapsed().as_secs() < 20);
    }
}
```

## Wiring it into `embassy_main`

`addresses` joins `udp` as a field on the shared `DerustingState` introduced back in [LWIP UDP](./lwip_udp.md):

```rust
pub struct DerustingState {
    addresses: AddressBook<ADDRESS_BOOK_ENTRIES>,
    udp: UdpSocket<UDP_CHANNEL_SIZE>,
    // ...more fields added by later chapters...
}

impl DerustingState {
    fn new() -> Self {
        Self {
            addresses: AsyncMutex::new(LinearMap::new()),
            udp: UdpSocket::new(),
            // ...
        }
    }
}
```

Both new tasks are then spawned alongside `heartbeat`, once we have an IP address, all taking the same `state`:

```rust
match heartbeat(state) {
    Ok(t) => spawner.spawn(t),
    Err(e) => log_error!("Spawn Error: {e}"),
}
match address_book_lifetime_check(state) {
    Ok(t) => spawner.spawn(t),
    Err(e) => log_error!("Spawn Error: {e}"),
}
match udp_receiver(state) {
    Ok(t) => spawner.spawn(t),
    Err(e) => log_error!("Spawn Error: {e}"),
}
```

`udp_receiver` also touches the job ledger once it exists (two of the three payload kinds affect it — that part is covered in [Job Ledger](./ledger.md)), via `state.jobs` the same way it reaches for `state.addresses` above. For this chapter you can ignore that part.

## Did it work?

**On your own**, flashing a single machine only gets you as far as:

```
[INFO  - derusting:0] Ready to receive UDP packets
```

which just confirms the receive loop started — there's nothing to receive yet, since a machine's own broadcast isn't delivered back to itself.

**With a second Buddy board** on the same network, both running derusting, each one's log should start showing the other's heartbeats arriving:

```
[INFO  - derusting:0] Received packet from: 192.168.x.y
```

every couple of seconds, one line per heartbeat repeat from the other machine. That's the address book being populated in real time — each machine now has a live, self-maintaining record of who else is on the network, entirely from gossip.

> [!TIP]
> The book's actual contents aren't logged anywhere by default. If you want to see it directly rather than infer it from the "Received packet from" lines, add a temporary `log_info!("Address book: {:?}", address_book.lock().await.keys())`-style line somewhere convenient (e.g. inside `address_book_lifetime_check`'s loop) while you're getting familiar with the system.
