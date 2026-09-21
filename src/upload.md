# File Upload

This is where everything meets: [HTTP Service](./http.md)'s parsing, [Submission Portal](./portal.md)'s `fetch()` PUT, and [FS](./fs.md)'s file API all come together in `handle_conn`'s `PUT` branch to turn an uploaded file into bytes safely written to the USB stick. Stage gate: **upload a real `.gcode` file through the portal and watch it land on disk, chunk by chunk, in the log.**

## Validating the request first

Before writing a single byte, `check_put_header` picks the fields we care about out of the request headers:

```rust
pub struct PutInfo {
    pub size: Option<usize>,
    pub guid: Option<Uuid>,
    pub is_gcode: bool,
}

fn check_put_header(headers: &str) -> PutInfo {
    let mut info = PutInfo { size: None, guid: None, is_gcode: false };
    for line in headers.lines() {
        let Some((key, val)) = line.split_once(':') else { continue; };
        let key = key.trim();
        let val = val.trim();
        if key.eq_ignore_ascii_case("content-length") {
            info.size = val.parse::<usize>().ok();
        } else if key.eq_ignore_ascii_case("content-type") && val == "text/x.gcode" {
            info.is_gcode = true;
        } else if key.eq_ignore_ascii_case("guid") {
            info.guid = val.parse::<Uuid>().ok();
        }
    }
    info
}
```

`guid` is only ever present when the request came from another machine forwarding a job (see [File Share](./file_share.md)'s `guid:` header) — a browser upload from the portal never sends one, so `info.guid` is `None` for a genuinely new job.

> [!NOTE]
> `assets/index.html`'s JavaScript never sets a `Content-Type` header explicitly — but browsers do send `Content-Type: text/x.gcode` for a `.gcode` file anyway, inferred from the file's extension via the OS's own MIME type registry. `is_gcode` relies on that inference holding, which is a little fragile (a `.gcode` file with no MIME association on the uploader's OS would fail this check) but has held up in practice.

Back in `handle_conn`, the `Put` branch rejects anything that doesn't look like a real, reasonably-sized gcode upload before opening a file at all:

```rust
Method::Put => {
    log_info!("/ PUT request received");
    let info = check_put_header(headers);
    if !info.is_gcode
        || info.size.is_none()
        || info.size.is_some_and(|s| s == 0 || s > 1_000_000)
    {
        let _ = conn.response(BAD_REQUEST.as_bytes()).await;
        return;
    }
    // ...
}
```

A 1MB cap keeps a single job's footprint bounded — reasonable for the small gcode files this demonstrator is designed around.

## Streaming the body to disk

Rather than buffer the whole upload in memory (which the STM32F407's RAM couldn't hold for anything but the smallest files), the handler writes each TCP chunk to the file as it arrives, using the `guid` (either the one just extracted, or a fresh one for a genuinely new upload) to build the path:

```rust
let guid = match info.guid {
    Some(guid) => guid,
    None => generate_uuid_v7(),
};
let path = fs::make_path(&guid, true); // .partial while incomplete

let Ok(mut f) = fs::open(path.as_c_str(), WriteBytes) else {
    let _ = conn.response(INTERNAL_SERVER_ERROR.as_bytes()).await;
    return;
};

let mut content_length = info.size.unwrap();

// Write whatever body bytes arrived in this first packet.
let to_write = core::cmp::min(content_length, body.len());
let _ = f.write(&body[..to_write]);
content_length = content_length.saturating_sub(to_write);

// The rest of the first pbuf chain, if the request spanned more than one.
let mut more_packets_needed = true;
for chunk in iter {
    log_info!("Chunk Length: {}", chunk.len());
    let to_write = core::cmp::min(content_length, chunk.len());
    let _ = f.write(&chunk[..to_write]);
    content_length = content_length.saturating_sub(to_write);
    if content_length == 0 {
        more_packets_needed = false;
        break;
    }
}

// Still more to come? Keep awaiting further packets from the connection.
if more_packets_needed {
    while content_length > 0 {
        let Some(pbuf) = conn.as_ref().receive().await else {
            log_error!("Handle Reset");
            f.close();
            fs::delete(&path);
            return;
        };
        for chunk in pbuf.into_iter() {
            log_info!("Chunk Length: {}", chunk.len());
            let to_write = core::cmp::min(content_length, chunk.len());
            let _ = f.write(&chunk[..to_write]);
            content_length = content_length.saturating_sub(to_write);
            if content_length == 0 { break; }
        }
    }
}
```

Three things worth noticing:

- **`.partial` first, `.gcode` after**: the file is written under `fs::make_path(&guid, true)` (the `.partial` suffix — see [FS](./fs.md)) and only renamed to its final `.gcode` name once every byte has arrived, so a half-received file never looks complete to anything else scanning the USB stick (like `manage_ledger`'s printable-job search — see [Job Ledger](./ledger.md)).
- **`content_length` is trusted, not re-derived**: it comes straight from the `Content-Length` header, tracked down to zero as bytes are written. There's no independent check that the connection didn't send more or fewer bytes than it claimed.
- **A dropped connection cleans up after itself**: if `receive()` returns `None` (the peer closed or reset the connection) before `content_length` reaches zero, the partial file is deleted rather than left behind as a half-written, forever-incomplete `.partial` file.

Once the whole body has landed:

```rust
f.close();
let new_path = fs::make_path(&guid, false);
fs::rname(&path, &new_path);
let _ = conn.response(OK.as_bytes()).await;

if info.guid.is_none() {
    // A genuinely new upload — never a file another machine already
    // shared with us (see the `guid` check above) — needs to be
    // announced to the rest of the network. Covered next, in Gossiping.
}
```

That last `if` is where a genuinely new upload hands off into the network — telling the job ledger about it and pushing a copy out to every other machine. [Gossiping](./gossip.md) fills in what actually goes there.

## Did it work?

Upload a small `.gcode` file through the portal (from [Submission Portal](./portal.md)). You should see the browser's alert say `Sent`, and the log show something like:

```
[INFO  - derusting:0] / PUT request received
[INFO  - derusting:0] Chunk Length: 1024
[INFO  - derusting:0] Chunk Length: 1024
...
```

(exact chunk sizes and count depend on the file size and how lwIP happened to segment it). Power the machine down, pull the USB stick, and check on your PC — you should find a `<guid>.gcode` file (not `.partial`) containing exactly what you uploaded.

> [!TIP]
> If an upload seems to hang or the browser reports an error partway through, it's a known rough edge: some browsers intermittently report `NS_ERROR_NET_RESET` mid-upload against this handler. It's been narrowed down enough to rule out `is_gcode`/`Content-Type` detection as the cause (see the note above), but the underlying trigger — something in how a `TcpConnection` gets torn down mid-transfer — is still being investigated as of this writing. If you hit it, the safest recovery is simply to retry the upload.

With a file safely on disk, [File Share](./file_share.md) covers pushing a copy of it out to every other machine on the network, and [Job Ledger](./ledger.md) covers who actually gets to print it.
