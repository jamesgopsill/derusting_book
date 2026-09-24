# Binding LWIP UDP

With an Embassy task up and running (see [Embassy Async Runtime in FreeRTOS](./embassy.md)), we can start giving it something useful to do. The first network primitive derusting needs is UDP: every machine on the network periodically shouts "I'm alive" so the others can build up an address book. This chapter binds a UDP socket and gets that heartbeat printing to the log every couple of seconds — our stage gate here is simple: **see our own machine talking on the wire**.

## Why not a Rust network stack?

The Buddy firmware already embeds [lwIP](https://www.nongnu.org/lwip/) as its TCP/IP stack — it's what the rest of the firmware (web UI, MQTT, mDNS) uses. Rather than bring in a second, independent stack (e.g. `smoltcp`) and fight the firmware for the network interface, we bind directly to lwIP's own C API from Rust.

lwIP's "raw" API is callback-driven: you call `udp_new`/`udp_bind`, register a `udp_recv` callback, and lwIP invokes that callback (on its own thread, with its own internal lock held) whenever a datagram arrives. That's a very C-shaped API, so the `lwip` module's job is to wrap it into something an `async fn` can `.await`.

## The `lwip` module

```
/lwip
  - bindings.rs      # extern "C" lwIP bindings (tcp_*, udp_*, pcb, err_t, ...)
  - ipaddr.rs
  - packet_buffer.rs # a safe wrapper around lwIP's `pbuf` (packet buffer)
  - put.rs           # outbound HTTP PUT client (used later, for job sharing)
  - tcp.rs
  - udp.rs
  - mod.rs
```

### Bridging lwIP's callbacks into async Rust

`mod.rs` gives us two helpers used throughout the `lwip` module:

- `blocking_lwip` — runs a closure with lwIP's core lock held. Used for anything that must happen immediately (and is already safe to block on), like tearing a socket down.
- `async_lwip` — schedules a closure to run on lwIP's own thread via `tcpip_callback`, then `.await`s an `embassy_sync::Signal` that the callback fires once it's done. This is how `bind()`/`broadcast()` below can be `async fn`s despite lwIP itself being callback-based.

```rust
pub(super) async fn async_lwip<F>(closure: F) -> Result<(), err_t>
where
    F: FnMut() -> Result<(), err_t>,
{
    unsafe extern "C" fn _tcpip_async_callback<F>(ctx: *mut c_void)
    where
        F: FnMut() -> Result<(), err_t>,
    {
        let ctx = unsafe { &mut *(ctx as *mut ThreadContext<F>) };
        let res = (ctx.closure)();
        ctx.signal.signal(res);
    }

    let mut ctx = ThreadContext { closure, signal: Signal::new() };
    let ctx_ptr = &mut ctx as *mut _ as *mut c_void;
    let err = unsafe { tcpip_callback(_tcpip_async_callback::<F>, ctx_ptr) };
    if err != err_t::Ok {
        return Err(err);
    }
    ctx.signal.wait().await
}
```

`mod.rs` also gives us `my_ipaddr()`, which reads the current IP off lwIP's default network interface (or `None` before DHCP has assigned one) — we'll need it below.

### `UdpSocket`

`udp.rs` wraps a single UDP `pcb` (lwIP's "protocol control block") behind a small, generic-over-channel-size type:

```rust
pub struct UdpSocket<const N: usize> {
    pcb: AtomicPtr<pcb>,
    channel: Channel<CriticalSectionRawMutex, (Ipv4Addr, PacketBuffer), N>,
}

impl<const N: usize> UdpSocket<N> {
    pub fn new() -> Self {
        Self {
            pcb: AtomicPtr::new(core::ptr::null_mut()),
            channel: Channel::new(),
        }
    }

    pub async fn bind(&'static self, port: u16) -> Result<(), err_t> {
        super::async_lwip(|| unsafe {
            let pcb = udp_new();
            if pcb.is_null() {
                return Err(err_t::Mem);
            }
            let err = udp_bind(pcb, &ip_addr_any, port);
            if err == err_t::Ok {
                udp_recv(pcb, Some(Self::_recv), self.as_mut_ptr());
                self.pcb.store(pcb, Ordering::Release);
                Ok(())
            } else {
                udp_remove(pcb);
                Err(err)
            }
        })
        .await
    }

    pub async fn broadcast(&self, mut pbuf: PacketBuffer, port: u16) -> Result<(), err_t> {
        super::async_lwip(|| unsafe {
            let pcb = self.pcb.load(Ordering::Acquire);
            if pcb.is_null() {
                return Err(err_t::Val);
            }
            let addr: ip_addr_t = ip_addr_t { addr: u32::MAX }; // broadcast
            let err = udp_sendto(pcb, pbuf.as_mut_ptr(), &addr, port);
            if err == err_t::Ok { Ok(()) } else { Err(err) }
        })
        .await
    }
}
```

Incoming datagrams arrive via `_recv`, lwIP's callback, which just forwards the sender's address and the packet onto the socket's internal channel — the async side picks it up with `receive().await`. We'll make use of `receive()` properly once we cover message parsing in the [Gossiping](./gossip.md) chapter; for this stage gate we only need `bind()` and `broadcast()`.

## Wiring it into `embassy_main`

Rather than give every task its own separate `static`, derusting collects everything shared across tasks into one struct, `DerustingState`, and hands every task a single `&'static DerustingState` reference instead of its own bespoke set of arguments. `udp` is the first field it needs:

```rust
pub const UDP_PORT: u16 = 9090;
pub const UDP_CHANNEL_SIZE: usize = 12;

pub struct DerustingState {
    udp: UdpSocket<UDP_CHANNEL_SIZE>,
    // ...more fields added by later chapters, as we need them...
}

impl DerustingState {
    fn new() -> Self {
        Self {
            udp: UdpSocket::new(),
        }
    }
}

static DERUSTING_STATE: StaticCell<DerustingState> = StaticCell::new();

async fn embassy_main(spawner: Spawner) {
    let state = DerustingState::new();
    let state = DERUSTING_STATE.init(state);

    if state.udp.bind(UDP_PORT).await.is_err() {
        log_critical!("UDP failed");
        return;
    };
    log_info!("UDP up on {UDP_PORT}");

    // Wait for an IP address before broadcasting anything.
    loop {
        if my_ipaddr().is_some() {
            break;
        };
        Timer::after_secs(1).await;
    }

    match heartbeat(state) {
        Ok(t) => spawner.spawn(t),
        Err(e) => log_error!("Spawn Error: {e}"),
    }
}
```

Every later chapter that needs a new piece of shared state — the address book, the job ledger, the TCP listener — adds a field to `DerustingState` rather than declaring another standalone `static`, and every task added from here on takes `state: &'static DerustingState` as its one parameter.

Then, in `tasks/udp.rs`, the `heartbeat` task itself — this is deliberately the simplest task in the whole codebase, and it's our stage gate:

```rust
#[embassy_executor::task(pool_size = 1)]
pub async fn heartbeat(state: &'static DerustingState) {
    loop {
        if let Some(addr) = my_ipaddr() {
            log_info!("[{:?}] heartbeat()", addr);
        } else {
            log_info!("[Unknown] heartbeat()");
        }
        Message::send_heartbeat(&state.udp).await;
        Timer::after_secs(2).await;
    }
}
```

`Message::send_heartbeat` serialises a small `Heartbeat` payload and calls `state.udp.broadcast(..)` — the actual message format (`postcard`-encoded, with an idempotency UUID) is covered in [Gossiping](./gossip.md), since it only matters once another machine is listening and parsing it. For now, the log line is enough proof that the socket is live and broadcasting.

## Did it work?

Rebuild, flash, and connect your machine to the network:

```bash
bash scripts/build_and_flash.sh
```

Once DHCP assigns an address you should see:

```
[INFO  - derusting:0] UDP up on 9090
[INFO  - derusting:0] [192.168.x.x] heartbeat()
[INFO  - derusting:0] [192.168.x.x] heartbeat()
...
```

repeating every 2 seconds, with your machine's real IP address in the brackets.

If you want to see the broadcast on the wire rather than just trust the log, run a UDP listener on a PC on the same network and point it at port `9090` — see [Getting Started](./getting_started.md) for a pointer to the derusting UDP listener tool.

> [!TIP]
> You may need to open port `9090`/UDP in your PC's firewall to see the broadcast arrive from the listener tool.
