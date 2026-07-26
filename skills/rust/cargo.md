# Cargo — Features, Workspaces, Lockfiles, and Builds That Reproduce

Cargo is a resolver plus a build cache. Most "cargo bugs" are the resolver doing exactly what the manifests asked, in a graph nobody has read end to end.

Contents: Features · Version Requirements and the Lockfile · Workspaces · Profiles · MSRV · Build Scripts · Toolchain and Registry · Maintenance Commands · CI-Only Failures.

## Features

- Features are **additive and unified across the whole graph** (SKILL.md rule 8). There is no way to turn off a feature another crate turned on; designing a feature whose presence *removes* behavior guarantees a broken build for someone.
- `default-features = false` applies only to your own dependency edge. If any other crate depends on the same package with defaults on, defaults win for everyone.
- Diagnosis, always the same command: `cargo tree -e features -i <crate>` prints who enables what, inverted onto that crate.
- Resolver v2 (default in edition 2021 and later) stops unifying features across build-dependencies, dev-dependencies, and target-specific dependencies with normal ones. A workspace still on `resolver = "1"` pulls dev-only features into the release build; set `resolver = "2"` at the workspace root.
- Optional dependencies create an implicit feature of the same name. `dep:serde` syntax lets you have a feature named `serde` that enables the dependency without exposing it as a separate switch.
- Test what you ship: `cargo hack --feature-powerset --depth 2 check` catches the combinations CI never builds. `--all-features` alone hides features that are mutually exclusive.

## Version Requirements and the Lockfile

- `serde = "1.0"` means `>=1.0.0, <2.0.0`. `"0.3"` means `>=0.3.0, <0.4.0` — for 0.x, the minor position acts as the major.
- `=1.2.3` pins exactly and blocks security patches; use it only to work around a known-broken release, with a comment and a date.
- Commit `Cargo.lock` in every project. For a library it does not affect consumers, and it makes your own CI reproducible.
- `cargo update` rewrites the lockfile within the existing requirements; changing a requirement is a manifest edit. `cargo update -p serde --precise 1.0.200` moves exactly one.
- `--locked` in CI fails instead of silently resolving newer versions (SKILL.md rule 9). `--frozen` additionally forbids network access.
- `[patch.crates-io]` redirects a dependency graph-wide — the correct tool for testing a fork. `[replace]` is deprecated; a `path` dependency only affects the crate that declares it.
- Duplicate versions: `cargo tree -d`. Two semver-incompatible copies of one crate compile fine and produce "expected `X`, found `X`" at the boundary between them.

## Workspaces

```toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
```

- `[workspace.dependencies]` plus `serde.workspace = true` in members is the only maintainable way to keep versions aligned; without it, proc-macro duplication and diamond conflicts arrive as you add crates.
- One `target/` directory and one `Cargo.lock` for the whole workspace. Features unify across all members being built together, so `cargo test --workspace` can enable features a single-crate build never sees — a real source of "passes alone, fails in the workspace".
- `default-members` keeps a bare `cargo build` from compiling every member.
- Splitting a big crate into several speeds up incremental builds, because rustc parallelizes poorly inside one crate and recompiles it whole.

## Profiles

| Key | dev default | release default | Notes |
|---|---|---|---|
| `opt-level` | 0 | 3 | `"z"`/`"s"` for size |
| `debug` | true | false | `debug = true` in release for profiling |
| `debug-assertions` | true | false | Controls `debug_assert!` |
| `overflow-checks` | true | false | SKILL.md rule 3 |
| `lto` | false | false | `"thin"` is the usual first step |
| `codegen-units` | 256 | 16 | 1 for maximum optimization, slowest build |
| `incremental` | true | false | Leave off in CI; it only helps repeated local builds |

- Custom profiles inherit: `[profile.ci] inherits = "release"` then override what CI needs.
- `[profile.dev.package."*"] opt-level = 3` optimizes dependencies while keeping your crate fast to rebuild.
- `[profile.release.package.foo]` tunes a single dependency, which is how you keep one hot crate optimized in a debug build.

## MSRV

- `rust-version = "1.75"` in `Cargo.toml` makes cargo refuse to build on older toolchains with a clear message rather than a syntax error.
- Cargo's MSRV-aware resolver (`resolver = "3"`, available from `cargo >=1.84`) prefers dependency versions compatible with your floor instead of the newest.
- Verify it, do not assert it: a CI job on the pinned old toolchain, and `cargo minimal-versions check` to catch requirements that are lower than what you actually use.
- Raising the MSRV is a semver-minor change by convention, and a breaking one for users who pinned. Announce it in the changelog.

## Build Scripts

- `build.rs` runs before the crate compiles, on the **host**, every time its inputs change. Emit `cargo::rerun-if-changed=path` for each input or it reruns on every build and destroys incremental compilation.
- Communicate via `cargo::rustc-cfg=`, `cargo::rustc-env=`, `cargo::rustc-link-lib=`. Writing generated code into `OUT_DIR` and `include!`ing it is the standard pattern; writing into `src/` breaks reproducible builds and confuses everyone.
- Build scripts and proc macros are compiled for the host even when cross-compiling, so `RUSTFLAGS` for the target does not reach them; `CARGO_TARGET_<TRIPLE>_RUSTFLAGS` is the targeted form.
- Every build script is arbitrary code execution at build time — the main supply-chain surface. `cargo vet`/`cargo crev` for review, `cargo deny` for policy.

## Toolchain and Registry

- `rust-toolchain.toml` pins channel, components, and targets, and rustup honors it automatically. Without it, "works on my machine" includes the compiler version.
- `cargo install --locked <tool>` for dev tools; without `--locked` the tool builds against whatever resolved today and frequently fails.
- The sparse registry protocol (default since `cargo >=1.70`) removed the full-index clone; a slow first build on an old CI image is usually still using git protocol.
- Vendoring: `cargo vendor` plus a `.cargo/config.toml` source replacement for air-gapped or audited builds.

## Maintenance Commands

| Question | Command |
|---|---|
| Who pulls in this crate? | `cargo tree -i <crate>` |
| Who enables this feature? | `cargo tree -e features -i <crate>` |
| Duplicate versions? | `cargo tree -d` |
| Unused dependencies? | `cargo machete` or `cargo udeps` (nightly) |
| Known vulnerabilities? | `cargo audit` |
| License and source policy? | `cargo deny check` |
| Newer versions available? | `cargo outdated` |
| Did I break semver? | `cargo semver-checks` |
| Where is build time going? | `cargo build --timings` |
| Anything else | `cargo <cmd> --help`; most answers are a flag on a command you already run |

## CI-Only Failures

| Symptom | Cause |
|---|---|
| Compiles locally, type errors in CI | Different toolchain, or resolution without `--locked` |
| `signal: 9, SIGKILL` during codegen | rustc OOM: reduce `-j`, raise `codegen-units`, drop `lto = "fat"` for CI |
| Passes alone, fails with `--workspace` | Feature unification across members |
| Passes with `--all-features`, fails without | A test depends on an optional dependency; gate it with `#[cfg(feature = ...)]` |
| Fails only on the first run of the day | Registry or network flake; cache the registry and use `--locked` |
| Slow every time | No cache for `~/.cargo/registry` and `target/`, or `cargo clean` in the script |
