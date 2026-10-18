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

## In a packaged game

A game packaged with [`rusteal package`](../shipping/packaging.md) stages the libraries of its runtime plugins next to its own. Editor plugins, and the editor library of a plugin that has both, are left out, as the engine leaves out editor modules.
