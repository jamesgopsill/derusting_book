# Scaffolding

This chapter takes you through the setup and structure of the derusting GitHub repository. It details how the different folders are set-up and how we build rust library, copy it over into the buddy firmware, build it into the firmware and then flash it onto a prusa machine. This is great info to know if you're looking at building your own integrating technology so lets begin.

## Creating the project

We start with:

```bash
cargo new --lib <project-name>
```
  
This creates a new rust library crate and git already initialised into it. We then `cd` into the directory and add:

```bash
git submodule add https://github.com/prusa3d/Prusa-Firmware-Buddy.git buddy
```

This links the Prusa firmware to the `buddy` folder within the repo and is where will be copying our built library into and then building together to form the firmware that will be flashed onto the device.

Once downloaded, open the `.gitmodules` and add the following.

```.gitmodules
[submodule "buddy"]
	path = buddy
	url = https://github.com/prusa3d/Prusa-Firmware-Buddy.git
++	ignore = all
```

This ignores any changes made within that directory. We will be patching and copying our library over to this folder. We will be overwriting this files as we develop so we do not need to keep a track of these changes as we can always we generate by triggering a build.

`cd` into this directory and create the buddy firmware's python build environment.

```bash
python -m venv .venv
```

And then activate it and install the necessary dependencies.

```bash
source .venv/bin/activate
pip install -r requirements.txt
```

You can even test building the firmware as-is by:

```bash
python utils/build.py --preset mini --build-type release --bootloader no
```

You may need to change some of these values for the machine you're working on and if you intend on loading the firmware using a USB stick or directly through a probe connected to the board - which is the way we will do it here.

> [!IMPORTANT]
> You will need to copy over the `firmware.bbf` to a USB stick and have it inserted into your device. This contains the assets for the GUI and it may need to pull them from time to time and load it back into the memory on the device. The firmware will panic otherwise.
>
> With that plugged in, why not check if you can load the buddy firmware by running:
> ```bash
> probe-rs run --chip STM32F407VG ./buddy/build/mini_release_noboot/firmware
> ```

### Targeting the buddy board platform

The buddy board features a `STM32F407vg` chip which requires the Rust library to built for a different target to your PC. Rust will target your PC by default so we need to add some configuration to tell it to compile the library for our target platform.

First, we create a `rust-toolchain.toml` file in the root directory which specifies the targets we're building for.

```toml
[toolchain]
targets = ["thumbv7em-none-eabihf"]
```

We need to the create a folder and file `.cargo/config.toml` and add:

```toml
[build]
target = "thumbv7em-none-eabihf"
```

which tells the compiler what target to build the library for.

We then want to edit `Cargo.toml` and add:

```toml
[package]
name = "derusting"
version = "0.1.0"
edition = "2024"

++ [lib]
++ crate-type = ["staticlib"]

++ [profile.release]
++ opt-level = "z"
++ lto = true
++ codegen-units = 1
++ panic = "abort"
++ strip = true


[dependencies]
  
```

Note., the `++` is just there to show you what has been added. This tells the compiler to build a static library which makes it portable and can be included into other codebases through the C-ABI that we will be using in a bit.

### Integrated Development Environment (IDE)

Now this will depend on your set-up as you will likely need to tell your IDE that you're targeting a different platform and therefore it needs to check the code against that target rather than the host target. I have been using the helix editor and to set this up you need to create a `.helix/languages.toml` folder and file and add:

```toml
[language-server.rust-analyzer.config]
check = { command = "clippy", allTargets = false }
cargo = { allTargets = false }
cfg = { setTest = false }
```

This gets rust-anlyzer to focus on the target of interest.

## Creating the firmware hook

Ok, it's meant to be a `Rust` project but we need to write a little but of `C` so we can provide the necessary glue to hook the library up to the buddy firmware and have it compiled into it.

Start by creating a folder - possibly `lib` + the name of your project `libderusting` - in the root directory. This will contain the glue and will be copied over into an appropriate place in the buddy firmware during the build.

In this folder, make the following files - `CMakeLists.txt`, `<project-name>.hpp` and `<project-name>.cpp`. In the `CMakeLists.txt` file add:

```cmake
target_include_directories(firmware PRIVATE .)

target_sources(
  firmware
  PRIVATE libderusting.cpp
)

file(GLOB RUST_LIBS "./*.a")
target_link_libraries(firmware PRIVATE ${RUST_LIBS})
target_link_options(firmware PRIVATE
  "-Wl,--start-group"
  ${RUST_LIBS}
  "-Wl,--end-group"
)
```

This provides information to CMake about our Rust library so it can be compiled and linked into the firmware. In the `lib<project-name>.cpp` (e.g., `libderusting.cpp`) add:


