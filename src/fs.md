# FS

The last primitive we need before we can wire everything together is a filesystem: machines need to save uploaded `.gcode` files, read them back to check whether a job is printable, and hand them off to Marlin to actually print. This chapter gets us reading and writing a file on the USB stick from Rust. Stage gate: **write a file from Rust, flash, and see it appear on the USB stick.**

> This module started out as a direct binding to ChanFS before settling on the friendlier `stdio`-style shim below — the Buddy firmware happens to expose both.

## The firmware already gives us a filesystem

The Buddy firmware mounts a FAT filesystem on the USB stick (ChanFS/FatFs under the hood) and exposes it through the same `stdio.h`-style calls you'd use on desktop Rust or C: `fopen`, `fread`, `fwrite`, `fclose`, `fseek`, and FatFs's own `f_stat` for metadata. There's no need for a separate embedded filesystem crate — `fs.rs` is just an `extern "C"` binding over these, wrapped in a safe Rust API.

```rust
#[repr(C)]
struct Fil {
    _opaque: [u8; 0],
}

unsafe extern "C" {
    fn fopen(path: *const c_char, flags: *const c_char) -> *mut Fil;
    fn fwrite(buf_ptr: *const u8, buf_item_byte_size: usize, buf_len: usize, fp: *mut Fil) -> usize;
    fn fread(buf_ptr: *mut u8, buf_item_byte_size: usize, buf_len: usize, fp: *mut Fil) -> usize;
    fn fclose(fp: *mut Fil);
    fn fseek(fp: *mut Fil, offset: c_long, whence: c_int) -> c_int;
    fn ftell(fp: *mut Fil) -> c_long;
    fn fflush(fp: *mut Fil) -> c_int;
    fn unlink(path: *const c_char) -> c_int;
    fn rename(oldpath: *const c_char, newpath: *const c_char) -> c_int;
    fn f_stat(path: *const c_char, fno: *mut FilInfo) -> FResult;
}
```

## Typed, safe file handles

Rather than expose raw `fopen` mode strings, we use a small `Mode` trait with `ReadBytes`/`WriteBytes` marker types, and only implement `embedded_io::Read`/`Write` for the mode that supports them:

```rust
pub trait Mode {
    fn as_cstr(&self) -> &'static CStr;
}

pub struct ReadBytes;
impl Mode for ReadBytes {
    fn as_cstr(&self) -> &'static CStr { c"rb" }
}
impl ImplementsEmbeddedIoRead for ReadBytes {}

pub struct WriteBytes;
impl Mode for WriteBytes {
    fn as_cstr(&self) -> &'static CStr { c"wb" }
}
impl ImplementsEmbeddedIoWrite for WriteBytes {}

pub fn open<T: Mode>(path: &CStr, mode: T) -> Result<File<T>, ()> {
    if !path.to_bytes().starts_with(b"/usb/") {
        log_error!("Path must start with /usb/");
        return Err(());
    }
    let res = unsafe { fopen(path.as_ptr(), mode.as_cstr().as_ptr()) };
    if res.is_null() {
        Err(())
    } else {
        Ok(File { fp: res, _mode: mode })
    }
}
```

Every path is required to start with `/usb/` — that's our one safety rail, since the underlying FatFs volume is the USB stick and nothing else. `File<T>` implements `embedded_io::Read`, `Write` and `Seek` depending on its mode, and closes itself automatically on `Drop`:

```rust
impl<T> Drop for File<T> where T: Mode {
    fn drop(&mut self) {
        unsafe { fclose(self.fp) };
    }
}
```

## The stage-gate test

`fs.rs` already ships a small, deliberately manual smoke test — `test_file()` — that opens a file, writes to it, and closes it again:

```rust
/// Manual smoke test that opens, writes to, and closes a file on the USB
/// stick.
#[allow(unused)]
pub fn test_file() {
    if let Ok(mut f) = open(c"/usb/test.txt", WriteBytes) {
        log_info!("Test File Opened");
        match f.write(b"Hello World\n") {
            Ok(written) => {
                log_info!("Bytes written: {written}");
            }
            Err(_) => {
                log_error!("Failed to write bytes");
            }
        };
        f.close();
        log_info!("Test File Closed");
    }
}
```

It's marked `#[allow(unused)]` because nothing calls it by default — it's there for exactly this stage gate. Call it once from somewhere that runs early, for example at the top of `embassy_main` in `lib.rs`:

```rust
async fn embassy_main(spawner: Spawner) {
    fs::test_file();
    // ...udp/tcp bring-up from the previous two chapters...
}
```

If you also want to prove the *read* side works (not just write), extend the test to open the file back up in `ReadBytes` mode and log what comes back:

```rust
if let Ok(mut f) = open(c"/usb/test.txt", ReadBytes) {
    let mut buf = [0u8; 32];
    if let Ok(n) = f.read(&mut buf) {
        log_info!("Read back: {}", core::str::from_utf8(&buf[..n]).unwrap_or("<invalid utf8>"));
    }
}
```

## A lighter check that's already running

You've actually already seen this filesystem binding in action without realising it — `derusting_main`'s very first action after booting the Embassy task is a `stat()` call against the firmware image itself, logging its size:

```rust
match fs::stat(c"/usb/firmware.bbf") {
    Ok(info) => log_info!("bbf file_size: {}", info.fsize),
    Err(e) => log_error!("stat error: {e}"),
}
```

That's a nice minimal confirmation that the USB stick is mounted and readable at all, before we get to writing anything ourselves.

## Cleaning up on boot

There's a second, less obvious filesystem action that happens on every boot, before any task or UDP/TCP socket is up: `derusting_main()` calls `clean_usb()`, which walks the USB stick and deletes any leftover `*.gcode` and `*.partial` files. If the machine loses power or reboots mid-upload, a `.partial` file for a job that was never finished would otherwise sit on the stick indefinitely; a completed `.gcode` file, meanwhile, is expected to have already been claimed and printed or removed by the ledger before a reboot, so any that remain are stale too. `clean_usb()` just clears both categories out so a fresh boot always starts from a known-empty job set, rather than risk an old or half-written file being mistaken for a valid pending job.

> [!NOTE]
> The `fs::test_file()` call added above is a manual stage-gate check, not part of the ongoing source — nothing in the current `lib.rs` calls it. Once you've seen it work, remove (or comment out) that call before moving on to later chapters, the same way [Marlin](./marlin.md) has you remove its own manual `home()` test call once it's proven the gcode path works.

## Did it work?

Rebuild and flash:

```bash
bash scripts/build_and_flash.sh
```

In the log you should see:

```
[INFO  - derusting:0] bbf file_size: 1234567
[INFO  - derusting:0] Test File Opened
[INFO  - derusting:0] Bytes written: 12
[INFO  - derusting:0] Test File Closed
```

Then power down, pull the USB stick, and check it on your PC — you should find a new `test.txt` file in its root containing `Hello World`.

With task initialisation, UDP, TCP and now the filesystem all working independently, the next chapters ([Gossiping](./gossip.md), [Submitting Jobs](./job_submission.md)) are about wiring these four primitives together into the full decentralised job-sharing system.
