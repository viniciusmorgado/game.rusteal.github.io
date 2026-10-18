# Packaging a plugin

**The goal:** a plugin ready to be dropped into another Rusteal project, or shared.

```bash
rusteal plugin package Inventory --output ~/Packages
```

It builds the plugin's library with the release profile and copies the plugin into `~/Packages/Inventory`:

- its sources: the C++ module, the Rust crate, the generated code;
- its content and configuration;
- its binaries for the platform it was built on: the Rust library and the editor module;
- none of the build output.

## Using it in another project

1. Copy the packaged directory into the other project's `Plugins/`. The project must be at the same Rusteal version.
2. List the plugin in the project's `.uproject`, under `Plugins`.
3. Build everything there:

   ```bash
   rusteal build --all
   ```

A binary for another platform comes from packaging the plugin on that platform.

## A precompiled plugin

```bash
rusteal plugin package Inventory --precompiled --output ~/Packages
```

With `--precompiled`, the engine builds the plugin's C++ modules too, with its `BuildPlugin`, for the editor and for games in Development and Shipping, so the project that takes the plugin compiles none of it:

- the binaries: the C++ modules' and the Rust libraries', for the platform it was built on;
- the plugin's sources, its Rust workspace (without `target/`) and `rusteal.toml`, which a [static build](../shipping/packaging.md) compiles the plugin's crates from;
- its content and configuration.

The plugin is marked installed (`"Installed": true` in its `.uplugin`), as the engine marks the plugins it builds: `rusteal build` in the project that takes it leaves it as it is, and says so, instead of building it again. The first time, the command adds `Config/FilterPlugin.ini` to the plugin, the list of the files the engine packages besides its own.

The project that takes it uses it as in [Using it in another project](#using-it-in-another-project), at the same Rusteal and engine version.

## Several platforms in one package

Each platform's binaries come from packaging the plugin on that platform. `rusteal plugin merge` joins them:

```bash
rusteal plugin merge ~/Packages/Inventory ~/FromWindows/Inventory ~/FromMac/Inventory
```

It adds to the first package the binaries and build files of the others' platforms (`Binaries/<Platform>/`, `Intermediate/Build/<Platform>/`), and refuses packages of another plugin or another version.

## In a packaged game

A game packaged with [`rusteal package`](../shipping/packaging.md) stages the libraries of its runtime plugins next to its own. Editor plugins, and the editor library of a plugin that has both, are left out, as the engine leaves out editor modules.
