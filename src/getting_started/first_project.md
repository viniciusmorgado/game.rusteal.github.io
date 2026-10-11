# Your first project

**The goal:** a new Unreal project with a Rust actor that writes to the log when the game starts.

## Creating the project

```bash
rusteal new MyGame --dir ~/Projects
```

`rusteal new` does everything a first build needs:

1. creates the Unreal project in `~/Projects/MyGame`, from the engine's Blank template;
2. installs the Rusteal plugins into its `Plugins/`;
3. writes the Rust workspace in `Rust/`, with your game crate and a `HelloActor`;
4. runs the whole build pipeline: it builds the project with the Unreal Build Tool, generates the Rust bindings for your engine, builds the game crate and deploys the library.

The first build takes a while: it compiles the engine module and generates bindings for the whole engine API. The builds after it only compile your crate.

`rusteal new` does not run `git init`, but it writes a `.gitignore` that covers Unreal's and Cargo's build output, so the right files are picked up when you do.

## The Rust code

The game crate is in `Rust/my-game/`. Its `src/lib.rs` declares the library and its modules:

```rust
rusteal_runtime::entry!();

mod hello;
```

`entry!()` is required once per library: it is the function the Unreal plugin calls when it loads the library. Every other file is a normal Rust module.

`src/hello.rs` declares the actor:

```rust
use bindings::prelude::*;
use rusteal_runtime::{uclass, uclass_impl};

#[uclass(parent = Actor)]
pub struct HelloActor {}

#[uclass_impl]
impl HelloActor {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        log::info!("[MyGame] Hello from Rust");
    }
}
```

- `#[uclass(parent = Actor)]` makes `HelloActor` an Unreal class whose parent is `AActor`.
- `#[uclass_impl]` marks the block where its Unreal functions are.
- `#[ufunction(Override)]` on `receive_begin_play` implements the engine's `ReceiveBeginPlay` event, the one a Blueprint's **Event BeginPlay** node implements.
- `log::info!` writes to the Output Log (see [Logging](../iteration/logging.md)).

## In the editor

1. Open `MyGame.uproject`.
2. In the main menu, open **Window > Place Actors**.
3. In the **Place Actors** panel, type `HelloActor` in the search box.
4. Drag **HelloActor** from the panel into the level viewport.
5. Press **Play** in the toolbar.
6. Open the **Output Log** tab (**Window > Output Log** if it is not open).

The log shows:

```text
LogRusteal: [MyGame] Hello from Rust
```

> **Screenshot:** the Place Actors panel with HelloActor found by the search, and the Output Log showing the message. *(images/getting_started/hello_actor.png)*

## Changing the code

Change the message in `hello.rs`, then build the crate again:

```bash
cd ~/Projects/MyGame
rusteal build
```

The editor reloads the library as soon as the build deploys it, without a restart, and shows a notification. Press **Play** again to see the new message. [The iteration loop](../iteration/hot_reload.md) explains what a reload changes, and how `rusteal watch` builds on every save.

## The flags of `rusteal new`

| Flag | What it does |
|---|---|
| `--dir <path>` | The directory the project is created in (the current one by default). |
| `--template <name>` | The template to start from, `blank` by default. See [Templates](templates.md). |
| `--variant <name>` | The template's variant. |
| `--no-build` | Writes the project and stops; run `rusteal build --all` later. |
