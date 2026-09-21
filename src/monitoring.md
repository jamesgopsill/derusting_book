# Monitoring

[Getting Started](./getting_started.md) already promised this: if you want to see what machines are actually saying to each other on the network, you don't need a second Buddy board or a debug probe — an ordinary PC on the same LAN can listen in on the UDP gossip protocol from [Gossiping](./gossip.md) directly. This chapter builds that listener: a small standalone Rust binary that decodes real broadcasts into readable heartbeats, job alerts, and ledger handoffs.

This is exactly what [Getting Started](./getting_started.md) calls the "derusting udp listener" — here's how to build one.

## Project setup

```bash
cargo new monitoring_derusting
cd monitoring_derusting
```

```toml
[dependencies]
postcard = "1"
serde = { version = "1", features = ["derive"] }
uuid = { version = "1", features = ["serde", "v7"] }
```

## Mirroring the wire format exactly

Every message in [Address Book](./address_book.md) is encoded with [`postcard`](https://docs.rs/postcard), a compact *binary* format — there are no field names in the packets, just tightly packed bytes. To decode one, the listener needs its own copy of `tasks/messages.rs`'s types and `kinds.rs`'s `Ledger` — not because the code is shared, but because `postcard` needs both ends of the wire to agree on the exact same shape:

```rust
// src/messages.rs
use std::net::Ipv4Addr;
use std::collections::HashSet;

use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize)]
pub struct Heartbeat {
    pub alive: bool,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct NewJob {
    pub guid: Uuid,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct Ledger {
    pub owner: Ipv4Addr,
    pub jobs: HashSet<Uuid>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct Message {
    pub idempotency: Uuid,
    pub payload: Payload,
}

#[derive(Debug, Serialize, Deserialize)]
pub enum Payload {
    Heartbeat(Heartbeat),
    NewJob(NewJob),
    Ledger(Ledger),
}
```

Two details matter more than they look like they should:

- **Enum variant order is the wire format.** `postcard` (like most compact binary serde formats) encodes an enum variant as its *index* — `Heartbeat` is 0, `NewJob` is 1, `Ledger` is 2 — not its name. Get this listener's `Payload` enum out of sync with the firmware's (reorder a variant, insert a new one in the middle, whatever) and you'd silently deserialise a `Heartbeat` packet as a `NewJob` — or fail outright — despite every field being spelled correctly. Keep it byte-for-byte in step with `tasks/messages.rs`.
- **The container type doesn't have to match, only its serialised shape.** The firmware's `Ledger.jobs` is a `heapless::FnvIndexSet<Uuid, 8>` (fixed-capacity, since it's running on a microcontroller); here it's a plain `std::collections::HashSet<Uuid>`. Both serialise via `serde` as a length-prefixed sequence of elements, so `postcard` doesn't care which one is on which end.

## The listener itself

```rust
// src/main.rs
use std::net::UdpSocket;

mod messages;
use messages::Message;

const PORT: u16 = 9090;

fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind(("0.0.0.0", PORT))?;
    println!("Listening on UDP {PORT}...");

    let mut buf = [0u8; 1024];
    loop {
        let (len, src) = socket.recv_from(&mut buf)?;
        match postcard::from_bytes::<Message>(&buf[..len]) {
            Ok(msg) => println!("[{src}] {msg:?}"),
            Err(e) => eprintln!("[{src}] {len} bytes, failed to decode: {e}"),
        }
    }
}
```

No `set_broadcast(true)` needed here — that flag controls whether a socket is allowed to *send* to the broadcast address, not whether it can *receive* broadcast traffic sent to it, so a plain bound listener picks up every machine's heartbeats without it.

## Did it work?

Run the listener on a PC connected to the same network as a flashed machine (see [Prerequisites](./prerequisites.md)'s note on the ethernet-cable-and-link-local setup):

```bash
cargo run
```

Within a couple of seconds you should see a `Heartbeat` arriving on a steady 2-second cadence (see [LWIP UDP](./lwip_udp.md)):

```
Listening on UDP 9090...
[192.168.x.x:9090] Message { idempotency: 0197..., payload: Heartbeat(Heartbeat { alive: true }) }
[192.168.x.x:9090] Message { idempotency: 0197..., payload: Heartbeat(Heartbeat { alive: true }) }
```

Upload a job through the machine's portal (see [Submitting Jobs](./job_submission.md)) and you should see a burst of `NewJob` or `Ledger` messages appear alongside the heartbeats — five repeats each, a couple of hundred milliseconds apart (see [Address Book](./address_book.md) for why). With two machines on the network, you'll also see the `Ledger`'s `owner` field alternate between their two IP addresses as [Job Ledger](./ledger.md)'s token-ring logic hands it back and forth.

> [!TIP]
> If nothing arrives at all, double-check your PC's firewall isn't silently dropping inbound UDP on port 9090 — this trips people up more often than anything actually being wrong with the machine.
