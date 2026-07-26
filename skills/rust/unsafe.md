# Unsafe — What It Actually Permits, and What Still Kills You

`unsafe` does not disable the borrow checker, the type system, or any lint. It permits exactly five operations, and it transfers the obligation to uphold Rust's rules from the compiler to you.

## The Five Permissions

1. Dereference a raw pointer
2. Call an `unsafe` function (including every `extern` function)
3. Access or modify a mutable `static`
4. Implement an `unsafe` trait (`Send`, `Sync`, and others)
5. Access a `union` field

Everything else inside an `unsafe` block is checked exactly as before. If your `unsafe` block is fifty lines, forty-five of them did not need to be there — narrow the block to the operation that requires it, so review has something small to look at.

## The Rules You Now Enforce

Violating any of these is undefined behavior, which means the optimizer may do anything, including producing code that works today and fails after an unrelated change:

- References must always be aligned, non-null, and point to initialized memory of the right type — including for the instant a reference merely exists, before any read.
- `&mut T` is **unique**. Creating a second `&mut` to the same memory, or a `&mut` while a `&` exists, is UB even if neither is ever used.
- No mutation through a `&T` unless the data is inside `UnsafeCell`. This is why `Cell`/`RefCell`/`Mutex` all wrap it.
- Values must be valid for their type: no `bool` that is not 0 or 1, no `char` outside Unicode scalar range, no null `&T` or `Box<T>`, no uninitialized bytes read as an integer.
- No data races. `unsafe` does not exempt you from the memory model.
- Pointers carry provenance: a pointer derived from one allocation may not be used to reach another, even at the same numeric address.

"It works in release" is not evidence of correctness — UB frequently manifests only after an inlining decision changes three refactors later.

## Raw Pointers

- `*const T` and `*mut T` may be null, dangling, unaligned, and aliased. They carry no lifetime, which is the whole point and the whole danger.
- Creating one is safe; dereferencing is not. `let p = &x as *const _;` needs no block.
- `ptr::null()`, `is_null()`, `NonNull<T>` when the invariant is "never null" — `NonNull` also gives you the niche optimization so `Option<NonNull<T>>` is pointer-sized.
- Use `ptr::read`, `ptr::write`, `copy_nonoverlapping` rather than dereference-and-assign when the destination may be uninitialized: assignment drops the old value, and dropping uninitialized memory is UB.
- `addr_of!` / `addr_of_mut!` take a pointer to a field **without** creating an intermediate reference — required for packed structs and uninitialized memory, where the reference itself would already be UB.
- Never conjure a reference from an integer address. Casting a pointer to `usize` and back loses provenance; keep the original pointer.

## `MaybeUninit`

```rust
let mut buf: [MaybeUninit<u8>; 1024] = unsafe { MaybeUninit::uninit().assume_init() };
```

- `mem::uninitialized()` and `mem::zeroed()` are UB for most types and are deprecated for exactly that reason. `MaybeUninit` is the only correct tool.
- `assume_init()` asserts the value is initialized — the assertion is yours, and it is where the bug lives when there is one.
- Reading even one uninitialized byte as a `u8` is UB, not "you get garbage". Compilers exploit this.

## `unsafe impl Send` / `Sync`

The only mechanism by which a data race can be introduced without a single raw pointer in sight. Write the argument in the SAFETY comment:

```rust
// SAFETY: the inner pointer is only dereferenced under `lock`, and the pointee
// is never observed from another thread outside that lock.
unsafe impl Send for Handle {}
```

Common wrong ones: a type holding a pointer into thread-local storage; a C library handle whose documentation says "not thread safe"; a struct with an `Rc` inside where the author only tested single-threaded use.

## Interior Mutability, Correctly

- `UnsafeCell<T>` is the only way to get a `*mut T` out of a `&T` legitimately. Every safe interior-mutability type is built on it.
- Building your own means proving the aliasing discipline yourself. Prefer composing `Cell`, `RefCell`, `Mutex`, or an atomic — they are the same performance and someone else already proved them.

## Transmute

- `mem::transmute` reinterprets bits with no checks; it is the sharpest tool in the language and almost never the right one.
- Alternatives that are usually what you meant: `as` casts for numbers, `f32::to_bits`/`from_bits`, `slice::from_raw_parts` for pointer-plus-length, `bytemuck`/`zerocopy` for plain-old-data conversions with compile-time validation.
- Transmuting to change a lifetime is UB the moment the referent dies (SKILL.md Traps). Transmuting between types whose layouts you did not check with `#[repr(C)]` is UB even if their sizes match, because Rust's default layout is unspecified.

## SAFETY Comments as a Discipline

```rust
// SAFETY: `idx < self.len` was checked on the line above, and `self.ptr` is
// valid for `self.len` elements for the lifetime of `&self`.
unsafe { &*self.ptr.add(idx) }
```

- Every `unsafe` block gets one, naming the invariant and where it is established. Enforce with `#![warn(clippy::undocumented_unsafe_blocks)]`.
- Every `pub unsafe fn` gets a `/// # Safety` section stating what the caller must guarantee — `clippy::missing_safety_doc` checks that it exists, not that it is true.
- A comment that says "this is fine" is worse than none: it signals review happened when it did not.

## Verification

1. `cargo +nightly miri test` on the tests that exercise the `unsafe` path. Miri catches aliasing violations, out-of-bounds access, use-after-free, and invalid values.
2. `-Zsanitizer=address` / `-Zsanitizer=thread` for code Miri cannot interpret, including anything crossing FFI.
3. Fuzz the safe wrapper, not the `unsafe` internals — the wrapper is where the invariant is supposed to be enforced.
4. Keep `unsafe` in the smallest possible module with a safe, total public API. Auditing 40 lines behind a checked interface is possible; auditing `unsafe` sprinkled through a crate is not.

## When Unsafe Is Actually Warranted

| Reason | Verdict |
|---|---|
| FFI boundary | Legitimate and unavoidable |
| Data structure the borrow checker cannot express (intrusive list, custom allocator) | Legitimate; encapsulate it |
| Measured hot path where a bounds check dominates | Rare — an iterator usually removes the check safely |
| A safe abstraction genuinely missing from std | Legitimate; check crates.io first, this is usually solved |
| "The borrow checker won't let me" | Not a reason: go up the ladder in SKILL.md |
| "It's faster" without a profile | Not a reason |
