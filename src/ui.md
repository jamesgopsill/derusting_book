# Updating the UI

Every other chapter in [Decentralisation](./decentralisation.md) has been about what happens over the network or on disk — none of it is visible on the machine itself unless something patches the screen. This short chapter steps back from the specific ready-flag button we already built in [Marlin](./marlin.md) to look at the *technique* behind it: how derusting patches the Buddy GUI at all, so you can extend it yourself if you want the screen to show more than just online/offline.

## What's actually been changed

Worth being upfront about scope here: as of this chapter, the **only** UI patch in the whole codebase is `assets/screen_home.cpp`/`.hpp`, already covered in full in [Marlin](./marlin.md) — the home screen's Print button repurposed to toggle `derusting_ready_flag`, its label swapped between `ONLINE`/`OFFLINE`, and the header retitled to `DERUSTING`. There's no dashboard, no job queue view, no address-book display. If you came here looking for that, it doesn't exist yet — this chapter is about the pattern you'd use to add it.

## The pattern: patch, don't fork

Rather than maintaining a diverged copy of the whole Buddy GUI, derusting takes the smallest possible slice — one screen — and overwrites just that file at build time. [Scaffolding](./scaffolding.md) already set up the mechanism for patching C++ source into the `buddy` submodule; `scripts/build_firmware.sh` does the same thing for GUI files, just with `rsync` instead of `sed`:

```bash
# UI elements
rsync -c assets/screen_home.hpp buddy/src/gui/screen_home.hpp
rsync -c assets/screen_home.cpp buddy/src/gui/screen_home.cpp
```

Because `.gitmodules` sets `ignore = all` on the `buddy` submodule (see [Scaffolding](./scaffolding.md)), these overwritten files never show up as changes to commit inside `buddy/` — the *source of truth* is always `assets/screen_home.cpp`, and every build re-applies it fresh. If Prusa updates `screen_home.cpp` upstream, you pull the submodule forward, diff their new version against your patched one, and update `assets/screen_home.cpp` by hand — there's no merge conflict to resolve, because the patched file just gets stamped over on the next build regardless.

The pattern generalises to any other screen you might want to touch: copy the stock file from `buddy/src/gui/` into `assets/`, make your changes, and add a matching `rsync` line to `scripts/build_firmware.sh`.

## What a real screen file looks like

`screen_home.cpp`'s two derusting-specific additions are small and easy to spot — they're the only lines marked `// JG:` in an otherwise-stock Prusa file:

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
```

and, in the constructor, `home_screen_instance = this;` — keeping a raw pointer to the live screen instance around so `derusting_update_ui()` (called from Rust, via `marlin::set_offline()` — see [Marlin](./marlin.md)) has something to call `refresh_ready_btn()` on. That's the whole trick for pushing a state change from Rust onto the screen: keep a static pointer to the live widget, and give Rust an `extern "C"` function that calls back into it.

`w_labels[0].Invalidate()` is what actually triggers a redraw — the Buddy GUI framework only repaints a widget once something marks it dirty; setting `SetText` alone doesn't do that.

## An exercise, not a shipped feature

If you want to go further than this book's code does today, the natural next step is showing more machine state on the home screen — for example, the number of jobs currently on the [Job Ledger](./ledger.md), or how many peers are in the [Address Book](./address_book.md). The shape of the change would follow the same pattern as `derusting_ready_flag`:

1. Add a new `extern "C"` accessor on the Rust side (in `marlin.rs` or a small new module) that reads whatever shared state you want to expose — the ledger and address book are both behind `embassy_sync::Mutex`es, so you'd need a way to read them synchronously from a GUI callback rather than `.await` them (a `try_lock()` that falls back to "unknown" on contention would be the simplest starting point).
2. Add a small `w_labels`-style text widget (or reuse the existing footer) in `screen_home.cpp`, and a `refresh_*` method that formats and sets it.
3. Call that `refresh_*` method from wherever the state actually changes — the same way `derusting_update_ui()` is called from `set_offline()` today.

This isn't built in the current source, so there's no "did it work?" log line or screenshot to check it against here — it's left as a genuine exercise if you want to practice the pattern rather than just read about it.
