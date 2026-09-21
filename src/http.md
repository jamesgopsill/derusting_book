# HTTP Service

[LWIP TCP](./lwip_tcp.md) gave us a listener and a way to read/write bytes over a connection — raw TCP, nothing HTTP-shaped about it yet. This chapter is the thin layer on top that turns those bytes into requests and responses: just enough HTTP to serve one page and accept one kind of upload, deliberately nothing more. Stage gate: **send the server a request it doesn't understand, and get back the right HTTP error rather than silence or a crash.**

## Only two methods, one path

The entire service understands exactly two things: `GET /` and `PUT /`. `http.rs` defines the `Method` enum and the canned responses for everything else:

```rust
pub enum Method {
    Get,
    Put,
}

pub const OK: &str = "HTTP/1.1 200 OK\r\nContent-length:4\r\nConnection: close\r\n\r\npong";
pub const BAD_REQUEST: &str =
    "HTTP/1.1 400 Bad Request\r\nContent-length:0\r\nConnection: close\r\n\r\n";
pub const METHOD_NOT_ALLOWED: &str =
    "HTTP/1.1 405 Method Not Allowed\r\nContent-length:0\r\nConnection: close\r\n\r\n";
pub const INTERNAL_SERVER_ERROR: &str =
    "HTTP/1.1 500 Internal Server Error\r\nContent-length:0\r\nConnection: close\r\n\r\n";
```

> [!NOTE]
> `OK`'s body is literally the text `pong` — a leftover from testing the connection plumbing before the real PUT handler existed. It's what a successful upload's response body still contains today; nothing in the portal's JavaScript reads it (see [Submission Portal](./portal.md)), so it's harmless, if a little cryptic if you go looking at it with `curl -v`.

## Parsing just enough of a request

We don't pull in a proper HTTP parsing crate — `tasks/tcp.rs` does the minimum by hand. `split_request` carves the first chunk of a connection's bytes into the start line, headers, and whatever body bytes arrived in the same packet:

```rust
fn split_request(buf: &[u8]) -> Option<(&str, &str, &[u8])> {
    let delim = b"\r\n";
    let idx = buf.windows(delim.len()).position(|win| win == delim)?;
    let (start_line, rest) = buf.split_at(idx);
    let rest = &rest[2..];
    let delim = b"\r\n\r\n";
    let idx = rest.windows(delim.len()).position(|win| win == delim)?;
    let (headers, rest) = rest.split_at(idx);
    let body = &rest[4..];
    let start_line = str::from_utf8(start_line).ok()?;
    let headers = str::from_utf8(headers).ok()?;
    Some((start_line, headers, body))
}
```

`check_start_line` then decides whether we support what's being asked, rejecting anything else immediately:

```rust
fn check_start_line(start_line: &str) -> Result<Method, &'static str> {
    let mut tokens = start_line.split(" ");
    let Some(method) = tokens.next() else {
        return Err(BAD_REQUEST);
    };
    let method = match method {
        "GET" => Method::Get,
        "PUT" => Method::Put,
        _ => return Err(METHOD_NOT_ALLOWED),
    };

    let Some(url) = tokens.next() else {
        return Err(BAD_REQUEST);
    };
    if url != "/" {
        return Err(BAD_REQUEST);
    }

    Ok(method)
}
```

Any HTTP version, any other path, any other verb — all rejected before we do anything more expensive. `handle_conn` (see [LWIP TCP](./lwip_tcp.md)) wires these together: parse the start line, bail out with the matching error response on failure, otherwise dispatch on `Method`.

> [!NOTE]
> `split_request` only looks at whatever arrived in the *first* TCP chunk. If a client somehow split the request line or headers themselves across multiple packets (vanishingly unlikely for the short GET/PUT requests this server expects, but worth knowing), parsing would fail and the connection would just be dropped rather than the server waiting for more data. This is a deliberate simplification, not a bug — a real HTTP server would need to buffer across packets to handle that case properly.

## Did it work?

With the firmware flashed and the portal reachable (from [LWIP TCP](./lwip_tcp.md)), try a request the server explicitly doesn't support:

```bash
curl -i -X DELETE http://[IP_ADDRESS_OF_MACHINE]:8080/
```

You should get back:

```
HTTP/1.1 405 Method Not Allowed
Content-length:0
Connection: close
```

and in the device log:

```
[INFO  - derusting:0] New Connection Received
[INFO  - derusting:0] Handling TCP Connection
```

(no further log lines — `check_start_line` rejects it before either the `GET` or `PUT` branch logs anything). Try an unsupported path too:

```bash
curl -i http://[IP_ADDRESS_OF_MACHINE]:8080/anything
```

which should come back `400 Bad Request` for the same reason — `url != "/"`. Getting the *correct* error back for a request we never anticipated is exactly the point: the server never crashes or hangs on something it doesn't understand, it just says no.
