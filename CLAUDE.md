# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

`telemaco-deno-core` is a fork of the crates.io `deno_core` 0.350.0 release,
published under a new package name for the Telemaco headless browser
(`~/Projects/telemaco`, crate `telemaco-js`). The library name stays
`deno_core`, so dependents rename only the package:

```toml
deno_core = { version = "=0.350.0", package = "telemaco-deno-core" }
```

`README.md` (top section) is the authoritative description of what differs
from upstream. Keep it in sync with every change to the patch.

## Scope of the fork

Only four files differ from upstream: `lib.rs`, `runtime/mod.rs`,
`runtime/jsruntime.rs`, `modules/map.rs`. Every other file must stay
byte-identical to the 0.350.0 archive (SHA-256
`e273b731fce500130790e777cb2631dc451db412975304d23816c1e444a10c5b`, present in
`~/.cargo/registry/cache/index.crates.io-*/deno_core-0.350.0.crate`). Do not
reformat, refactor or "fix" upstream code: a small, isolated diff is what keeps
the fork portable to newer deno_core releases.

Check the diff against upstream with:

```bash
U=$(ls -d ~/.cargo/registry/src/index.crates.io-*/deno_core-0.350.0)
diff -ru "$U" . -x target -x .jj -x .git -x Cargo.lock -x README.md -x CLAUDE.md -x LICENSE.md -x .gitignore -x Cargo.toml \
  -x Cargo.toml.orig -x .cargo_vcs_info.json -x .cargo-ok
```

Expected output: exactly the four patched files differ.

`Cargo.toml` is the normalized manifest from the published crate (no
workspace inheritance); only `name`, `description` and `repository` differ.

## The ModuleRealm patch

`JsRuntime::attach_module_realm(context, loader)` gives an embedder-created V8
context (a Telemaco iframe) its own `ModuleMap` and loader, returned as a
`ModuleRealm` (`load`, `resolve`, `evaluate`). The pieces that span files:

- **Ownership**: the context holds the map through a typed slot plus the
  `MODULE_MAP_SLOT_INDEX` / `CONTEXT_STATE_SLOT_INDEX` embedder pointers, so
  late V8 callbacks stay valid until context collection. The runtime keeps only
  `Weak` references to realms in `JsRuntimeState::module_realms`.
- **Event loop**: `poll_event_loop` polls each live child map in its own
  `ContextScope`, while the shared op driver advances once. Pending child
  imports/evaluations make the loop return `Poll::Pending` without the
  main-realm stalled-import check.
- **Teardown**: `ModuleRealmInner::drop` calls `ModuleMap::retire`, which keeps
  weak `(module, name, type)` metadata so `import.meta` in functions retained by
  another context still resolves, then `destroy()` clears handles and pending
  state. Dead weak entries are pruned on each retire. The drop is a no-op once
  the runtime state is gone, because the isolate is disposed with it.

## Build and verify

There is no workspace here; the meaningful checks run from Telemaco, which
already has the V8 build cached in its `target/`:

```bash
cd ~/Projects/telemaco
PATCH='patch.crates-io.telemaco-deno-core.path="/Users/albz/Projects/telemaco-deno-core"'
CARGO_INCREMENTAL=0 CARGO_BUILD_JOBS=2 cargo build --release -p telemaco-cli --bins --features render --config "$PATCH"
cargo nextest run --release --features render -p telemaco-js --config "$PATCH"
TELEMACO_BIN=./target/release/telemaco python3 acceptance/run.py
```

Use `cargo nextest`, not `cargo test` (single V8 isolate per process). The
`--config` patch must not leave a path source in Telemaco's committed
`Cargo.lock`: rebuild without it once the version is published.

Before publishing: `cargo publish --dry-run` here. A published version cannot be
deleted, only yanked.

## Upgrading deno_core

Port the four-file patch onto the new release (starting from its crates.io
archive, not the git repo), bump `version` to match upstream, update the SHA in
the README, then rerun Telemaco's iframe isolation, module lifecycle,
cancellation and runtime teardown tests.

## VCS

`jj` (colocated git), same as Telemaco: work directly on the working copy, no
per-task branches.
