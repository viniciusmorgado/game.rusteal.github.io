# Editor tools for any project

An editor plugin written in Rust can be packaged for projects that do not use Rusteal: Blueprint projects, C++ projects, or Rusteal ones. The plugin carries what it needs from Rusteal inside it, runs only in the editor, and never goes into a packaged game. This is how an editor tool made with Rusteal is shared or sold.

| | Editor tool for any project | Rusteal plugin |
|---|---|---|
| Runs in | the editor only | the editor and, for a runtime plugin, the game |
| Projects | any: Blueprint, C++, Rusteal | Rusteal projects |
| Installed | in the engine, `Engine/Plugins/Marketplace`, as plugins from Fab | in the project's `Plugins/` |
| Rust sources | not included: the library is prebuilt | included |
| Made with | `rusteal plugin package --standalone` | `rusteal plugin package [--precompiled]` |

## Packaging one

Make the tool as any editor plugin, in a Rusteal project where you can try it (see [A new plugin](new_plugin.md) and [Editor tools](editor_tools.md)):

```bash
rusteal plugin new LevelTools --template editor
```

Then package it:

```bash
rusteal plugin package LevelTools --standalone --output ~/Packages
```

It gives two directories:

- `~/Packages/LevelTools`: the plugin, built by the engine for the platform it ran on, ready to be installed.
- `~/Packages/LevelTools-Source`: the same plugin as sources, with its Rust library prebuilt and no Rust code, for a store that builds plugins itself, as Fab does.

Only editor plugins (`kind = "editor"`) can be packaged this way. A gameplay plugin goes into a game, so it needs the game to be a Rusteal one.

## Installing it

Copy the packaged directory into the engine's `Engine/Plugins/Marketplace/`, then, in each project that uses it, enable it:

1. In the editor's main menu, open **Edit > Plugins**.
2. Search for the plugin's name.
3. Tick its **Enabled** box.
4. Restart the editor when it asks.

The project builds nothing for it: a Blueprint project stays a Blueprint project.

## What the user of the tool gets

The tool works as it does in your project: its menu entries, console commands, asset validators and editor subsystems. Its classes are in its own packages, and its log lines and console commands carry its name (`LevelToolsRusteal.Reload`), so two such tools, from you or from anyone, and the Rusteal of a Rusteal project, never meet.

## What it cannot do

- **Only the platform it was packaged on.** Package it on each platform the tool supports and join the packages with [`rusteal plugin merge`](packaging.md#several-platforms-in-one-package).
- **The same engine version.** The plugin is built against one engine version, like any code plugin.
- **No Rust sources.** Fixing the tool means packaging it again from your project.
