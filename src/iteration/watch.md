# rusteal watch

**The goal:** save a Rust file and have the change in the editor, with nothing else to run.

```bash
cd ~/Projects/MyGame
rusteal watch
```

`rusteal watch` builds the Rust libraries every time a source file changes, the same build as `rusteal build`. With the editor reloading each library when it is deployed, saving is the whole loop: save, build, deploy, reload.

```text
rusteal watch: watching
  /home/me/Projects/MyGame/Rust
Each change builds and deploys the libraries (rusteal build); Ctrl+C stops.
```

Leave it running in a terminal next to the editor. `Ctrl+C` stops it.

## What it watches

The `.rs` and `.toml` files of the project's Rust workspace, and of each Rust plugin's workspace. It does not watch `target/` or the generated `bindings/`.

It waits until the files have settled before building, so saving several files at once, or a formatter rewriting them, makes one build.

## When the build fails

The build's errors are printed, and the watch goes on:

```text
rusteal watch: the build failed (above), waiting for changes
```

The editor keeps running the last library that built. Fix the error and save: the next build deploys and reloads.

## One plugin

```bash
rusteal watch --plugin Inventory
```

Watches and builds only that plugin's library, which is what working on a plugin needs.

## What it does not do

`rusteal watch` runs steps 4 and 5 of the build. A change to `rusteal.toml` or to the engine version needs `rusteal build --all`: stop the watch, run it, and start the watch again.
