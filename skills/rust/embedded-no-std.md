# Embedded and `no_std` — Firmware, Bare Metal, and Constrained Targets

`no_std` removes the operating system, not the language. You keep traits, generics, iterators, `Result`, and the borrow checker; you lose the heap by default, threads, files, sockets, `std::time`, and panic-with-a-message.

Contents: What You Actually Lose · Skeleton · Interrupts and Shared State · Choosing an Execution Model · The HAL Layers · Flashing and Debugging · Size and Determinism · Traps.

## What You Actually Lose

| Gone | Replacement |
|---|---|
| `Vec`, `String`, `Box`, `HashMap` | `heapless::Vec`/`String`/`FnvIndexMap` (fixed capacity, no allocator), or the `alloc` crate with a `#[global_allocator]` |
| `std::time::Instant` | A hardware timer via the HAL, or `embassy_time` |
| `println!` | `defmt` (deferred formatting, tiny on-device cost) or `rtt-target` |
| Threads | Interrupts, or an executor (`embassy`, RTIC) |
| `std::error::Error` | `core::fmt::Debug` plus your own enums; the `Error` trait moved to `core` only recently and most of the ecosystem does not use it |
| Default panic handler | `#[panic_handler]` is mandatory, and it must diverge |
| `std::sync::Mutex` | `critical_section::Mutex` (disables interrupts) or a spinlock from the HAL |

`#![no_std]` still allows `alloc` if you provide an allocator (`embedded-alloc`). Decide deliberately: a heap in firmware means fragmentation and non-deterministic latency, which is exactly what the fixed-capacity types exist to avoid.

## Skeleton

```rust
#![no_std]
#![no_main]

use panic_probe as _;          // panic handler, linked for its side effect
use defmt_rtt as _;            // logging transport

#[cortex_m_rt::entry]
fn main() -> ! {
    let p = hal::init(Default::default());
    loop { /* ... */ }
}
```

- `#![no_main]` plus `#[entry]` because there is no runtime to call `main`. The entry function returns `!` — returning from it is a fault.
- The `use ... as _;` imports exist purely to link the panic handler and the logger. Deleting an "unused" import here breaks the build in a way the error message does not explain.
- `.cargo/config.toml` holds the target, the runner (`probe-rs run`), and the linker script flags. Building without it produces a host binary that does nothing.
- `memory.x` describes flash and RAM regions; a wrong origin or length produces a binary that flashes and hangs with no diagnostic.

## Interrupts and Shared State

- An interrupt handler can preempt `main` at any instruction. Sharing a value between them is exactly the data-race problem, on one core.
- The standard-safe pattern is `critical_section::Mutex<RefCell<Option<T>>>` in a `static`: the critical section disables interrupts, the `RefCell` catches reentrancy in debug, the `Option` handles "not initialized yet".
- Keep critical sections to a few instructions. Disabling interrupts is global latency for every other handler in the system.
- `static mut` is a footgun the language now discourages: taking a reference to it is unsound whenever an interrupt could also touch it. Editions from 2024 make `static mut` references a hard error.
- Atomics: not every target has them. `thumbv6m` (Cortex-M0) lacks compare-and-swap entirely; the `portable-atomic` crate polyfills with critical sections. Check before designing around a lock-free structure.

## Choosing an Execution Model

| Model | Shape | Fits |
|---|---|---|
| Superloop | `loop { poll(); }` plus interrupt flags | Small, single-purpose firmware |
| RTIC | Static priorities, compile-time-checked resource sharing, no allocation | Hard-real-time with several tasks |
| Embassy | `async`/`await` on a `no_std` executor, one stack, futures for peripherals | IO-heavy firmware (radio, USB, networking) |
| An RTOS with Rust bindings | Preemptive threads | Existing RTOS codebase you must fit into |

Embassy brings hosted async's cancellation rules to firmware: dropping a future is how timeouts work, and there is still no async `Drop`.

## The HAL Layers

- `embedded-hal` is the trait layer (`SpiBus`, `I2c`, `DelayNs`, `digital::OutputPin`); driver crates depend on the traits, not on your chip.
- A chip-specific HAL (`stm32f4xx-hal`, `rp2040-hal`, `esp-hal`) implements them; a PAC (peripheral access crate, generated from the vendor SVD) sits below with raw register access.
- Write drivers against `embedded-hal` traits: they become testable on the host with a mock implementation, which is the only way to unit-test firmware logic without hardware.
- `embedded-hal` 1.0 changed trait names and shapes from 0.2. Mixing a 0.2 driver with a 1.0 HAL produces trait-bound errors that look like the impl is missing; check versions first.

## Flashing and Debugging

| Task | Tool |
|---|---|
| Flash and run with logs | `cargo run` wired to `probe-rs run` as the runner |
| Interactive debugging | `probe-rs` with GDB, or the VS Code probe-rs extension |
| Log output | `defmt` over RTT: formatting stays on the host, so a log line costs bytes, not a formatter |
| Panic visibility | `panic-probe` prints the location over RTT; `panic-halt` prints nothing and just stops |
| Size breakdown | `cargo size -- -A`, `cargo nm --size-sort`, `cargo bloat` |
| Stack usage | `cargo-call-stack`, or fill RAM with a pattern and inspect the high-water mark |

Stack overflow in firmware silently corrupts memory rather than faulting, unless the MPU is configured to guard the stack. On Cortex-M, `flip-link` moves the stack so an overflow hits a guard region and faults immediately — a two-line change that converts a class of impossible bugs into an obvious one.

## Size and Determinism

```toml
[profile.release]
opt-level = "z"      # or "s"; measure both, "s" is sometimes smaller in practice
lto = "fat"
codegen-units = 1
panic = "abort"
strip = false        # keep symbols for defmt and stack analysis
```

- `debug = true` costs nothing in flash (debug info is not loaded onto the device) and makes every later investigation possible. Keep it on.
- Formatting machinery (`core::fmt`) is often the single largest contributor to binary size; `defmt` exists to avoid linking it.
- Floating point on a chip without an FPU pulls in soft-float routines. Check whether fixed-point arithmetic fits the problem before accepting the size.
- Determinism beats throughput in most firmware: prefer fixed-capacity types, avoid allocation, and measure worst-case interrupt latency rather than average.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Adding a crate that quietly depends on `std` | Link errors naming `std` symbols, far from the cause | `default-features = false`; check the crate declares `no_std` support |
| `static mut` shared with an interrupt | Unsound, and a hard error in recent editions | `critical_section::Mutex<RefCell<_>>` |
| A long critical section | Blocks every interrupt in the system | Copy the data out, release, then process |
| Blocking delay inside an interrupt handler | Stalls the whole system at that priority and below | Set a flag; do the work in the main loop or a lower-priority task |
| Assuming atomics exist | `thumbv6m` has no CAS; the error is a missing trait impl | `portable-atomic` |
| Testing only on hardware | Slow loop, no CI | Split logic from IO; test logic on the host against `embedded-hal` mocks |
| `panic-halt` in development | A crash looks identical to a hang | `panic-probe` with `defmt` |
