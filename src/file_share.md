# File Share

An uploaded job is only useful to the machine it was uploaded to unless every other machine on the network has a copy too — otherwise whichever machine ends up owning the job ledger might not have the file it's meant to print. This chapter covers how a freshly uploaded `.gcode` file — landed on disk by [File Upload](./upload.md)'s `handle_conn` — gets pushed out to every peer in the [Address Book](./address_book.md), using [LWIP TCP](./lwip_tcp.md)'s primitives as an outbound client this time instead of a server. Stage gate: **upload a job to one machine, then find it's appeared on a second machine's USB stick without uploading it there yourself.**

## Where sharing starts: a successful upload

Picking up right where [File Upload](./upload.md) left off — recall the placeholder comment left in `handle_conn`'s `PUT` branch for a genuinely new upload. The first thing that goes there is pushing the file out to every peer:

```rust
if info.guid.is_none() {
    // Updating the job ledger too is covered next, in Job Ledger.
    if state.files_to_distribute.try_send(guid).is_err() {
        state.log_to_udp("Error sending on channel").await;
    };
}
```

The `info.guid.is_none()` check is what stops this from looping forever: a genuinely new upload from a browser carries no `guid` header, so its `PutInfo::guid` is `None` and a fresh UUID is minted for it. But when *this* machine forwards the file on to a peer (below), it includes that UUID as a `guid:` header — so when the peer receives it, `info.guid` is `Some(..)`, and it knows not to distribute the file any further itself. That recognition is also broadcast over UDP as a `Log` message — `"Receiving file from machine"` — so watching [Monitoring](./monitoring.md) on a third device lets you tell an inbound share apart from a genuinely new upload, even though both look identical in the local serial log (`/ PUT request received` either way).

## Pushing the file out

PUTting to every peer synchronously from inside `handle_conn` would leave the uploader's browser waiting on however many peer connections there are before it gets its own response back. Instead, `distribute_file` is its own long-running task, decoupled from the upload handler by a small channel — `files_to_distribute` — that lives on `DerustingState` alongside everything else:

```rust
pub struct DerustingState {
    // ...
    files_to_distribute: Channel<ThreadModeRawMutex, Uuid, 4>,
}
```

`handle_conn` (above) just drops a `guid` onto that channel and moves on; `distribute_file` sits in a loop pulling guids back off it and doing the actual work, one job at a time, walking the current address book and PUTting the file to every peer in it:

```rust
#[embassy_executor::task(pool_size = 1)]
pub async fn distribute_file(state: &'static DerustingState) {
    loop {
        let guid = state.files_to_distribute.receive().await;
        let addrs = state.addresses.lock().await.clone();
        for (addr, _v) in addrs {
            let msg = format!(64; "PUT {guid} to {addr}:{TCP_PORT}").unwrap();
            state.log_to_udp(&msg).await;
            if let Err(err) = put_file(guid, addr, TCP_PORT).await {
                let err = format!(64; "Put Error: {err}").unwrap();
                state.log_to_udp(&err).await;
            };
        }
    }
}
```

It clones the address book first rather than holding the lock while it PUTs to (potentially several) peers, since each PUT can take a while and we don't want to block `udp_receiver` or `heartbeat` from updating the book in the meantime.

Neither the per-peer progress line nor the error, if `put_file` fails, are printed to this machine's own local log any more — both go out purely as `Message::send_log` broadcasts, via the `state.log_to_udp` helper. If you want to watch a distribution happen, [Monitoring](./monitoring.md) is where to look, not the serial console.

## Wiring it into `embassy_main`

`distribute_file` is spawned once, like every other background task:

```rust
match distribute_file(state) {
    Ok(t) => spawner.spawn(t),
    Err(e) => log_error!("Spawn Error: {e}"),
}
```

## The outbound PUT client

`lwip/put.rs`'s `put_file` is the client-side counterpart to the server we built in [LWIP TCP](./lwip_tcp.md) — same raw lwIP TCP API, but this time *we* initiate the connection (`tcp_connect`) instead of accepting one:

```rust
pub async fn put_file(guid: Uuid, addr: Ipv4Addr, port: u16) -> Result<(), err_t> {
    let Ok(put) = PutRequest::new(guid, addr, port) else {
        return Err(err_t::Val);
    };
    let put = pin!(put);
    put.send().await
}
```

`PutRequest` opens the job's file for reading, and streams it to the peer in fixed-size chunks as the connection's send buffer allows — refilling from the file whenever lwIP's `_sent` callback tells us there's room for more:

```rust
pub struct PutRequest {
    guid: Uuid,
    addr: Ipv4Addr,
    port: u16,
    fsize: u32,
    fh: File<ReadBytes>,
    buf: Vec<u8, BUF_CAP>, // BUF_CAP = 1024
    // ...
}
```

Once connected, it writes the HTTP request line and headers — including the `guid:` header that's the whole reason the receiving end knows not to re-share this file itself — before streaming the file body:

```rust
unsafe extern "C" fn _connected(arg: *mut c_void, pcb: *mut pcb, err: err_t) -> err_t {
    // ...
    let headers = format!(
        "PUT / HTTP/1.1\r\n\
guid: {}\r\n\
Content-Type: text/x.gcode\r\n\
Content-Length: {}\r\n\
Connection: close\r\n\r\n",
        this.guid, this.fsize
    );
    let _ = this.buf.extend_from_slice(headers.as_bytes());
    unsafe { this.pump(pcb) };
    unsafe { tcp_output(pcb) };
    err_t::Ok
}
```

Each subsequent `_sent` callback calls `pump` again, which tops the staging buffer back up from the file and writes whatever the peer's `tcp_sndbuf` currently has room for — so the whole file streams across without ever needing to hold it all in memory at once.

## Did it work?

You'll need two machines on the network for this one, both flashed with everything up to and including this chapter, both showing each other in their address book (see [Address Book](./address_book.md)'s "did it work" section).

Upload a `.gcode` file through machine A's web form (`http://[A's IP]:8080`, from [Submission Portal](./portal.md) and [File Upload](./upload.md)). In A's local log you should see:

```
[INFO  - derusting:0] / PUT request received
```

A's distribution to B no longer shows up as a local log line — it's broadcast over UDP instead. Run [Monitoring](./monitoring.md)'s listener on a PC on the same network and you should see it arrive there:

```
[192.168.x.a:9090] Message { idempotency: 0197..., payload: Log("PUT <guid> to 192.168.x.y:8080") }
```

where `192.168.x.y` is machine B's address. On B's side, the incoming share arrives as an ordinary PUT — in B's local log:

```
[INFO  - derusting:0] on_accept
[INFO  - derusting:0] / PUT request received
```

and, over UDP, B also broadcasts `Log("Receiving file from machine")` the moment it recognises the `guid:` header on the incoming request.

Power down B, pull its USB stick, and plug it into your PC — you should find a `<guid>.gcode` file matching the one you uploaded to A, even though you never uploaded anything to B directly.

> [!NOTE]
> If you have a third machine, C, also on the network, watch the Monitoring listener rather than A's own serial log: you should see two separate `PUT <guid> to ...` broadcasts from A, one for B and one for C, and neither of them should re-forward the file again themselves (their `PutInfo::guid` was set from A's `guid:` header, so `info.guid.is_none()` is `false` on their end — each just logs its own `Receiving file from machine` broadcast instead).
