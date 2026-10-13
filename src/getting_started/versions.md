# Versions and upgrades

A project is tied to one Rusteal version: `Rust/Cargo.toml` pins `rusteal-runtime`, `rusteal-core` and `rusteal-ffi` to it (`"=x.y.z"`), and the plugins in `Plugins/` carry it.

Before doing anything, `rusteal build`, `rusteal generate` and `rusteal setup` compare it with the CLI's own version:

- **The CLI is newer.** Upgrade the project:

  ```bash
  rusteal upgrade
  ```

  It moves the pins, replaces `Plugins/Rusteal` and `Plugins/RustealGenerator` with the CLI's, and runs a full build.

- **The project is newer.** Install the CLI version the project uses:

  ```bash
  cargo install rusteal@x.y.z
  ```

The plugin checks the library as well: a library built against another Rusteal version is refused when it loads, and the Output Log shows both versions. So is a library whose bindings come from another code generation than the plugin's C++ wrappers (after changing `rusteal.toml` or the engine without `rusteal build --all`): the Output Log says so, instead of the game calling the wrong engine functions.

A build also brings an older project's Rust workspace up to date when the layout changed: since 0.9, the bindings crate is a package named after the game crate (`my-game-bindings`, imported as `bindings`): `rusteal build` rewrites the dependency and the profile in `Rust/` and prints the files it changed. A game crate no longer declares a `crate-type`: `rusteal build` makes the dynamic library itself (`cargo rustc --crate-type cdylib`), and `rusteal package --static` rewrites an older crate's `crate-type` to `["rlib"]`.

## Before upgrading

Rusteal is in early development, and its API changes between versions. Commit your project before running `rusteal upgrade`, so the changes it makes, and the compile errors your code may get from a changed API, can be reviewed in a diff.

## Changing the engine version

The bindings are generated from the engine you build with. After installing another engine version, point the CLI at it (see [Installing the CLI](install.md)) and regenerate everything:

```bash
rusteal build --all
```
