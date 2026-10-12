# Project layout

A Rusteal project is an Unreal project with a Cargo workspace inside it. Everything it needs lives in its directory; the only thing outside is the engine path in the CLI's configuration.

```text
MyGame/
├── MyGame.uproject
├── rusteal.toml                # Rusteal's project configuration
├── Source/MyGame/              # the C++ module the Unreal Build Tool needs
├── Content/                    # the project's assets
├── Config/                     # the project's ini files
├── Plugins/Rusteal/            # the runtime plugin and the generated C++ wrappers
├── Plugins/RustealGenerator/   # the plugin that exports the engine's reflection
├── Intermediate/Rusteal/uht/   # the exported reflection (build output)
└── Rust/
    ├── Cargo.toml              # the workspace
    ├── my-game/                # your game crate
    └── bindings/               # the generated bindings crate
```

## What to version

Version everything except the build output, which the `.gitignore` written by `rusteal new` already leaves out. That includes `Rust/bindings/` and the generated C++ wrappers in `Plugins/Rusteal/`: they are generated, but a clone of the project builds without generating them again.

## The workspace

`Rust/Cargo.toml` lists the crates and pins the Rusteal crates to the version that created the project:

```toml
[workspace]
members = ["my-game", "bindings"]
resolver = "3"

[workspace.dependencies]
rusteal-runtime = "=x.y.z"
rusteal-core = "=x.y.z"
rusteal-ffi = "=x.y.z"
glam = "0.34"
log = "0.4"
```

It also holds the build profiles, which only a workspace's `Cargo.toml` can set. See [Development and release builds](../cli/profiles.md).

## The game crate

`Rust/my-game/Cargo.toml` makes the crate a library, dynamic for the editor and a Rust library for [static packaging](../shipping/packaging.md#linking-the-rust-code-into-the-game), and depends on the runtime and the bindings:

```toml
[package]
name = "my-game"
version = "0.1.0"
edition = "2024"
publish = false

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
rusteal-runtime = { workspace = true }
bindings = { package = "my-game-bindings", path = "../bindings" }
glam = { workspace = true }
log = { workspace = true }
```

Add any crate from crates.io to `[dependencies]` as in any Rust project: `serde`, `rand`, `rayon`, a physics or a pathfinding library. They are compiled into your game's library.

Your code goes in `Rust/my-game/src/`. `lib.rs` starts with `#![deny(unsafe_code)]`, calls `rusteal_runtime::entry!()` once and declares the modules:

```rust
#![deny(unsafe_code)]

rusteal_runtime::entry!();

mod hello;
```

**A game written with Rusteal never needs `unsafe`.** Everything a game does, Rusteal's API does in safe Rust, and the code its macros generate is accepted by `#![deny(unsafe_code)]`; the line makes the compiler refuse an `unsafe` that slips in, so keep it. If something seems to need `unsafe`, look for the safe call in this handbook; if there is none, it is a gap in Rusteal to report, not something to work around.

How you split the rest into modules is up to you. The templates keep each class in its own file, and the rules that do not touch the engine in a `model.rs` next to it, where `cargo test` can test them (see [Testing without the engine](../iteration/testing.md)).

## The bindings crate

`Rust/bindings/` is generated from the engine you build with, and only for the engine modules `rusteal.toml` lists. Do not edit it: the next generation overwrites it. Its package is named after your crate (`my-game-bindings`), so that the game and each plugin have their own, and your crate imports it as `bindings`. `entry!()` connects the two when the library starts: it uses the crate named `bindings`, and `entry!(other_name)` names another one. To bind more of the engine (AI, Niagara, StateTree, Gameplay Tags...), see [Binding more of the engine](../cli/rusteal_toml.md).

## Where the library goes

`rusteal build` deploys the library to `Plugins/Rusteal/Binaries/<Platform>/`: `librusteal.so` on Linux, `librusteal.dylib` on macOS, `rusteal.dll` on Windows. The plugin loads it from there when the editor or the game starts.
