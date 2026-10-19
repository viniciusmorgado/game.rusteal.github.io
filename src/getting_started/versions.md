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

When the versions it crosses change an API your code may use, `rusteal upgrade` lists what changed before it builds, and the compiler points at each line to change.

## Changes by version

### 0.10

- **Engine soft and weak references have their own types.** An engine `TSoftObjectPtr<T>` is a `SoftObjectRef<T>` and an engine `TWeakObjectPtr<T>` a `TWeakObjectPtr<T>`, in properties, struct fields, function parameters and returns; they were `UObjectRef<T>`. A soft reference keeps its path when the asset is not loaded, so a function such as `KismetSystemLibrary::load_asset_blocking` takes a path. See [Object references](../engine_api/object_refs.md#engine-soft-and-weak-references).

  ```rust
  // before
  let component = hit.get_component();
  // now: a TWeakObjectPtr, checked() resolves it as it does for UObjectRef
  let component = hit.get_component().checked()?;
  ```

- **Engine functions that change a struct passed by reference take `&mut OwnedStruct<T>`;** they took a `*mut u8`:

  ```rust
  let mut guid = KismetGuidLibrary::new_guid();
  KismetGuidLibrary::invalidate_guid(&mut guid);
  ```

- **Every engine struct is an `OwnedStruct<T>`.** Those with no `StaticStruct` in C++ (`FBox`, `FIntPoint`, `FTimespan`, `FPlatformUserId`, the engine's `NoExportTypes`) were taken and given as a `*const u8`; they are `OwnedStruct<T>` like the others:

  ```rust
  let span = KismetMathLibrary::from_minutes(1.5);
  let seconds = KismetMathLibrary::get_total_seconds(&span);
  ```

- **Engine structs read from the engine are copies Rust owns.** A struct a function returns or a property gives, `OwnedStruct<T>`, is a complete copy, arrays and strings included; dropping it frees them, and `clone()` copies them.

- **Engine `FText` is `Text`,** which keeps its localization ([Text and localization](../engine_api/text.md)). Functions take `impl Into<Text>`, so `set_text("Game over")` still works; what gave a `String` gives a `Text`: `to_string()` for the string. An `FText` parameter with a default value is a required one.

- **Engine soft class references are bound,** as `SoftClassRef<T>`: `KismetSystemLibrary::load_class_asset_blocking(&SoftClassRef::new("/Script/Engine.StaticMeshActor"))`.

- **Engine functions and properties that take an interface are bound.** They were left out; they take the object as a `UObjectRef` of the interface (see [Interfaces](../engine_api/object_refs.md#engine-interfaces)).

## Changing the engine version

The bindings are generated from the engine you build with. After installing another engine version, point the CLI at it (see [Installing the CLI](install.md)) and regenerate everything:

```bash
rusteal build --all
```
