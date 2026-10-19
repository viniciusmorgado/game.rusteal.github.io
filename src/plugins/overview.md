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
- **The game cannot use a plugin's Rust types directly.** The game's crate and the plugin's are separate libraries, so the game cannot `use` the plugin's classes. It reaches them as Blueprints do: through their Unreal functions with a [dynamic call](../engine_api/dynamic_calls.md), through an interface, or through the engine (a subsystem found by class path).
- **Editor UI beyond menus.** Slate UI, details panel customizations, custom asset editors, scripted asset actions, importers and Blueprint nodes are not covered yet.