```c++
#include "marlin_client.hpp"
#include "logging/log.hpp"

LOG_COMPONENT_DEF(derusting, logging::Severity::info);

extern "C" void derusting_log_event(logging::Severity severity, const char* msg) {
  log_event(severity, derusting, "%s", msg);
}

extern "C" void derusting_gcode_cmd(const char* cmd) {
  marlin_client::gcode(cmd);
}
```

And in the `<project-name>.hpp` add:

```c++
#pragma once

/**
 * @file libderusting.hpp
 * @brief Calling Rust from C/C++.
 */
#ifdef __cplusplus
extern "C" {
#endif
  /**
   * @brief Our entrypoint to our Rust library.
   */
  void derusting_main(void);
#ifdef __cplusplus
}
#endif
```

This should be all the `c++` you'll need to do (Apart from defining the extern "C" types in Rust for whatever bindings you may need :D).

### The Rust Code

Ok, now we can start our Rust library. In our `src` folder, we will be creating three files:

- `panic.rs`
- `log.rs`
- `lib.rs`

> [!NOTE]
> An earlier version of this chapter also had you create a `free_rtos_alloc.rs` here, hooking FreeRTOS's `pvPortMalloc`/`vPortFree` up as a `#[global_allocator]` so `alloc` (`String`, `Vec`, `format!`, …) would work in this `#![no_std]` crate. It turned out derusting never actually needs heap allocation — everything fits in fixed-capacity `heapless` types instead — so the allocator was removed again later in the project. We're skipping it here to save you the round trip; `log.rs` below uses `heapless::format!` for exactly this reason.

The next module we will create is `panic.rs`. Our Rust code requires a panic handler if something goes wrong. The compiler will fail and tell us that we need to implement one if it isn't present.

```rust
use core::panic::PanicInfo;

unsafe extern "C" {
    fn abort() -> !;
}

/// We need a panic handler and here it is. It hooks into the
/// abort function present in the firmware.
#[panic_handler]
pub fn panic(_info: &PanicInfo) -> ! {
    unsafe { abort() };
}
```

The panic handler simply calls the `abort()` function present in the Buddy firmware.

Next is a vital piece of any code and that is the ability to log so we can see the progress of any code we write. Create a `log.rs` file and add the following:

```rust
use core::ffi::c_char;

use heapless::format;

/// The severity of the log event.
#[repr(i32)]
#[allow(unused)]
#[derive(Debug, Copy, Clone, PartialEq, Eq)]
pub enum Severity {
    Debug = 1,
    Info = 2,
    Warning = 3,
    Error = 4,
    Critical = 5,
}

// The `.cpp` hook exposes an extern "C" function to the
// firmware logger so we can hook into it.
unsafe extern "C" {
    pub fn derusting_log_event(severity: Severity, msg: *const c_char);
}

/// Our internal log function
pub fn log(severity: Severity, args: core::fmt::Arguments) {
    // `heapless::format!` is a fallible, fixed-capacity `format!` — no heap
    // allocator required. 64 bytes is enough for typical log lines; a
    // message that overflows it is replaced rather than truncated silently.
    if let Ok(msg) = format!(64; "{}\0", args) {
        unsafe { derusting_log_event(severity, msg.as_ptr() as *const _) };
    } else {
        unsafe { derusting_log_event(severity, c"Message too long...".as_ptr() as *const _) };
    }
}

#[macro_export]
macro_rules! log_info {
    ($($arg:tt)*) => {{
        $crate::log::log($crate::log::Severity::Info, format_args!($($arg)*));
    }};
}

#[macro_export]
macro_rules! log_debug {
    ($($arg:tt)*) => {{
        $crate::log::log($crate::log::Severity::Debug, format_args!($($arg)*));
    }};
}

#[macro_export]
macro_rules! log_error {
    ($($arg:tt)*) => {{
        $crate::log::log($crate::log::Severity::Error, format_args!($($arg)*));
    }};
}

#[macro_export]
macro_rules! log_warning {
    ($($arg:tt)*) => {{
        $crate::log::log($crate::log::Severity::Warning, format_args!($($arg)*));
    }};
}

#[macro_export]
macro_rules! log_critical {
    ($($arg:tt)*) => {{
        $crate::log::log($crate::log::Severity::Critical, format_args!($($arg)*));
    }};
}
```

This files hooks into the Buddy logging function through our `derusting_log_event` we exposed on the C side in `libderusting.cpp`. This makes our logs fit seamlessly with the other Buddy log events.

Now we have all we need to allocate memory on the heap, log our activity and panic if all goes wrong. We can now go to our `lib.rs` and write our entrypoint:

```rust
#![no_std]

mod log;
mod panic;

/// # Safety
/// We will ensure that we call this function in an
/// appropriate place in the Buddy firmware.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn derusting_main() {
    log_info!("Hello from Rust");
}
```

We need to declare our modules and then create our `derusting_main()` function which is exposed and not mangled so it can be picked up and linked in with the Buddy firmware.

We should now be good to go to build and flash the firmware onto a Prusa machine!


### The Build Script

