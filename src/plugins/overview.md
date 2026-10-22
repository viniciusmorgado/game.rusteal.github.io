# Plugins

A Rusteal plugin is an Unreal plugin whose code is a Rust library of its own, loaded next to the game's. A gameplay system written once (an inventory, dialogue, quests, AI behaviors) goes into any Rusteal project as a plugin; an editor tool brings Rust into the editor: work on many assets at once, parsers and libraries from crates.io, all of it off the game thread when it is heavy.

| Kind | Runs in | What for |
|---|---|---|
| `runtime` | games and the editor | gameplay systems, anything a packaged game needs |
| `editor` | the editor only | tools: menus, asset validation, editor scripting |

An editor plugin can also be packaged for projects without Rusteal, Blueprint and C++ ones, carrying what it needs from Rusteal inside it (see [Editor tools for any project](standalone.md)). A runtime plugin can also have an editor library next to its runtime one: the gameplay system and its editor tools in one plugin (see [A runtime and an editor library](new_plugin.md#a-runtime-and-an-editor-library)).

Everything in [Classes](../classes/uclass.md), [The engine API](../engine_api/object_refs.md) and [Systems](../systems/subsystems.md) applies to a plugin's code: classes, properties, functions, components, delegates and interfaces are written the same way.

## Where a plugin's classes live

A plugin's classes are in its module's package, as a C++ plugin's are: `/Script/Inventory.InventoryComponent`, where the game's are in `/Script/Rusteal`. An editor library's are in its own module's, `/Script/InventoryEditor`. That is their path in assets, ini sections and editor Python:

```python
unreal.load_class(None, "/Script/Inventory.InventoryItem")
```

## What a plugin cannot do yet

- **An editor library cannot use its runtime library's Rust types.** In a plugin with both, the editor library reaches the runtime library's classes as Blueprints do, through their Unreal functions; the build refuses an editor crate that depends on the runtime crate.
- **The game sees a plugin's classes as it sees the engine's** (since 0.10): their properties, functions and events, not their Rust-only fields, and it can derive from them. See [A plugin's classes in the game](game_uses_plugin.md).
- **Editor UI beyond menus.** Slate UI, details panel customizations, custom asset editors, scripted asset actions, importers and Blueprint nodes are not covered yet.
