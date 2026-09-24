# Embassy Async Runtime in FreeRTOS

By the end of the [Scaffolding](./scaffolding.md) chapter we had a Rust static library linked into the Buddy firmware, and `derusting_main()` logging a single `Hello from Rust` line before returning. That's fine for a one-shot log line, but everything we build from here on (the UDP heartbeat, the TCP server, background job management) needs to run concurrently, wait on timers, and await I/O — none of which you want to hand-roll with blocking FFI calls sat on the firmware's own boot thread.

This chapter gets an [Embassy](https://embassy.dev/) async executor running as its own dedicated FreeRTOS task. By the end you'll flash the device and watch a task boot, sleep for 5 seconds, and wake back up — our "hello world" for concurrent Rust running inside the firmware.

## Why a dedicated FreeRTOS task?

Embassy usually pairs with a HAL that owns the whole microcontroller and drives the executor from an interrupt (e.g. `SysTick`). We don't have that luxury: the STM32F407 is already fully owned by the Buddy firmware's own FreeRTOS scheduler, running the GUI, USB host stack, motion control, and everything else. We're a guest.

So instead we:

1. Create a normal (if unusually long-lived) FreeRTOS task.
2. Run a raw Embassy `Executor` inside that task's own loop.
3. Give Embassy a custom time driver backed by FreeRTOS ticks, so `embassy_time::Timer` works without a hardware timer of our own.

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
  features = ["tick-hz-1_000", "generic-queue-32"]
}
static_cell = "2"
```

## The `free_rtos` module

We group everything that talks to FreeRTOS directly under a new `free_rtos` module:

```
/free_rtos
  - bindings.rs     # extern "C" FreeRTOS API bindings
  - executor.rs     # the Embassy executor, driven from inside a FreeRTOS task
  - mod.rs
  - task.rs         # a safe wrapper for creating FreeRTOS tasks
  - time_driver.rs   # an embassy_time_driver impl backed by FreeRTOS ticks
```

`bindings.rs` is a straightforward `extern "C"` block over the FreeRTOS functions we need (`xTaskCreate`, `xTaskCreateStatic`, `vTaskDelay`, …) — nothing interesting to show here, it just mirrors FreeRTOS's own headers.

### A safe `Task` wrapper

`task.rs` wraps FreeRTOS task creation. We use the **static** variant, `new_static`, because we want to hand FreeRTOS a stack and TCB (task control block) that we've already reserved at compile time, rather than have it allocate them from the heap:

```rust
impl Task {
    pub fn new_static(
        name: &CStr,
        fcn: TaskFunction_t,
        priority: u32,
        stack_buf: &mut [u8],
        tcb_buf: &mut [u8],
    ) -> Self {
        let task = unsafe {
            xTaskCreateStatic(
                fcn,
                name.as_ptr(),
                (stack_buf.len() / 4) as u16,
                ptr::null_mut(),
                priority,
                stack_buf.as_mut_ptr(),
                tcb_buf.as_mut_ptr(),
            )
        };
        Self { inner: task }
    }
}
```

### A time driver backed by FreeRTOS

Embassy's `embassy-time` crate expects *something* to implement `embassy_time_driver::Driver` so `Timer::after_secs(..)` etc. have a clock and a wake-up mechanism to work with. `time_driver.rs` implements that on top of FreeRTOS's own tick count, and we register it as a global:

```rust
embassy_time_driver::time_driver_impl!(static DRIVER: FreeRtosTimeDriver = FreeRtosTimeDriver {
    queue: Mutex::new(RefCell::new(Queue::new())),
    timekeeper: AtomicU64::new(u64::MIN),
    free_rtos_now: AtomicU32::new(u32::MIN),
});
```

### The executor itself

`executor.rs` wraps Embassy's `raw::Executor` and runs it in a plain loop, blocking the FreeRTOS task on `DRIVER.wait_for_interrupt_or_timeout()` between polls rather than spinning:

```rust
impl FreeRtosTaskExecutor {
    pub fn new(task: *mut BaseType_t) -> Self {
        Self {
            inner: raw::Executor::new(task as _),
            not_send: PhantomData,
        }
    }

