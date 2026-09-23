# Binding LWIP TCP

With UDP broadcasting our heartbeat (see [LWIP UDP](./lwip_udp.md)), the next primitive is TCP — this is how a user actually talks to a machine: pointing a browser at it and submitting a job. This chapter binds a TCP listener on port `8080` and serves a minimal HTTP response. Our stage gate: **open a browser, hit the machine's IP, and see a real HTML page served back from our own Rust code.**

## Raw TCP is a bit more involved than UDP

UDP only needed one long-lived object (the bound socket). TCP needs two concepts:

- A **listener** — bound and listening on a port, whose only job is to accept new incoming connections.
- A **connection** — one accepted client, with its own send/receive state, that outlives the individual `accept` callback that created it.

lwIP's raw TCP API mirrors that split (`tcp_listen_with_backlog` + a `tcp_accept` callback handing you a fresh `pcb` per connection), so `lwip/tcp.rs` gives us two matching Rust types: `TcpListener<N, M>` and `TcpConnection<M>`.

### `TcpListener`

`listen()` creates a pcb, binds it, and puts it into the `LISTEN` state, registering `_accept` as the callback for new connections:

```rust
pub async fn listen(&'static self, port: u16) -> Result<(), err_t> {
    super::async_lwip(|| unsafe {
        let pcb = tcp_new();
        if pcb.is_null() { return Err(err_t::Mem); }
        let err = tcp_bind(pcb, &ip_addr_any, port);
        if err != err_t::Ok {
            tcp_close(pcb);
            return Err(err);
        }
        let listen_pcb = tcp_listen_with_backlog(pcb, 2);
        if listen_pcb.is_null() {
            return Err(err_t::Mem);
        }
        tcp_arg(listen_pcb, self.as_mut_ptr());
        tcp_accept(listen_pcb, Some(Self::_accept));
        self.pcb.store(listen_pcb, Ordering::Release);
        Ok(())
    })
    .await
}
```

