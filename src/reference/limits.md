# Known limits

What Rusteal does not do yet, and what to do meanwhile.

## Platforms

| Not yet | Meanwhile |
|---|---|
| Android and iOS builds (being actively worked on) | Run and test the game in the editor on Linux, macOS or Windows, touch controls included; the `ar` template runs there without an AR session. |
| Console builds (being actively worked on: what needs no SDK is done) | Linux, macOS and Windows builds; `rusteal package --static` and the habits in [Getting ready for consoles](../shipping/consoles.md). |
| `rusteal package --static` on Windows and macOS (written, not yet validated) | `--static` on Linux; the default packaging elsewhere. |
| `rusteal package --server` (written, not yet validated: it needs an engine built from source) | A dedicated server in Play In Editor, or Unreal's packaging of the server target. |

## Classes

| Not yet | Meanwhile |
|---|---|
| Custom network serialization (`NetSerialize`, `FFastArraySerializer`) | A C++ struct with it, used as a Rust property's type. |
| Server-only or client-only Rust code left out at compile time | Check `KismetSystemLibrary::is_dedicated_server` at run time. |
| Implementing a C++-only interface other than `IGameplayTagAssetInterface` without a base of your own | Declare a base that implements it in `rusteal.toml` (see [Engine virtuals](../classes/virtuals.md#c-only-interfaces)). |
| Overriding a C++ virtual of a class Rusteal has no base for, without declaring one | Declare the base in `rusteal.toml`'s `[[codegen.shims]]`. |
| Bases declared in a plugin's `rusteal.toml` | Declare them in the project's `rusteal.toml`. |
| A virtual whose parameters a Blueprint event cannot take (pointers to non-objects, raw C++ enums), unless they are dropped | `drop` them in the base, or leave the virtual out. |

## Hot reload

| Not yet | Meanwhile |
|---|---|
| Reloading a changed `#[ustruct]` | Restart the editor. |
| Replacing a changed class or interface outside the editor (`-game`) | Restart the game. |
| Keeping Rust-only fields across a reload | Keep what must survive in `#[uproperty]` fields. |
| Freeing the previous library on macOS (macOS never unloads a Rust library: each reload keeps the previous one in memory) | Restart the editor after a long session of reloads. |

## Delegates

| Not yet | Meanwhile |
|---|---|
| Binding C++-only engine delegates other than the [native events](../engine_api/delegates.md#use-case-an-engine-event-that-is-not-a-delegate) | A Blueprint-exposed delegate raised for the same event (`AIController::ReceiveMoveCompleted`). |

## Async code

| Not yet | Meanwhile |
|---|---|
| An `async #[ufunction]` returning a value or with out parameters | Set properties of the class before it returns; the Blueprint reads them after the node. |
| An `async #[ufunction]` taking `&self` or `&mut self` | Take `self` by value (see [What a task holds](../engine_api/async.md#what-a-task-holds)). |
| Engine latent functions that take or give a container (`KismetSystemLibrary::load_assets`) | `loading::load_all`. |
| Checking that a `ThreadSafe` function calls only thread-safe engine functions | Call none but the math libraries' from it, as in C++. |

## Plugins

| Not yet | Meanwhile |
|---|---|
| An editor library using its runtime library's Rust types | Their Unreal functions: dynamic calls, interfaces, Blueprint classes. |
| The game using a plugin's Rust-only fields, structs, enums and delegates | Its classes' properties and functions, which the game's bindings have (see [A plugin's classes in the game](../plugins/game_uses_plugin.md)); dynamic calls for the rest. |
| A plugin using another plugin's Rust classes | Dynamic calls, interfaces, subsystems found by class. |
| Slate lists, trees, combo boxes, images, styles and windows outside tabs | Editor Utility Widgets in Blueprint calling Rust functions. |
| Property type customizations, custom asset editors, editor modes, Scriptable Tools, Interchange pipelines, Blueprint nodes | Editor Utility Widgets and Blueprint nodes calling Rust functions. |
| Asset and actor actions and Editor Utility Widgets with a Rust parent (written, not tested) | An `EditorUtilityObject` with a Rust parent, run from a menu entry. |

## Gameplay frameworks

| Not yet | Meanwhile |
|---|---|
| The Gameplay Ability System over the network (prediction, replicated attributes), gameplay cues and execution calculations in Rust | Abilities, effects and attribute sets in single player (see [The Gameplay Ability System](../systems/abilities.md)). |
| CommonUI, MVVM view models, Game Features, Smart Objects, PCG, MetaSounds and Online Services bindings | Use them from Blueprints; their modules can be listed in `rusteal.toml`'s `[codegen.modules]`, which has not been tried with them yet. |

## Bindings

| Not yet | Meanwhile |
|---|---|
| Functions the engine reflects but does not export | They are in the blocklist of `rusteal.toml`; use the Blueprint-callable alternative or a dynamic call. |
| Modules the project does not list | Add them to `rusteal.toml` and run `rusteal build --all`. |

## Platforms

| Platform | Status |
|---|---|
| Linux (x64) | Supported |
| Windows (x64) | Supported |
| macOS (Apple silicon) | Supported |

macOS on Intel processors has not been tested.

| Not yet | Meanwhile |
|---|---|
| Packaging a Mac game from the editor's **Platforms** menu or with `RunUAT` on Xcode 27 (Unreal Engine 5.8 fails to finalize the `.app` with Xcode, for any C++ project) | Package with `rusteal package`, which works around it. |
