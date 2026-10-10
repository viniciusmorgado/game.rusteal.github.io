# Known limits

What Rusteal does not do yet, and what to do meanwhile.

## Platforms

| Not yet | Meanwhile |
|---|---|
| Android and iOS builds (being actively worked on) | Run and test the game in the editor on Linux or Windows, touch controls included; the `ar` template runs there without an AR session. |
| Console builds (being actively worked on) | Linux and Windows builds. |

## Classes

| Not yet | Meanwhile |
|---|---|
| `TMap` and `TSet` properties | Two `UeArray`s side by side, or a Rust-only `HashMap`. |
| `String`, arrays and `FName` as `#[ufunction]` parameters | A property the caller sets before the call. |
| Network replication (`Replicated`, RPCs) | Single player and local multiplayer. |
| Implementing a C++-only interface | Interfaces whose functions Blueprints can implement. |

## Hot reload

| Not yet | Meanwhile |
|---|---|
| Reloading a changed `#[ustruct]` | Restart the editor. |
| Replacing a changed class or interface outside the editor (`-game`) | Restart the game. |
| Keeping Rust-only fields across a reload | Keep what must survive in `#[uproperty]` fields. |

## Delegates

| Not yet | Meanwhile |
|---|---|
| Binding C++-only engine delegates (`UPathFollowingComponent::OnRequestFinished`) | A Blueprint-exposed delegate raised for the same event (`AIController::ReceiveMoveCompleted`). |

## Plugins

| Not yet | Meanwhile |
|---|---|
| A runtime and an editor library in one plugin | Two plugins, `Name` and `NameEditor`. |
| The game using a plugin's Rust types | Dynamic calls, interfaces, subsystems found by class. |
| Slate UI, details customizations, asset editors, importers, Blueprint nodes | Editor Utility Widgets in Blueprint calling Rust functions. |

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
| macOS | Not tested yet |
