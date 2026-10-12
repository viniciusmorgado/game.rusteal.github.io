# Known limits

What Rusteal does not do yet, and what to do meanwhile.

## Platforms

| Not yet | Meanwhile |
|---|---|
| Android and iOS builds (being actively worked on) | Run and test the game in the editor on Linux, macOS or Windows, touch controls included; the `ar` template runs there without an AR session. |
| Console builds (being actively worked on) | Linux, macOS and Windows builds. |

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
| Freeing the previous library on macOS (macOS never unloads a Rust library: each reload keeps the previous one in memory) | Restart the editor after a long session of reloads. |

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
| macOS (Apple silicon) | Supported |

macOS on Intel processors has not been tested. Packaging on macOS has not been verified: on the test machine (macOS 27, Xcode 27), Unreal Engine 5.8 fails to finalize the `.app` with Xcode (`Failed to finalize the .app with Xcode`) for any C++ project, the engine's own blank template included. Developing in the editor, building and hot reload work.
