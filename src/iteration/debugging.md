# Debugging

**The goal:** stop at a breakpoint in Rust code while the editor or the game runs, inspect the Rust values, and step between Rust and the engine's C++.

The Rust code runs inside the Unreal process, so the debugger attaches to that process: the editor started by `rusteal watch`, or a game.

## VS Code

```bash
rusteal new MyGame --template third-person --feature vscode
```

The `vscode` feature (since 0.10) adds `Rust/.vscode`, for VS Code opened on the project's `Rust` folder:

| File | What it holds |
|---|---|
| `tasks.json` | **Rusteal Watch** (`cargo rusteal watch`), **Rusteal Build** and **Rusteal Test** |
| `launch.json` | **Attach to Unreal (Rust)**, CodeLLDB's attach, which asks for the process; **Attach to Unreal (Visual Studio debugger)** for Windows |
| `extensions.json` | rust-analyzer and CodeLLDB |

Run the **Rusteal Watch** task, wait for the editor, then **Run and Debug** > **Attach to Unreal (Rust)** and pick `UnrealEditor`. Breakpoints in the crate's files bind once the library is loaded.

## RustRover

The `rustrover` feature adds the **Rusteal Watch** and **Rusteal Test** run configurations. Start **Rusteal Watch**, then **Run** > **Attach to Process** and pick `UnrealEditor`.

## Visual Studio and Xcode

Visual Studio (**Debug** > **Attach to Process**, native code) and Xcode (**Debug** > **Attach to Process**) attach to the editor the same way and read the Rust library's debug info. These recipes, and the VS Code ones, have not been tested on every desktop yet.

## Debug info

The development profile gives the game crate full debug info and the dependencies and the bindings file and line (see [Development and release builds](../cli/profiles.md)). A packaged game is built with the release profile, without debug info; give it some for crash reports or a debugger with Cargo's environment, which overrides the profile:

```bash
CARGO_PROFILE_RELEASE_DEBUG=line-tables-only rusteal package
```

`line-tables-only` gives backtraces with files and lines; `true` gives the debugger everything, and a much larger library.