    pub fn run(&'static mut self, init: impl FnOnce(Spawner)) -> ! {
        log_info!("Spawning Tasks");
        init(self.inner.spawner());

        loop {
            unsafe { self.inner.poll() };
            DRIVER.wait_for_interrupt_or_timeout();
        }
    }
}
```

Embassy needs a way to wake this task up when a timer fires or a future is ready to be polled again. That's the job of the `__pender` export — Embassy calls it, and it in turn notifies our FreeRTOS task (from an ISR or from thread context, depending on where the wake happened):

```rust
#[unsafe(export_name = "__pender")]
pub fn __pender(context: *mut c_void) {
    let task_handle = context as *mut BaseType_t;
    // ...notifies task_handle via xTaskGenericNotify / xTaskGenericNotifyFromISR
}
```

## Wiring it into `lib.rs`

`derusting_main()` reserves a static stack and TCB, then creates the task:

```rust
const STACK_BYTES: usize = 1024 * 10;
static mut RTOS_STACK: [u8; STACK_BYTES] = [0u8; STACK_BYTES];
static mut RTOS_TCB: [u8; 128] = [0u8; 128];

#[unsafe(no_mangle)]
pub unsafe extern "C" fn derusting_main() {
    log_info!("derusting_main()");
    let stack = unsafe { RTOS_STACK.as_mut_slice() };
    let tcb = unsafe { RTOS_TCB.as_mut_slice() };
    let _ = Task::new_static(c"Embassy", embassy, 1, stack, tcb);
}
```

That task's entry point is `embassy()`. This is our hello-world moment — it logs, sleeps for 5 seconds using `vTaskDelay` (still plain FreeRTOS, no Embassy yet), then logs again to prove the task is alive and scheduling correctly, before finally booting the Embassy executor:

```rust
#[unsafe(no_mangle)]
unsafe extern "C" fn embassy(_pv_parameters: *mut pvParameters) -> ! {
    log_info!("Rust Embassy Task. Waiting 5secs...");
    unsafe {
        vTaskDelay(5 * 1_000);
    }
    log_info!("Rust Waking up...");

    let current_task = unsafe { xTaskGetCurrentTaskHandle() };
    if current_task.is_null() {
        log_error!("We should only be called within a FreeRTOS task.");
    }

    log_info!("Initialising Executor");
    let executor = EXECUTOR.init(FreeRtosTaskExecutor::new(current_task as _));

    executor.run(|spawner| match embassy_main(spawner) {
        Ok(t) => spawner.spawn(t),
        Err(e) => log_error!("Spawn Error: {e}"),
    })
}
```

`embassy_main` is our top-level async task, spawned once the executor is running. For now it can be as small as you like — an empty `async fn embassy_main(_spawner: Spawner) {}` is enough to prove the plumbing works. We'll grow it steadily over the next three chapters as we bring up UDP, TCP and the filesystem.

> [!NOTE]
> The FreeRTOS task created here is never meant to return — `embassy()`'s return type is `!`. `executor.run(..)` loops forever polling the executor, and that's exactly what we want: it's the permanent home for every async task the rest of the book adds.

## Did it work?

Rebuild and flash as before:

```bash
bash scripts/build_and_flash.sh
```

Watch the device's log console. You should see, in order:

```
[INFO  - derusting:0] derusting_main()
[INFO  - derusting:0] Rust Embassy Task. Waiting 5secs...
(... 5 second pause ...)
[INFO  - derusting:0] Rust Waking up...
[INFO  - derusting:0] Initialising Executor
[INFO  - derusting:0] Spawning Tasks
```

That five-second gap between the two log lines is the tell: it's `vTaskDelay` proving our FreeRTOS task was created correctly, scheduled, and put to sleep and woken up on time — a real task is alive, not just a function called inline from the firmware's own boot sequence. Everything from here on runs inside it.
