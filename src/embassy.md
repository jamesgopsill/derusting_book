# Embassy Async Runtime in FreeRTOS

[Commit](https://github.com/jamesgopsill/derusting/commit/f46d4e24490b59a6bfac85c87a2568206e336e4a)

- /free_rtos
  - alloc.rs
  - bindings.rs
  - executor.rs
  - mod.rs
  - task.rs
  - time_driver.rs
  
## Dependencies

```toml
cortex-m = {
  version = "0.7",
  features = ["critical-section-single-core"]
}
critical-section = "1"
embassy-executor = {
  version = "0.10",
  features = ["embassy-time-driver"]
}
embassy-time-queue-utils = "0.3"
embassy-time-driver = "0.2"
embassy-time = {
  version = "0.5",
  features = ["tick-hz-1_000", "generic-queue-8"]
}
portable-atomic = "1"
static_cell = "2"
```

```Rust
embassy_time_driver::time_driver_impl!(static DRIVER: FreeRtosTimeDriver = FreeRtosTimeDriver {
    queue: Mutex::new(RefCell::new(Queue::new())),
    timekeeper: AtomicU64::new(u64::MIN),
    free_rtos_now: AtomicU32::new(u32::MIN),
}); 
```


**NOTE:** Task should never return
