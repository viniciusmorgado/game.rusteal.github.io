# Development and release builds

The game library is built with Cargo's `dev` profile by default, and with the `release` profile when `--release` is given. The two want opposite things, and the projects Rusteal creates set them up for that in `Rust/Cargo.toml`, the only place Cargo reads profiles from (in the game crate's own `Cargo.toml` they are ignored with a warning).

## dev: iterating

```bash
rusteal build
```

Rebuilding after a change must be fast, and the game must still run well in the editor:

- your game crate, the one that changes, is barely optimized, so it compiles in about a second;
- the Rusteal crates and the generated bindings, which compile once, are fully optimized;
- your crate has full debug info, for the debugger; the dependencies and the bindings keep only file and line, enough for backtraces, so the library stays small enough to reload quickly.

## release: shipping

```bash
rusteal build --release
```

The fastest library possible, however long it takes to build: one codegen unit per crate and link-time optimization across crates, which also drops the parts of the bindings the game does not use. `rusteal package` always builds this profile; with `--static`, the Rust code is built once more for the game, with link-time optimization across every library at once (see [Linking the Rust code into the game](../shipping/packaging.md#linking-the-rust-code-into-the-game)).

## Tuning

The profiles are yours to change, in `Rust/Cargo.toml`. The optimization levels follow Bevy's recommendations for game projects:

```toml
[profile.dev]
opt-level = 1

[profile.dev.package."*"]
opt-level = 3
debug = "line-tables-only"

[profile.dev.package.my-game-bindings]
opt-level = 3
debug = "line-tables-only"

[profile.release]
codegen-units = 1
lto = "thin"
```

Every crates.io dependency you add is covered by `"*"`: optimized, with file and line only. To step into one of them in the debugger, give it full debug info alone:

```toml
[profile.dev.package.rapier3d]
debug = true
```

Keep the bindings at `line-tables-only`: with full debug info, the library grows past a hundred megabytes, and every hot reload loads it again. On macOS the debug info stays in the object files next to the library (Rust's default there), so the library itself stays small; the reload still has to copy it, and each reload keeps the previous library in memory until the editor closes.
