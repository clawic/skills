# Cross-Compilation — Targets, Linkers, and Binaries That Run Elsewhere

Rust compiles for another target trivially; **linking** for another target is the hard part, because that needs a linker and system libraries for the destination, not for your machine.

Contents: The Three Things a Cross Build Needs · Reading a Target Triple · Tools, in Order of Effort · glibc Versions · musl and Static Linking · The `-sys` Crate Problem · Build Scripts and Proc Macros · Containers · macOS and Windows Specifics · Verify Before Shipping.

## The Three Things a Cross Build Needs

1. The Rust standard library for the target: `rustup target add aarch64-unknown-linux-gnu`.
2. A linker that emits that target's object format, configured in `.cargo/config.toml`.
3. The target's C libraries, if any dependency links to C (openssl, sqlite, libz, and anything with a `-sys` crate).

Item 3 is where cross builds fail. Pure-Rust dependency trees cross-compile with almost no setup; one `-sys` crate changes the problem from a flag to a sysroot.

```toml
# .cargo/config.toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
```

## Reading a Target Triple

`<arch>-<vendor>-<os>-<abi>`: `x86_64-unknown-linux-gnu`, `aarch64-apple-darwin`, `x86_64-pc-windows-msvc`, `armv7-unknown-linux-gnueabihf`, `thumbv7em-none-eabihf`.

- The last field is the one that bites: `gnu` links glibc dynamically, `musl` links musl (usually statically), `msvc` and `gnu` on Windows are different ABIs with different toolchains, `eabihf` means hardware floating point.
- `rustc --print target-list` for every supported triple; `rustc --print cfg --target <triple>` for what `cfg!` will see.
- Tier 1 targets are tested by the project and have prebuilt std; tier 3 may not even have a prebuilt std (`-Zbuild-std` on nightly).

## Tools, in Order of Effort

| Tool | What it does | When |
|---|---|---|
| `cross` | Runs the build in a container that already has the toolchain and sysroot | First choice for Linux targets from any host |
| `cargo-zigbuild` | Uses `zig cc` as a universal cross linker, including a selectable glibc version | Great when you need an old glibc floor without an old distro |
| Docker with the target as the base image | Build inside the environment you ship to | CI where you already have containers |
| Native runner per architecture | No cross-compilation at all | The fastest and least surprising option when CI offers arm64 runners |
| Manual toolchain plus `.cargo/config.toml` | Full control | Embedded, unusual targets |
| QEMU emulation of a whole build | Last resort | Typically several times slower than cross-compiling; use only to *test* the artifact |

## glibc Versions (the most common production failure)

Building on a newer distro and running on an older one gives `version 'GLIBC_2.34' not found` at startup. glibc is backward compatible, never forward compatible: a binary linked against a newer glibc will not run on an older system.

Fixes, in order of preference:

1. Build against the oldest glibc you must support — an old container image, or `cargo zigbuild --target x86_64-unknown-linux-gnu.2.17`.
2. Target musl and ship a static binary: no libc dependency at all.
3. Ship a container image so the libc travels with the binary.

## musl and Static Linking

- `x86_64-unknown-linux-musl` produces a fully static binary that runs on any Linux, including `scratch` and distroless images.
- Costs to know before choosing it: musl's default allocator is markedly slower than glibc's under multi-threaded allocation-heavy load — link `mimalloc` or `jemalloc` if a benchmark shows it. Some `-sys` crates need a musl-aware sysroot. `dlopen` and NSS-based name resolution do not work in a static binary, which breaks anything relying on `/etc/nsswitch.conf`.
- Verify what you produced: `ldd target/.../app` should say "not a dynamic executable"; `file` should say "statically linked".

## The `-sys` Crate Problem

| Crate | Symptom when cross-compiling | Fix |
|---|---|---|
| `openssl-sys` | "failed to run custom build command", pkg-config cannot find OpenSSL for the target | `rustls` instead (pure Rust), or the `vendored` feature to build OpenSSL from source |
| `libsqlite3-sys` | Same shape | `bundled` feature |
| `ring`, `aws-lc-rs` | Needs a C compiler for the target | `cross`, or the container approach |
| Anything using `cc` | Uses `CC_<triple>` and `AR_<triple>` env vars | Set them, or let `cross` do it |

Prefer pure-Rust alternatives at dependency-selection time: `rustls` over `openssl`, `rusqlite` with `bundled`, `time` or `jiff` over C date libraries. That one decision removes most cross-compilation work permanently.

## Build Scripts and Proc Macros

- They compile for the **host**, always, even in a cross build. `RUSTFLAGS` intended for the target does not apply to them; `CARGO_TARGET_<TRIPLE>_RUSTFLAGS` is the targeted form.
- Inside `build.rs`, `TARGET` and `HOST` env vars differ during a cross build. Code branching on `cfg!(target_os)` in a build script reads the **host**, which is a silent, frequent bug — read the `TARGET` variable instead.
- `#[cfg(target_os = ...)]` in normal crate code is evaluated for the target and behaves as expected.

## Containers

```dockerfile
FROM rust:1-bookworm AS build
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs && cargo build --release --locked
COPY . .
RUN touch src/main.rs && cargo build --release --locked

FROM gcr.io/distroless/cc-debian12
COPY --from=build /app/target/release/app /app
ENTRYPOINT ["/app"]
```

- The dummy-`main` trick caches the dependency layer so a source edit does not rebuild the world; `cargo-chef` does the same thing more robustly for workspaces.
- Mount the cargo registry and target dir as BuildKit caches (`--mount=type=cache`) for a much larger win than layer ordering alone.
- Runtime base: `distroless/cc` for a glibc binary, `scratch` for a musl static binary. `alpine` runs musl binaries and is convenient when you want a shell.
- Building on an arm64 laptop for an amd64 server without `--platform` produces `exec format error` at deploy time — the same class of mistake as any other container architecture mismatch.

## macOS and Windows Specifics

- Cross-compiling **to** Apple targets requires the Apple SDK and is licence-restricted off Apple hardware; a macOS runner is the practical answer. Universal binaries: build both arches, then `lipo -create -output app app-x86_64 app-aarch64`.
- Apple ships codesigning requirements: an unsigned or unnotarized binary is blocked on Gatekeeper-enabled machines regardless of how it was built.
- Windows `-msvc` needs the MSVC toolchain (build on Windows, or `cargo-xwin`); `-gnu` cross-compiles from Linux with mingw-w64 but produces a binary linked against a different runtime — choose one for the whole project.

## Verify Before Shipping

| Check | Command |
|---|---|
| Right architecture | `file target/<triple>/release/app` |
| Dynamic dependencies | `ldd` (Linux), `otool -L` (macOS) |
| glibc symbol floor | `objdump -T app \| grep GLIBC_ \| sort -u` |
| It actually starts | Run it in the target environment (container, device, or emulator) as a CI step |
| Size | `cargo bloat`, and strip in a release profile |
