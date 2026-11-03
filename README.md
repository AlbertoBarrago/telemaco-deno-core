# telemaco-deno-core

Fork of [deno_core](https://github.com/denoland/deno_core) 0.350.0, maintained
for [Telemaco](https://github.com/AlbertoBarrago/telemaco). The library name is
still `deno_core`, so a dependent only renames the package:

```toml
deno_core = { version = "=0.350.0", package = "telemaco-deno-core" }
```

## What differs from upstream

Based on the crates.io `deno_core` 0.350.0 release, archive SHA-256
`e273b731fce500130790e777cb2631dc451db412975304d23816c1e444a10c5b`.
Every upstream file is byte-identical except `lib.rs`, `runtime/mod.rs`,
`runtime/jsruntime.rs` and `modules/map.rs`.

`JsRuntime::attach_module_realm` attaches a separate `ModuleMap` and loader to
an embedder-created context, returned as a `ModuleRealm` (`load`, `resolve`,
`evaluate`). It reuses deno_core module loading, evaluation, `import.meta` and
dynamic imports, which is how Telemaco gives each iframe its own module graph.

- The shared op driver is advanced once per event-loop turn; child module maps
  are polled in their own contexts.
- Dropping a realm cancels its graph and releases its V8 handles. If the
  runtime is already gone, the drop is a no-op instead of touching a disposed
  isolate.
- The context owns a reference to its map, so late callbacks stay safe until V8
  collects the context.
- Retired maps keep only weak module metadata, so `import.meta` in a function
  retained by another context still resolves without rooting the removed frame.
  Entries for collected modules are pruned on every retire.

## Upgrading

Before moving to a newer deno_core, port this API onto the new release and rerun
Telemaco's iframe isolation, module lifecycle, cancellation and runtime teardown
tests.

## License

MIT, same as upstream. See `LICENSE.md`.

---

Upstream README follows.

# Deno Core Crate

[![crates](https://img.shields.io/crates/v/deno_core.svg)](https://crates.io/crates/deno_core)
[![docs](https://docs.rs/deno_core/badge.svg)](https://docs.rs/deno_core)

The main dependency of this crate is
[rusty_v8](https://github.com/denoland/rusty_v8), which provides the V8-Rust
bindings.

This Rust crate contains the essential V8 bindings for Deno's command-line
interface (Deno CLI). The main abstraction here is the JsRuntime which provides
a way to execute JavaScript.

The JsRuntime implements an event loop abstraction for the executed code that
keeps track of all pending tasks (async ops, dynamic module loads). It is user's
responsibility to drive that loop by using `JsRuntime::run_event_loop` method -
it must be executed in the context of Rust's future executor (eg. tokio, smol).

Rust functions can be registered in JavaScript using the `deno_core::extension!`
macro. A conventional way to write ops is using the
[`deno_ops`](../ops/README.md) crate.

Documentation for this crate is thin at the moment. Please see
[hello_world.rs](./examples/hello_world.rs) and
[eval_js_value.rs](./examples/eval_js_value.rs) as examples of usage.

TypeScript support and lots of other functionality are not available at this
layer. See the [CLI](https://github.com/denoland/deno/tree/main/cli) for that.
