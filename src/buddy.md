# Buddy Firmware

Before we start [Scaffolding](./scaffolding.md) a project around it, it's worth knowing what the Buddy firmware actually is and how it's put together — not in exhaustive detail, but enough that when later chapters say "patch this file" or "bind to this library", you know roughly where you are in someone else's (very large) codebase.

## What it is

[Prusa-Firmware-Buddy](https://github.com/prusa3d/Prusa-Firmware-Buddy) is Prusa's own open-source firmware for their 32-bit ARM printers — the Mini, the MK3.5/3.9/4, the XL, the CORE One — built around a customised [Marlin](https://marlinfw.org/) (the print/motion engine almost every open-source 3D printer runs some variant of), an RTOS to keep the GUI, networking, and motion control running concurrently, and Prusa's own GUI and connectivity layers on top. It's large — the repository is a proper multi-year commercial firmware project, not a toy — but derusting only ever needs to touch a handful of specific corners of it.

Rather than fork it, derusting pulls it in as a git submodule (see [Scaffolding](./scaffolding.md)) and patches a small number of files into it at build time, so upstream updates can still be pulled cleanly without derusting's changes fighting them.

## The pieces derusting actually touches

The firmware tree (`buddy/`) is organised as a fairly conventional CMake project, with `buddy/src/` holding the firmware's own code and `buddy/lib/` holding vendored third-party libraries. The parts relevant to this book:

| Component | Where it lives | Covered in |
|---|---|---|
| FreeRTOS (the RTOS everything else runs inside) | `lib/Middlewares/Third_Party/FreeRTOS`, wrapped by `src/freertos` | [Embassy Async Runtime in FreeRTOS](./embassy.md) |
| lwIP (the TCP/IP stack) | `lib/Middlewares/Third_Party/LwIP` | [LWIP UDP](./lwip_udp.md), [LWIP TCP](./lwip_tcp.md) |
| FatFs (the FAT filesystem driver for the USB stick) | `lib/Middlewares/Third_Party/FatFs`, wrapped by `src/buddy/fatfs.cpp` | [FS](./fs.md) |
| Marlin (the print/motion engine) | `lib/Marlin`, with printer-specific gcode handlers in `src/marlin_stubs` | [Marlin](./marlin.md) |
| The GUI framework (screens, widgets, the home screen) | `src/gui` | [Updating the UI](./ui.md) |
| The firmware's own entry point | `src/buddy/main.cpp` | [Scaffolding](./scaffolding.md) — this is the file that gets patched to call `derusting_main()` |

Everything else in the tree — `can/` (CANbus, for the MMU and toolchanger boards), `puppies/`/`puppy/` (peripheral boards), `connect/` (Prusa Connect, their cloud service — explicitly disabled in derusting's build via `-DCONNECT:STRING=NO`), `mmu2/`, `resources/`, and so on — derusting never touches at all. You can safely ignore it while working through this book.

## The build system

The firmware builds via CMake, wrapped by a Python script (`utils/build.py`) that handles picking a target printer and toolchain:

```bash
python utils/build.py --preset mini --build-type release --bootloader no
```

`--preset mini` selects the Prusa Mini specifically — the same tree builds firmware for every supported printer, with per-model configuration living under `utils/presets`. [Scaffolding](./scaffolding.md) walks through wiring `scripts/build_firmware.sh` around this same command, with a couple of extra flags (`-DWUI:STRING=YES`, `-DCONNECT:STRING=NO`, …) to enable the web UI derusting's TCP server needs and disable the parts of the stock firmware derusting doesn't want running alongside it.

## Next

With a rough map of the territory, [Prerequisites](./prerequisites.md) covers what you'll need installed and wired up to actually build and flash any of this, before [Scaffolding](./scaffolding.md) gets a Rust library linked into it for the first time.
