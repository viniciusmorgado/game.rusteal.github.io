# A new plugin

**The goal:** a Rust plugin, built and loaded in a project where you can try it.

A plugin lives in a Rusteal project, which builds it with the engine and gives it a level to be tried in: the game it is for, or a project made for it.

```bash
rusteal new PluginLab
cd PluginLab
rusteal plugin new Inventory
rusteal plugin new LevelTools --template editor
```

`rusteal plugin new` writes the plugin into the project's `Plugins/`, lists it in the `.uproject` and runs the build pipeline. The plugin's name is also its C++ module's: letters and digits, not the project's own name.

| Flag | What it does |
|---|---|
| `--template <name>` | The plugin template, `blank` by default. |
| `--project <path>` | The project to add it to, when not run from inside it. |
| `--no-build` | Writes the plugin and stops. |

## Plugin templates

| Template | What it is |
|---|---|
| `blank` | A runtime plugin with a `HelloComponent` in Rust. |
| `editor` | An editor plugin: an editor subsystem, a **Tools** menu entry that counts the project's assets on a background thread, a console command doing the same, and an asset validator. |
| `inventory` | A runtime gameplay plugin: items as data assets, an inventory component that stacks them and broadcasts its changes, a world subsystem, project settings for its limits, and console commands that give, take and list the player's items. See [An inventory](../recipes/inventory.md). |
| `mannequin` | The `mannequin` feature, a mannequin that copies the player's movement. See [Adding features](../getting_started/features.md). |

## Layout

```text
Plugins/Inventory/
├── Inventory.uplugin           # the plugin, depending on Rusteal
├── rusteal.toml                # the plugin's library: crate, kind, engine modules
├── Source/Inventory/           # its C++ module, which hands the library to Rusteal
│   └── Generated/              # generated C++ wrappers
├── Rust/
│   ├── Cargo.toml              # the plugin's own workspace
│   ├── inventory/              # its code
│   └── bindings/               # its generated bindings
├── Binaries/<Platform>/        # librusteal_Inventory.so, rusteal_Inventory.dll
└── Content/                    # the plugin's assets
```

The plugin carries everything it needs: the directory can be copied into another Rusteal project at the same version, or kept in a repository of its own. Its `Rust/Cargo.toml` pins the Rusteal crates as the project's does, and `rusteal upgrade` moves both.

## The plugin's rusteal.toml

```toml
[plugin]
crate = "inventory"
kind = "runtime"

[codegen]
features = ["core", "engine", "input", "slate", "umg"]
```

`kind` is `runtime` or `editor`. The `[codegen]` section works as the game's (see [Binding more of the engine](../cli/rusteal_toml.md)). A module far bigger than a plugin needs can be limited to some classes, and the ones they derive from:

```toml
UnrealEd = { module = "unreal_ed", feature = "editor", classes = ["EditorActorSubsystem", "EditorAssetSubsystem"] }
```

## Building and reloading

```bash
rusteal build
rusteal build --plugin Inventory
rusteal watch --plugin Inventory
```

`rusteal build` builds every library of the project, the game's and each plugin's. `--plugin <Name>` limits the codegen, Cargo and deploy steps to one plugin, which is what working on it needs. In the editor, `Rusteal.Reload Inventory` reloads only that plugin's library, and each build reloads it on its own (see [Hot reload](../iteration/hot_reload.md)).

## In the editor

The plugin shows in **Edit > Plugins**, under the project's plugins, and its content folder in the **Content Browser** once **Settings > Show Plugin Content** is on.

> **Screenshot:** the Plugins window listing the Rust plugin. *(images/plugins/plugins_window.png)*
