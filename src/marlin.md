# Marlin

We can boot a task ([Embassy](./embassy.md)), talk to the network ([LWIP UDP](./lwip_udp.md), [LWIP TCP](./lwip_tcp.md)), and read and write files ([FS](./fs.md)). The last primitive is the one that actually makes this a manufacturing system: getting a line of gcode from Rust into Marlin, the firmware's print engine, and getting a "ready to accept work" signal back out onto the physical screen. Stage gate for this chapter: **flash the firmware, toggle the machine ONLINE/OFFLINE on its own screen, and watch it physically home its axes when told to print.**

## Talking to Marlin from Rust

Marlin isn't something we're binding to directly — the Buddy firmware already wraps it behind `marlin_server`, which owns a command queue and knows whether the printer is currently busy. `libderusting.cpp` adds two small `extern "C"` shims over that:

```cpp
// Submits a line of gcode to Marlin's command queue from Rust.
extern "C" bool derusting_gcode_cmd(const char* cmd) {
  log_info(derusting, "Gcode from Rust: %s", cmd);
  return marlin_server::enqueue_gcode_try(cmd);
}

// Reports whether Marlin's print engine is currently idle.
extern "C" bool derusting_is_idle() {
  return marlin_server::printer_idle();
}
```

`marlin.rs` wraps these in safe Rust:

```rust
unsafe extern "C" {
    fn derusting_gcode_cmd(cmd: *const c_char) -> bool;
    fn derusting_is_idle() -> bool;
}

pub fn is_idle() -> bool {
    unsafe { derusting_is_idle() }
}

#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("Marlin returned false.")]
    MarlinReturnedFalse,
    #[error("Error build gcode: {0}.")]
    ExtendError(#[from] heapless::c_string::ExtendError),
    #[error("the printer is is busy.")]
    Busy,
}

/// All the prints sent through `derusting` are all dry print (no extrusion)
/// for demonstration purposes.
pub fn print(path: &CStr, dry: bool) -> Result<(), Error> {
    if is_idle() {
        let mut cmd = CString::<64>::new();
        cmd.extend_from_bytes(b"M32 ")?;
        cmd.extend_from_bytes(path.to_bytes())?;
        if dry {
            // M111 S8 puts Marlin into "dry run" mode: it executes gcode
            // without actually driving the extruder, so testing this is safe.
            let res = unsafe { derusting_gcode_cmd(c"M111 S8".as_ptr()) };
            if !res {
                return Err(Error::MarlinReturnedFalse);
            }
        }
        let res = unsafe { derusting_gcode_cmd(cmd.as_ptr()) };
        if res { Ok(()) } else { Err(Error::MarlinReturnedFalse) }
    } else {
        Err(Error::Busy)
    }
}

pub fn home() -> Result<(), Error> {
    if is_idle() {
        let res = unsafe { derusting_gcode_cmd(c"G28".as_ptr()) };
        if res { Ok(()) } else { Err(Error::MarlinReturnedFalse) }
    } else {
        Err(Error::Busy)
    }
}
```

