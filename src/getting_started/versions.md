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

- **Engine arrays, maps and sets of soft references hold `SoftObjectRef<T>`,** with their paths; they held the loaded objects, as `UObjectRef<T>`.

- **Rust properties of every type:** the other integers, `Text`, `TWeakObjectPtr<T>`, `SoftClassRef<T>`, `ScriptInterface<dyn I>`, `UeMap<K, V>`, `UeSet<T>`, arrays of `String`, `Text`, soft and weak references; `meta(...)` and doc comment tooltips. See [Properties](../classes/properties.md).

- **Rust functions of every type:** a `#[ufunction]`, a Blueprint event and a `#[udelegate]` take and return what a property holds, with `Vec<T>`, `HashMap<K, V>` and `HashSet<T>` for containers; `&mut T` parameters are out parameters. `BlueprintNativeEvent`, `CallInEditor`, `category`, `meta(...)`, doc comment tooltips, static functions and function libraries; single-cast and sparse delegates (`#[udelegate(single)]`, `#[udelegate(sparse)]`). See [Functions](../classes/functions.md) and [Delegates](../engine_api/delegates.md).

- **`#[uclass]`, `#[component]`, `#[uclass_impl]` and `#[class_defaults]` refuse the arguments they do not take,** which they ignored: remove the ones the compiler points at, they did nothing.

- **Every engine struct is an `OwnedStruct<T>`.** Those with no `StaticStruct` in C++ (`FBox`, `FIntPoint`, `FTimespan`, `FPlatformUserId`, the engine's `NoExportTypes`) were taken and given as a `*const u8`; they are `OwnedStruct<T>` like the others:

  ```rust
  let span = KismetMathLibrary::from_minutes(1.5);
  let seconds = KismetMathLibrary::get_total_seconds(&span);
  ```

- **Engine structs read from the engine are copies Rust owns.** A struct a function returns or a property gives, `OwnedStruct<T>`, is a complete copy, arrays and strings included; dropping it frees them, and `clone()` copies them.

- **Engine `FText` is `Text`,** which keeps its localization ([Text and localization](../engine_api/text.md)). Functions take `impl Into<Text>`, so `set_text("Game over")` still works; what gave a `String` gives a `Text`: `to_string()` for the string. An `FText` parameter with a default value is a required one.

- **Engine soft class references are bound,** as `SoftClassRef<T>`: `KismetSystemLibrary::load_class_asset_blocking(&SoftClassRef::new("/Script/Engine.StaticMeshActor"))`.

- **Engine functions and properties that take an interface are bound.** They were left out; they take the object as a `UObjectRef` of the interface (see [Interfaces](../engine_api/object_refs.md#engine-interfaces)).

- **`bind_action` takes a closure,** or a method with `method(self, Self::look)`, and returns an `ActionBinding` (`unbind()` stops it); it took the name of a `#[ufunction]`. The by-name version is `bind_action_by_name`, for a function a Blueprint defines. `bind_action_instance` gives the action's value with its elapsed and triggered times. See [Input](../recipes/input.md).

  ```rust
  // before
  bind_action(&self.as_ref(), action, ETriggerEvent::Triggered, "Look")?;
  // now
  bind_action(&self.as_ref(), action, ETriggerEvent::Triggered, method(self, Self::look))?;
  ```

- **Engine latent functions are `async`.** `KismetSystemLibrary::delay`, `load_asset`, `move_component_to`, `GameplayStatics::load_stream_level` and the others no longer take an `FLatentActionInfo` and return a future of their outputs: add `.await`, in async code. See [Async code](../engine_api/async.md#the-engines-latent-functions).

- **Async code:** `task::spawn_local` runs a future on the game thread, cancelled with its owner; `task::next_frame`, `task::run_blocking`, `timer::sleep` and `timer::sleep_real` wait in it. `SoftObjectRef::load_async`, `loading::load_all` and `loading::load_primary_asset` load in the background; `http::get` and `http::post` send HTTP requests; every multicast delegate has `next()`; an `async #[ufunction]` is a latent Blueprint node. See [Async code](../engine_api/async.md).

- **Timers with closures:** `timer::set_timer` returns a `TimerHandle` to clear, pause and query it, and frees its closure with the timer, as `k2_set_timer_delegate` now does. See [Timers](../engine_api/timers.md).

- **Methods as callbacks:** `method(self, Self::x)` binds a method of the class to a delegate, a timer or an input action, without `#[ufunction]`. See [Delegates](../engine_api/delegates.md#binding-a-method).

- **Native engine events:** the application's lifecycle, worlds and levels, players logging in and out, and a path following component's finished move. See [Delegates](../engine_api/delegates.md#use-case-an-engine-event-that-is-not-a-delegate).

## Changing the engine version

The bindings are generated from the engine you build with. After installing another engine version, point the CLI at it (see [Installing the CLI](install.md)) and regenerate everything:

```bash
rusteal build --all
```