Building the fimrware requires us to build our firmware, patch a couple of files in the buddy firmware so it knows about our library and to call it during boot, copy our lib folder and lib, and then build the buddy firmware. The following bash scripts handles all of this for us. Keep them together in a `scripts` directory at the repo root:

```bash
mkdir -p scripts
```

Create `scripts/build_firmware.sh` as:

```bash
cargo build --release || {
  echo "Cargo Build Failed"
  exit 1
}

RLIB=libderusting
TARGET_DIR=$(cargo metadata --format-version 1 --no-deps | jq -r '.target_directory')

echo "Copying Rust lib folder..."
cp -r "./${RLIB}" ./buddy/src

cp "${TARGET_DIR}/thumbv7em-none-eabihf/release/${RLIB}.a" "./buddy/src/${RLIB}"

echo "Patching files..."

CMAKE_LIST=buddy/src/CMakeLists.txt

if ! grep -q "add_subdirectory(${RLIB})" ${CMAKE_LIST}; then
  echo "Patching ${CMAKE_LIST}"
  echo "add_subdirectory(${RLIB})" | cat - ${CMAKE_LIST} >tmp.cpp && mv tmp.cpp ${CMAKE_LIST}
fi

BUDDY_MAIN=buddy/src/buddy/main.cpp

if ! grep -q "#include <${RLIB}/${RLIB}.hpp>" ${BUDDY_MAIN}; then
  echo "Patching ${BUDDY_MAIN} (Include)"
  echo "#include <${RLIB}/${RLIB}.hpp>" | cat - ${BUDDY_MAIN} >tmp.cpp && mv tmp.cpp ${BUDDY_MAIN}
fi

if ! grep -q "derusting_main();" ${BUDDY_MAIN}; then
  echo "Patching ${BUDDY_MAIN} (Function Call)"
  sed -i '/metrics_reconfigure();/a \      derusting_main();' ${BUDDY_MAIN}
fi

cd ./buddy || {
  echo "Failed to find buddy dir"
}

python utils/build.py --preset mini --build-type release --bootloader no

cd ..

echo "BUILD FINISHED"
```

> [!NOTE]
> The script above is the bare minimum needed to get `Hello from Rust` flashed. The one checked into the repo (`scripts/build_firmware.sh`) has grown a few more steps since — it takes a required `--bootloader yes|no` argument, bumps FreeRTOS's `configTOTAL_HEAP_SIZE`, passes a couple more CMake flags (see [Buddy Firmware](./buddy.md)), and rsyncs a GUI asset over (added in [Updating the UI](./ui.md)). Compare against the real script if you want the fully up-to-date version rather than reproducing it step by step here.

## Flashing onto the device

We can now create a script to flash the device on a successful build. Create `scripts/flash.sh` that as:

```bash
probe-rs run --chip STM32F407VG ./buddy/build/mini_release_noboot/firmware
```

which will do the job of erasing and programming our buddy board with the new firmware.

We can also create a `scripts/build_and_flash.sh` script that combines the two and only flashes the device on a successful build. Since both scripts always run from the repo root (not from inside `scripts/` itself), it needs to reference its siblings with the `scripts/` prefix too. Note the `--bootloader no` argument — the real `build_firmware.sh` requires it (it exits with a usage message if you don't pass one):

```bash
if bash scripts/build_firmware.sh --bootloader no 2>&1 | tee /dev/stderr | grep -q "SUCCESS"; then
  bash scripts/flash.sh
fi
```

### Flashing the bootloader

`flash.sh` above only flashes the firmware itself — it assumes a compatible bootloader is already on the device. The first time you set up a board (or after updating to a bootloader version the current one can't chain-load), you'll also need `scripts/flash_bootloader.sh`:

```bash
probe-rs download --chip STM32F407VG --binary-format bin --base-address 0x08000000 ./buddy/.dependencies/bootloader-mini-2.6.0/bootloader.bin
```

This flashes the bootloader binary that the buddy build pulled down as a dependency directly to the start of flash. You normally only need to run this once per board (or after a bootloader version bump) — day-to-day iteration just uses `build_and_flash.sh`.

With that all done. Let's give it a go:

```bash
bash scripts/build_and_flash.sh
```

This may take a bit of time on the first build so make sure you have a coffee/tea at the ready.

## Did it Work?

You should see a SUCCESS from the build process and the probe programming the device. The device should reboot and in the terminal you should see the logs and you will be looking for our `Hello from Rust` event.

```
...
19:19:06.509: 5.472s [INFO  - USBHost:6] MSC Device ready
19:19:06.509: 5.474s [INFO  - USBHost:6] MSC Device
19:19:10.993: 9.973s [INFO  - Network:11] Reconfigure
19:19:10.993: 9.973s [INFO  - derusting:0] Hello from Rust
19:19:10.993: 9.973s [ERROR - Network:11] mdns_resp_remove_netif: Not an active netif
...
```