`print()` submits `M32 <path>` (Marlin's "print SD file" command, repurposed here to print straight off the USB stick), and always issues `M111 S8` first when `dry` is `true` — this puts the print engine into a debug mode that runs the gcode without moving the extruder motor, which is exactly why derusting can be tested and demoed safely on real hardware. `home()` is a much smaller, self-contained command (`G28`) — useful for proving the whole FFI path works without needing a `.gcode` file at all.

## A "ready" flag the technician controls

Everything above assumes a technician has actually authorised this machine to accept jobs — derusting shouldn't start printing autonomously the moment it powers on. That authorisation is a single `AtomicBool`, `derusting_ready_flag`, defined on the C++ side and shared across the FFI boundary:

```rust
unsafe extern "C" {
    static derusting_ready_flag: core::sync::atomic::AtomicBool;
    fn derusting_update_ui();
}

pub fn is_ready() -> bool {
    unsafe { derusting_ready_flag.load(core::sync::atomic::Ordering::SeqCst) }
}

/// We need to set the ready_flag to false when a job has been selected by
/// the machine for manufacture.
pub fn set_offline() {
    unsafe { derusting_ready_flag.store(false, core::sync::atomic::Ordering::SeqCst) };
    unsafe { derusting_update_ui() };
}
```

`is_ready()` is what [Job Ledger](./ledger.md)'s logic checks before this machine will pick a job off the ledger and print it; `set_offline()` flips it back off once a job has been claimed, so the machine doesn't grab a second job mid-print.

## Repurposing the home screen's Print button

The flag needs a physical control, so `assets/screen_home.cpp`/`.hpp` patch the Buddy GUI's home screen: the Print button (position `0`) is rewired to toggle `derusting_ready_flag` instead of opening the file browser, and its label swaps between `OFFLINE`/`ONLINE`:

```cpp
// JG: Providing the bridge between Buddy and Derusting
extern "C" std::atomic<bool> derusting_ready_flag(false);
static screen_home_data_t *home_screen_instance = nullptr;

void screen_home_data_t::refresh_ready_btn() {
    if (derusting_ready_flag.load()) {
        w_labels[0].SetText(_("ONLINE"));
    } else {
        w_labels[0].SetText(_("OFFLINE"));
    }
    w_labels[0].Invalidate();
}

extern "C" void derusting_update_ui() {
    if (home_screen_instance != nullptr) {
        home_screen_instance->refresh_ready_btn();
    }
}
```

and in the button table itself:

```cpp
// JG: Commandering the Print Button
{ this, Rect16(), nullptr, [this](window_t&) {
    bool is_ready = !derusting_ready_flag.load();
    derusting_ready_flag.store(is_ready);
    this->refresh_ready_btn();
} },
```

The header text is also relabelled from the Prusa version string to `DERUSTING`, so you can tell at a glance you're running our firmware rather than stock Buddy.

These two files aren't part of the Rust crate — they're firmware assets, patched in at build time. `scripts/build_firmware.sh` copies them over the stock ones just before the Buddy build runs:

```bash
# UI elements
rsync -c assets/screen_home.hpp buddy/src/gui/screen_home.hpp
rsync -c assets/screen_home.cpp buddy/src/gui/screen_home.cpp
```

## Wiring it into `lib.rs`

`derusting_main` reports the ready flag's state on boot, and always starts a session in the "offline" state (a technician has to opt back in after every flash/reboot):

```rust
log_info!("Is Ready: {}", is_ready());
set_offline();
```

The actual print trigger lives in `tasks/udp.rs`'s `manage_ledger` task, which we'll cover properly in [Job Ledger](./ledger.md) — it only calls `marlin::print(..)` once this machine owns the job ledger, is `is_ready()`, and `is_idle()`:

```rust
if ledger_state.is_owner() && marlin::is_ready() && marlin::is_idle() {
    // ...find a printable job on the USB stick...
    match marlin::print(&path, true) {
        Ok(_) => {
            marlin::set_offline();
            ledger_state.ledger.jobs.remove(&guid);
        }
        Err(e) => log_error!("Print Error: {e}"),
    }
}
```

For this chapter's stage gate, you don't need the ledger at all — a single manual call to `marlin::home()` proves the whole FFI chain (Rust → `libderusting.cpp` → `marlin_server`) without anything else in place.

## Did it work?

Rebuild and flash:

```bash
bash scripts/build_and_flash.sh
```

**1. The ONLINE/OFFLINE toggle.** On the home screen you should see `DERUSTING` in the header, and the button that's normally labelled `Print` now reads `OFFLINE`. Tap it — it should flip to `ONLINE` and back each time you press it, with no code changes needed beyond what's above. This alone confirms `derusting_ready_flag` and the UI patch are wired correctly.

**2. A real gcode command.** Temporarily call `marlin::home()` from somewhere that runs once at boot (e.g. `embassy_main`, guarded so it only fires the first time):

```rust
if let Err(e) = marlin::home() {
    log_error!("Home error: {e}");
}
```

Flash again and watch the printer itself: the head and bed should physically move to home position, and the log should show:

```
[INFO  - derusting:0] Gcode from Rust: G28
```

That's the full loop confirmed — Rust submitted real gcode, Marlin executed it, and the printer moved. Remove the manual `home()` call once you've seen it work; [Job Ledger](./ledger.md) wires `print()` up properly, gated behind the ready flag and the job ledger, so the machine only ever prints (in dry-run mode) a job it was actually given.

With all five primitives now working independently — task initialisation, UDP, TCP, FS, and Marlin — the remaining chapters are about composition: [Submitting Jobs](./job_submission.md) turns an HTTP upload into a job on disk, and [Gossiping](./gossip.md) shares that job across the network and gets it printed, dry-run, by whichever machine's turn it is on the job ledger.
