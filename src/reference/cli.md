# CLI reference

Commands that take `[project]` find the project by walking up from the current directory when it is left out.

## rusteal new

```bash
rusteal new <Name> [--dir <path>] [--template <name>] [--variant <name>] [--feature <name>]... [--no-build] [--optimizations <on|off>] [--engine <path>] [--in-place]
```

Creates a project from a template, installs the plugins, adds each `--feature` as a plugin, writes the Rust workspace and runs the whole build. See [Your first project](../getting_started/first_project.md), [Starting from a template](../getting_started/templates.md) and [Adding features](../getting_started/features.md).

`--engine` uses that engine instead of the configured one (and saves it when none is configured). `--in-place` sets up a project the editor made from a Rust template, in its own directory; the launcher runs it (see [Creating a project from the editor](../getting_started/from_the_editor.md)).

## rusteal setup

```bash
rusteal setup <project>
```

Installs the Rusteal plugins into an existing project and writes a starter `rusteal.toml`. See [Adding Rusteal to an existing project](../getting_started/existing_project.md).

## rusteal build

```bash
rusteal build [project] [--all | --from <n> | --step <n>] [--release] [--plugin <Name>]
```

Without flags, steps 4 and 5: build the Rust libraries and deploy them. See [Building](../cli/build.md).

## rusteal watch

```bash
rusteal watch [project] [--plugin <Name>] [--optimizations <on|off>]
```

Writes the [platform optimizations](../cli/optimizations.md), then runs `rusteal build` every time a Rust source changes, until `Ctrl+C`. See [rusteal watch](../iteration/watch.md).

## rusteal generate

```bash
rusteal generate [project]
```

Regenerates the bindings crate and the C++ wrappers from the exported reflection.

## rusteal upgrade

```bash
rusteal upgrade [project]
```

Moves the project to the CLI's Rusteal version. See [Versions and upgrades](../getting_started/versions.md).

## rusteal package

```bash
rusteal package [project] [--shipping] [--output <dir>] [--static] [--platform <name|file>] [--optimizations <on|off>] [--server]
```

Builds the release libraries and packages the game, with the [platform optimizations](../cli/optimizations.md) unless `--optimizations off`. `--static` links the Rust code into the game's executable; `--platform` packages for another platform, from its name or its definition file; `--server` packages a dedicated server, with an engine built from source (see [Multiplayer](../systems/multiplayer.md#a-dedicated-server)). See [Packaging a game](../shipping/packaging.md).

## rusteal platform

```bash
rusteal platform list
rusteal platform show <name|file>
```

Lists the built-in platforms and the definition files found, or prints a platform's definition. See [Packaging for another platform](../shipping/packaging.md#packaging-for-another-platform).

## rusteal plugin new

```bash
rusteal plugin new <Name> [--template <name>] [--project <path>] [--no-build] [--editor]
```

Creates a Rust plugin in the project; `--editor` gives a runtime plugin an editor library too. See [A new plugin](../plugins/new_plugin.md).

## rusteal plugin editor

```bash
rusteal plugin editor <Name> [--project <path>] [--no-build]
```

Adds an editor library to a runtime plugin. See [A runtime and an editor library](../plugins/new_plugin.md#a-runtime-and-an-editor-library).

## rusteal plugin package

```bash
rusteal plugin package <Name> [--project <path>] [--output <dir>] [--precompiled | --standalone]
rusteal plugin merge <into> <package>...
```

Copies a plugin out with its release library; `--precompiled` builds its C++ modules too, with the engine's `BuildPlugin`, so the project that takes it compiles nothing. `--standalone` packages an editor plugin for any project, Blueprint and C++ ones included, with what it needs from Rusteal inside it and no Rust sources (see [Editor tools for any project](../plugins/standalone.md)). `merge` joins packages made on several platforms into the first. See [Packaging a plugin](../plugins/packaging.md).

## rusteal launcher

```bash
rusteal launcher install [--engine <path>]
rusteal launcher remove [--engine <path>]
```

Builds the launcher with the engine's `BuildPlugin` and installs it in `Engine/Plugins/Marketplace/RustealLauncher`, or removes it. See [Creating a project from the editor](../getting_started/from_the_editor.md).

## Console commands

| Command | What it does |
|---|---|
| `Rusteal.Reload` | Reloads every Rust library of the project. |
| `Rusteal.Reload <Name>` | Reloads one library: the project's name for the game's, the plugin's name for a plugin's, `<Plugin>Editor` for a plugin's editor library. |
| `Rusteal.AutoReload 0` | Stops reloading on deploy. |
| `Rusteal.AutoReload 1` | Reloads on deploy (the default). |
| `Rusteal.Launcher.Create` | In a project made from a Rust template, runs its setup again, as the **Retry** button does. |

See [Console commands](../iteration/console.md).
