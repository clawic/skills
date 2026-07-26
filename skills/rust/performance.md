# Performance — Measure First, and Measure the Right Binary

Rule zero is SKILL.md rule 4: only release builds carry information. Rule one is that the fix is almost never the thing you guessed — in Rust the top four findings are allocation in a loop, an unnecessary `clone`, a hash map where a `Vec` would do, and IO or lock waiting misread as CPU time.

## Getting a Usable Profile

```toml
[profile.release]
debug = true          # symbols; does not change the generated code
```

| Tool | Answers |
|---|---|
| `cargo flamegraph` (perf/dtrace) | Where wall time goes, with inlined frames attributed |
| `perf stat` | Whether you are compute-bound, memory-bound, or branch-mispredicting |
| `samply` / Instruments / VTune | Sampling profile with a UI, no root on macOS |
| `heaptrack`, `dhat-rs`, `valgrind --tool=dhat` | Allocation count and volume — the number that most often explains a Rust regression |
| `criterion` | Statistically sound A/B of a specific function |
| `cargo bench` with libtest | Nightly-only; criterion is the stable answer |
| `hyperfine` | End-to-end CLI timing including startup and IO |

- Inlining destroys naive stack traces: build with `debug = true` and let the profiler read the inline tables rather than adding `#[inline(never)]` everywhere.
- Benchmark inputs must escape optimization: `criterion::black_box(input)`, or LLVM deletes the computation whose result you never use and reports a nanosecond.
- Compare like with like: same machine, no other load, CPU frequency scaling pinned if you care about small deltas. A 3% difference between two runs on a laptop is noise.

## Allocation

- `Vec::with_capacity` / `String::with_capacity` whenever the size is known within a factor of two. Reallocation copies.
- Reuse buffers across loop iterations: `buf.clear()` keeps the capacity, `buf = Vec::new()` throws it away.
- `format!` allocates. In a loop, `write!(&mut s, ...)` into a reused `String` instead.
- `collect()` into an intermediate `Vec` only to iterate it once is pure allocation (SKILL.md Traps).
- `Box<dyn Trait>` per item in a hot collection means one allocation and one indirection per item; an enum over the known variants removes both.
- Swap the global allocator (`mimalloc`, `jemalloc`) before restructuring code when the profile shows allocator time — it is a two-line change and often the largest single win in allocation-heavy services.
- `Arc::clone` is an atomic increment, not free but cheap; `String::clone` is an allocation plus a memcpy. Treat them differently when reading a profile.

## Compile-Time Settings That Change Runtime

| Setting | Effect | Cost |
|---|---|---|
| `opt-level = 3` (release default) | Full optimization | Compile time |
| `lto = "thin"` | Cross-crate inlining; commonly a few percent, occasionally much more | Noticeably longer link |
| `lto = "fat"` | Maximum cross-crate inlining | Long link, high peak memory (the usual cause of CI OOM) |
| `codegen-units = 1` | Better optimization within the crate | Loses build parallelism |
| `panic = "abort"` | Smaller binary, no landing pads | No `catch_unwind`, breaks `#[should_panic]` tests |
| `target-cpu=native` in `RUSTFLAGS` | Uses the build machine's instruction set | The binary may not run on other machines |
| PGO (`-Cprofile-generate`/`-Cprofile-use`) | Real gains on branch-heavy code | A two-stage build and a representative workload |

Apply them in that order and re-measure after each; stacking all of them blind buys build time you cannot attribute.

## Code-Level Wins, in Order of Frequency

1. **Delete a clone.** Follow the borrow ladder (SKILL.md) rather than cloning to compile.
2. **Hoist work out of the loop.** Regex compilation, allocation, lock acquisition, `to_string()` on a constant.
3. **Iterators over manual indexing.** They eliminate the bounds check that `v[i]` keeps; `get_unchecked` is almost never necessary once the loop is an iterator.
4. **Right collection.** Linear scan over a small `Vec` beats a `HashMap` well past the size most people assume, because of cache locality and hashing cost.
5. **Right hasher.** Only for internal keys, never for untrusted input.
6. **Batch IO.** `BufReader`/`BufWriter` around any file or socket; an unbuffered `write!` per line is a syscall per line.
7. **`&[u8]` over `String` in parsers.** Skip UTF-8 validation until you actually need text.
8. **Reduce monomorphization** when compile time or binary size is the constraint: keep the generic wrapper tiny and have it call a non-generic inner function immediately.

## Reading a Rust Profile

| Symptom in the profile | Usual cause |
|---|---|
| Time in `memcpy`, `__rust_alloc`, `free` | Allocation churn or clone-heavy code |
| Time in `core::ptr::drop_in_place` | Dropping a large tree of owned data; consider arena or reuse |
| Time in hashing (`SipHash`, `FxHash`) | Hot map lookups; wrong hasher or wrong collection |
| Time in `Mutex::lock` / futex | Contention — shrink critical sections |
| Flat profile, no hotspot, high IPC | You are memory-bound; improve layout before instructions |
| Flat profile, low CPU | You are waiting on IO or a lock, not computing |

## Data Layout

- Struct fields are reordered by the compiler for packing unless `#[repr(C)]`. Do not hand-order fields for size — do it for cache locality of fields accessed together.
- Struct-of-arrays over array-of-structs when a loop touches one field of many elements; this is the single largest layout win and it is invisible to the profiler until you try it.
- `Box<[T]>` over `Vec<T>` for a collection that never grows: one word smaller and it documents the intent.
- Small-size optimizations (`smallvec`, `compact_str`, `arrayvec`) pay when the common case fits inline and the allocation is the measured cost — not before.
- Enum size is the largest variant plus a tag: one 200-byte variant makes every value 200 bytes. `Box` that variant and the enum shrinks to a pointer.

## Build Time (a different problem with different fixes)

`cargo build --timings` produces an HTML graph naming the crates on the critical path. Then, in order of payoff:

1. Faster linker: `lld` or `mold` in `.cargo/config.toml`. Often the largest single improvement for incremental builds of large binaries.
2. `cargo check` in the edit loop; `cargo clippy` only before committing.
3. `[profile.dev] debug = 1` (line tables only) cuts link time materially with little debugging loss.
4. `[profile.dev.package."*"] opt-level = 3` — optimize dependencies once, keep your own crate fast to rebuild. The standard fix for a slow debug build of a compute-heavy program.
5. Split the crate: rustc parallelism is limited within one crate, so a workspace of several crates rebuilds less on each edit.
6. Cut features and dependencies; `cargo tree --duplicates` and `cargo machete` find what nobody uses.
7. `sccache` for CI and for switching branches; useless for edit-compile loops on one branch.

## Binary Size

`opt-level = "z"`, `lto = "fat"`, `codegen-units = 1`, `panic = "abort"`, `strip = "symbols"` in a dedicated profile, then `cargo bloat --release --crates` to see who is actually big. For wasm the list is the same plus `wasm-opt -Oz` after the build; for firmware keep `strip = false`, since `defmt` and stack analysis need the symbols and debug info costs no flash.
