# FFI — Calling C from Rust and Rust from C

Two directions, one discipline: at the boundary, nothing is checked. Decide explicitly who owns each allocation, what happens on panic, and which layout both sides agree on — those three answers are the whole design.

## Layout

- Rust's default struct layout is unspecified and the compiler reorders fields. Any type crossing the boundary needs `#[repr(C)]`.
- `#[repr(transparent)]` for a newtype that must have the identical ABI as its single field — the right way to give a raw handle a distinct Rust type at zero cost.
- Enums: `#[repr(C)]` for a C-compatible tagged union, or `#[repr(i32)]`/`#[repr(u8)]` for a plain C enum. A fieldless `#[repr(u8)]` enum can be passed as an integer, but a value outside the declared discriminants is instantly UB — validate on the way in with `TryFrom`.
- `bool` is one byte with only 0 and 1 valid; C code writing 2 into it produces UB. Take a `u8` and convert.
- Rust `&T`/`&mut T` are guaranteed non-null and are ABI-compatible with C pointers; `Option<&T>` and `Option<NonNull<T>>` are pointer-sized with `None` as null, which is how you express a nullable C pointer without raw pointers.
- Never expose `String`, `Vec`, `&str`, or a trait object across the boundary. Their layout is not stable and not C-compatible: use pointer plus length.

## Strings

| Direction | Type | Trap |
|---|---|---|
| Rust → C | `CString::new(s)?` then `.as_ptr()` | Fails on an interior nul; the `CString` must outlive the pointer, so binding it to a variable is mandatory |
| C → Rust, borrowed | `CStr::from_ptr(p)` (unsafe) | The lifetime is invented — tie it to something real immediately |
| C → Rust, owned | `CStr::from_ptr(p).to_owned()` / `.to_str()?` | `to_str` validates UTF-8 and can fail; C strings are byte strings, not text |
| Windows wide strings | `OsStrExt`/`OsStringExt` from `std::os::windows::ffi` | UTF-16, and not necessarily well-formed |

The classic dangling pointer, worth reading twice: `let p = CString::new(s)?.as_ptr();` — the `CString` is a temporary and is dropped at the end of that statement, so `p` dangles on the next line. Bind it: `let c = CString::new(s)?; let p = c.as_ptr();`.

## Ownership Across the Boundary

Pick one rule per allocation and document it in the header and the Rust doc comment:

- **Rust allocates, Rust frees**: export a `free_thing(ptr)` function; C must call it. Reconstruct with `Box::from_raw` (or `Vec::from_raw_parts` with the original length **and** capacity) and let it drop.
- **C allocates, C frees**: Rust only borrows; never pass a C pointer to `Box::from_raw`. Mixing allocators is a heap corruption bug that surfaces far from the cause.
- Hand out an opaque handle rather than a struct layout whenever you can: `Box::into_raw(Box::new(state)) as *mut c_void` keeps the Rust type private and makes the API versionable.
- `Box::into_raw` leaks deliberately; forgetting the matching `from_raw` is a leak, calling it twice is a double free. Every `into_raw` needs a named counterpart in the same module.

## Panics

- Unwinding out of an `extern "C"` function aborts the process (`rust >=1.71`) instead of being UB, which is safer but is still a crash your caller cannot handle.
- Wrap any exported function whose body can panic: `std::panic::catch_unwind(AssertUnwindSafe(|| ...))` and map the panic to an error code.
- `extern "C-unwind"` when unwinding must cross the boundary deliberately (a C++ caller with its own handling); both sides must agree.
- `panic = "abort"` in the profile makes `catch_unwind` useless — check the profile before relying on it.

## Exporting Rust to C

```rust
#[unsafe(no_mangle)]                 // `#[no_mangle]` in editions before 2024
pub extern "C" fn parse_config(ptr: *const c_char, out: *mut Config) -> i32 {
    let result = std::panic::catch_unwind(|| { /* ... */ });
    match result { Ok(Ok(())) => 0, Ok(Err(e)) => e.code(), Err(_) => -1 }
}
```

- `crate-type = ["cdylib"]` for a shared library C loads, `["staticlib"]` for static linking. `rlib` is Rust-only.
- Return an integer status and write results through out-parameters; that is the ABI C callers expect, and it keeps errors representable without exceptions.
- Generate the header with `cbindgen` and check it into version control — a hand-maintained header drifting from the Rust signature is a segfault waiting for a release.
- Every exported symbol is global: prefix names with your library's name or expect a collision.

## Importing C into Rust

```rust
#[link(name = "z")]
extern "C" { fn compress(dest: *mut u8, dest_len: *mut u64, src: *const u8, src_len: u64) -> i32; }
```

- `bindgen` generates these from the real headers at build time; hand-written declarations drift silently and a wrong signature is UB, not a compile error.
- Convention: crate `foo-sys` holds only the raw bindings and the link directive, crate `foo` holds the safe wrapper. Keeping them separate is what lets the ecosystem share one linkage.
- The build script emits `cargo::rustc-link-lib=` and `cargo::rustc-link-search=`; use `pkg-config` or `vcpkg` crates rather than hardcoding paths.
- Thread safety of the C library is not visible to the compiler. Do not `unsafe impl Send`/`Sync` on a handle until the library's documentation says it is safe.
- Callbacks into Rust: pass `extern "C" fn` plus a `void*` user-data pointer. A Rust closure is not an `extern "C" fn`; box it, pass the raw pointer as user data, and reconstruct inside a trampoline function.

## Verifying the Boundary

| Check | Tool |
|---|---|
| Layout and size agree with C | `assert_eq!(size_of::<T>(), N)` in a test; `bindgen`'s generated layout tests |
| No UB in the Rust side | Miri cannot cross FFI — test the pure-Rust logic separately |
| No memory errors across the call | `-Zsanitizer=address` on nightly, or Valgrind on the final binary |
| No leaks | Run the C test suite under Valgrind or LeakSanitizer, not just the Rust tests |
| Symbols actually exported | `nm -D libfoo.so \| grep parse_config` |

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `CString::new(s)?.as_ptr()` in one expression | The `CString` drops at the semicolon; the pointer dangles | Bind the `CString` to a variable first |
| Passing `&str` or `String` to C | Not nul-terminated, layout not C-compatible | `CString`, or pointer plus length |
| Freeing a Rust allocation with C's `free` | Different allocators; heap corruption | Export a free function from Rust |
| Assuming `char` maps to `c_char` | Rust `char` is a 4-byte scalar value | `c_char` (which is signed or unsigned depending on the platform) |
| Storing a `CStr` borrowed from C beyond the call | The lifetime was invented by `from_ptr` | Copy to an owned `CString`/`String` immediately |
| `#[repr(Rust)]` struct through the boundary | Field order is not what the header says | `#[repr(C)]` on every boundary type |
| Letting a panic escape an exported function | Aborts the caller's process | `catch_unwind` and an error code |
