# WebAssembly — Browser, Node, and WASI Targets

Rust compiles to wasm well, and then the environment removes things you assumed: there is no filesystem, no clock you can trust, no threads by default, and blocking is illegal in the browser's main thread.

## Pick the Target First

| Target | Environment | Use |
|---|---|---|
| `wasm32-unknown-unknown` | Browser or a JS host, no OS | Web apps and libraries via `wasm-bindgen` |
| `wasm32-wasip1` | WASI runtimes (wasmtime, wasmer, edge platforms) | Server-side wasm with files, env, clock, sockets |
| `wasm32-wasip2` | WASI 0.2 component model | Component-model hosts; toolchain support is younger |
| `wasm32-unknown-emscripten` | Emscripten runtime | Porting existing C/C++ toolchains, rarely a first choice |

`rustup target add wasm32-unknown-unknown` — then `cargo build --target wasm32-unknown-unknown`. The `wasm32-wasi` target was renamed `wasm32-wasip1` (`rust >=1.78`); the old name is what older tutorials and CI configs still use.

## What Breaks on `wasm32-unknown-unknown`

| Assumption | Reality | Fix |
|---|---|---|
| `std::time::Instant::now()` | Panics — there is no clock in this target | `web_sys::window().performance().now()`, or the `instant`/`web-time` crates |
| `std::thread::spawn` | Panics; no threads without the atomics feature and shared memory | `wasm_bindgen_futures::spawn_local`, or web workers |
| `std::fs`, `std::net` | Return errors or fail to link | `fetch` via `web_sys`, or move the work server-side |
| Blocking on a future | Deadlocks the single-threaded event loop | Everything async, driven by the JS event loop |
| `rand`/`getrandom` | No entropy source by default | Enable `getrandom`'s `js` feature (older versions) or configure the backend in `.cargo/config.toml` (newer ones) |
| A panic | Shows as `unreachable executed` with no message | `console_error_panic_hook` in your init function |
| `std::process::exit` | No process to exit | Return a value or throw a JS exception |

## `wasm-bindgen` Rules

```rust
#[wasm_bindgen]
pub fn parse(input: &str) -> Result<JsValue, JsValue> { ... }
```

- The `wasm-bindgen` crate version and the `wasm-bindgen-cli` version must match **exactly**. The mismatch error is long and mentions schema versions; that is what it means, every time. Pin the CLI with `cargo install --locked wasm-bindgen-cli --version <same>`.
- Use `wasm-pack build --target web|bundler|nodejs` rather than invoking the CLI by hand; the target flag decides what glue is generated and choosing the wrong one produces import errors in the host, not in Rust.
- Crossing the boundary costs: every `String` is copied and re-encoded, every `Vec` is copied. Batch the calls. A per-pixel or per-row call into wasm is slower than doing the whole loop in JS.
- `serde-wasm-bindgen` converts structs to JS objects without going through a JSON string; `JsValue::from_serde` (the old path) serializes to JSON and back.
- Return `Result<T, JsValue>` and the error becomes a thrown JS exception. Map your error type once at the boundary rather than at every call.
- `#[wasm_bindgen(start)]` runs on module instantiation — the right place for the panic hook and logger setup.

## Async in the Browser

- `wasm_bindgen_futures::spawn_local` runs a future on the JS microtask queue. There is one thread: a future that computes for 50 ms freezes the page.
- `JsFuture::from(promise)` converts a JS promise into a Rust future; `future_to_promise` goes the other way.
- No tokio. Timers come from `gloo-timers` or `setTimeout` through `web_sys`; `tokio::time` will not work on this target even if it compiles.
- Long computations belong in a web worker (`gloo-worker`, or a second wasm instance) with message passing, exactly as they would in JS.

## Size (the metric users actually feel)

```toml
[profile.release]
opt-level = "z"
lto = "fat"
codegen-units = 1
panic = "abort"
strip = true
```

Then, in order of payoff:

1. `wasm-opt -Oz` (from binaryen) — `wasm-pack` runs it by default; a manual build must run it or leave a significant fraction of the size on the table.
2. `twiggy top -n 20` to see what is actually large. The answer is often formatting machinery, a panic message table, or one dependency pulling in `regex` or `chrono`.
3. Remove `std::fmt`-heavy paths and `#[derive(Debug)]` on large types in release builds.
4. `default-features = false` on every dependency and re-add what you need.
5. `opt-level = "s"` sometimes beats `"z"` after `wasm-opt`; measure both rather than assuming.

Compressed size over the wire is what matters: compare gzip or brotli sizes, not the raw `.wasm` bytes.

## WASI Server-Side

- `wasm32-wasip1` gives you files, env vars, a clock, and (with recent runtimes) sockets — much of `std` works unchanged, which is the point of the target.
- Capabilities are granted by the host: a preopened directory, an allowed set of env vars. Code that opens an arbitrary path fails with a permission error even though the path exists.
- No threads in wasip1. Concurrency is async on a single thread, or multiple instances.
- Component model (`wasip2`, WIT interfaces, `cargo component`) is where the ecosystem is moving; check whether your host runtime supports it before committing, since tooling maturity varies more than the specification does.

## Testing

- `wasm-bindgen-test` runs tests in a headless browser (`wasm-pack test --headless --firefox`) or in Node. `#[wasm_bindgen_test]` replaces `#[test]`.
- Keep the pure logic free of `wasm_bindgen` types so the bulk of the suite runs natively with `cargo test` — fast, debuggable, and the same code.
- CI: build for wasm on every PR even if you only test natively. The target-specific breakages above are compile or link failures, and they appear the moment a dependency updates.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `wasm-bindgen` crate and CLI at different versions | Generated glue speaks a different schema | Pin both; `cargo install --locked` |
| `Instant::now()` anywhere in the dependency graph | Panics at runtime on `wasm32-unknown-unknown` | `web-time` as a drop-in, or feature-gate the timing code |
| Chatty JS↔wasm calls in a loop | Boundary crossing and copying dominate | One call with a batch of data |
| Forgetting `console_error_panic_hook` | Every panic is `unreachable executed` | Install it in the `start` function |
| Shipping without `wasm-opt` | Substantially larger download | `wasm-pack`, or run binaryen in the build |
| Assuming `getrandom` works | No entropy source is configured by default | Enable the JS backend for the target |
| A dependency that spawns threads | Compiles, then traps at runtime | Audit with `cargo tree`; most crates gate this behind a feature |