When a client connects, `_accept` fires on lwIP's thread. It doesn't do any application logic itself — it just parks the new pcb behind a temporary "hold" receive callback (so lwIP buffers any data the client sends before we're ready) and hands the raw pcb over to the async side via a channel:

```rust
unsafe extern "C" fn _accept(arg: *mut c_void, pcb: *mut pcb, err: err_t) -> err_t {
    // ...
    unsafe {
        tcp_arg(pcb, core::ptr::null_mut());
        tcp_recv(pcb, Some(Self::_hold_recv));
    }
    if listener.channel.try_send(pcb).is_err() {
        unsafe { tcp_abort(pcb) };
    }
    err_t::Ok
}
```

`with_connection` is the async entry point the rest of the app uses — it waits for the next accepted pcb, wraps it in a pinned `TcpConnection`, attaches its real callbacks, and runs your handler against it:

```rust
pub async fn with_connection<F>(&'static self, fcn: F)
where
    F: AsyncFnOnce(Pin<&mut TcpConnection<M>>),
{
    let pcb = self.channel.receive().await;
    let conn = TcpConnection::<M>::new(pcb);
    let mut conn = core::pin::pin!(conn);
    let _ = conn.as_mut().attach_callbacks().await;
    fcn(conn).await;
}
```

### `TcpConnection`

Once attached, a connection gives us `receive()` (await the next chunk of the request, or `None` on EOF/error) and `response()` (write bytes back and flush):

```rust
pub async fn response(self: Pin<&mut Self>, bytes: &[u8]) -> Result<(), err_t> {
    super::async_lwip(|| unsafe {
        let pcb = self.pcb.load(Ordering::Acquire);
        if pcb.is_null() { return Err(err_t::Mem); }
        let err = tcp_write(pcb, bytes.as_ptr(), bytes.len() as u16, TCP_WRITE_FLAG_COPY);
        if err != err_t::Ok { return Err(err); }
        let err = tcp_output(pcb);
        if err != err_t::Ok { return Err(err); }
        Ok(())
    })
    .await
}
```

## A minimal HTTP layer

We don't need (or want) a full HTTP server for this — just enough to tell `GET /` from `PUT /` and respond appropriately. `http.rs` holds the canned response strings, plus the HTML page itself, embedded at compile time from `assets/index.html`:

```rust
pub const OK: &str = "HTTP/1.1 200 OK\r\nContent-length:4\r\nConnection: close\r\n\r\npong";
pub const BAD_REQUEST: &str =
    "HTTP/1.1 400 Bad Request\r\nContent-length:0\r\nConnection: close\r\n\r\n";

const HTML_STR: &str = include_str!("../assets/index.html");
pub const INDEX_HTML: &str = formatcp!(
    "HTTP/1.1 200 OK\r\n\
    Content-Type: text/html\r\n\
    Content-Length: {}\r\n\
    Connection: close\r\n\r\n\
    {}",
    HTML_STR.len(),
    HTML_STR
);
```

`assets/index.html` is the job-submission upload form users will eventually see — we're just serving it as a static page for now.

`tasks/tcp.rs` ties it together. `tcp_worker` loops forever, handing each accepted connection to `handle_conn`, which does a minimal parse of the request line and, for a `GET`, responds with `INDEX_HTML`:

```rust
#[embassy_executor::task(pool_size = 2)]
pub async fn tcp_worker(tcp: &'static TcpListener<..>, /* ... */) {
    loop {
        tcp.with_connection(async |conn| {
            log_info!("New Connection Received");
            handle_conn(conn, /* ... */).await
        })
        .await
    }
}

pub async fn handle_conn<const N1: usize, const N2: usize, const N3: usize>(
    conn: Pin<&mut TcpConnection<N1>>,
    // ...
) {
    let Some(pbuf) = conn.as_ref().receive().await else { return; };
    let Some((start_line, headers, body)) = split_request(/* first chunk */) else {
        let _ = conn.response(BAD_REQUEST.as_bytes()).await;
        return;
    };
    let method = check_start_line(start_line)?;

    match method {
        Method::Get => {
            log_info!("/ GET request received");
            let _ = conn.response(INDEX_HTML.as_bytes()).await;
        }
        Method::Put => {
            // Accepting an uploaded .gcode file and writing it to the USB
            // stick — this needs the filesystem primitives from the next
            // chapter, so we'll come back to it once FS is in place.
        }
    }
}
```

For this stage gate, focus on the `Method::Get` branch — that's the one that needs nothing beyond what we've built so far. The `PUT` branch streams the uploaded file to disk via `fs::open(..)`/`fs::make_path(..)`, which we haven't introduced yet; it's covered properly once we get to [FS](./fs.md) and, later, [File Upload](./upload.md).

## Wiring it into `embassy_main`

```rust
pub const TCP_PORT: u16 = 8080;

static TCP: StaticCell<TcpListener<MAX_TCP_CONNECTIONS, MAX_TCP_CONNECTION_CHANNEL_SIZE>> =
    StaticCell::new();

async fn embassy_main(spawner: Spawner) {
    // ...udp bind from the previous chapter...

    let tcp = TCP.init_with(TcpListener::<MAX_TCP_CONNECTIONS, MAX_TCP_CONNECTION_CHANNEL_SIZE>::new);
    if let Err(err) = tcp.listen(TCP_PORT).await {
        log_critical!("TCP Failed: {err:?}");
        return;
    };
    log_info!("TCP up on {TCP_PORT}");

    // ...wait for IP...

    match tcp_worker(tcp, udp, address_book, ledger) {
        Ok(t) => spawner.spawn(t),
        Err(e) => log_error!("Spawn Error: {e}"),
    }
}
```

## Did it work?

Rebuild and flash:

```bash
bash scripts/build_and_flash.sh
```

Find the machine's IP address (the log lines from the [LWIP UDP](./lwip_udp.md) chapter will show it), then browse to `http://[IP_ADDRESS_OF_MACHINE]:8080`. You should be served the derusting upload page:

<p align="center">
  <img src="https://github.com/jamesgopsill/derusting_book/blob/main/src/assets/website.png?raw=true" height="400" alt="Website">
</p>

In the log you'll see:

```
[INFO  - derusting:0] TCP up on 8080
[INFO  - derusting:0] on_accept
[INFO  - derusting:0] New Connection Received
[INFO  - derusting:0] Handling TCP Connection
[INFO  - derusting:0] / GET request received
```

> [!TIP]
> `Handling TCP Connection` (and, once you get to the `PUT` branch, `/ PUT request received`) is now also broadcast over UDP as a `Log` message, not just printed to this local console — see [Address Book](./address_book.md)'s `Message::send_log` and [Monitoring](./monitoring.md) if you want to watch it from another machine on the network.

> [!NOTE]
> Don't try submitting a job through the form just yet — the `PUT` handler needs the filesystem work from the next chapter before it can save anything, and there's a known intermittent `NS_ERROR_NET_RESET` some browsers report on upload that's still being tracked down. `GET /` returning the page is the whole stage gate here.
