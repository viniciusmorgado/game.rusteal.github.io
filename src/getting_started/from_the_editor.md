# Creating a project from the editor

Rusteal can also create a project from Unreal's **New Project** dialog, next to the Blueprint and C++ templates: an engine plugin, the Rusteal launcher, adds a **Rust** category with every Rusteal template. The project it creates is the same one `rusteal new` creates.

## Installing the launcher

The launcher is installed once per engine, from the CLI:

```bash
rusteal launcher install
```

It builds the launcher with the engine's own tools (about a minute) and installs it in `Engine/Plugins/Marketplace/RustealLauncher`, so the engine directory must be writable. `--engine <path>` picks another engine than the configured one, and `rusteal launcher remove` uninstalls it.

## Creating a project

1. Open the editor without a project: the **Unreal Project Browser** opens.
2. In the column of categories on the left, click **Rust**.
3. Click a template: **Blank**, **First Person**, **Third Person: Combat**, and so on. Every variant of a Rusteal template is a template of its own.
4. Under **Project Name**, type the name: letters and digits only, starting with a letter, 20 characters at most.
5. Click **Create**.

The dialog warns that the template comes from a plugin that is not enabled by default: the launcher is that plugin, and the warning does not stop anything.

The project opens, and the **Creating the Rust project** window shows up on its own: Rusteal turns the project into a Rust project from the template you picked and builds it, with its log in the window. This is the same first build `rusteal new` runs, and it takes a few minutes. When it is done, the editor restarts into the new project, with its Rust library loaded.

## What it needs

The window checks two things before it starts, and offers a button for each one that is missing:

| Requirement | Missing | Button |
|---|---|---|
| Rust (`cargo`) | **Not found** | **Get Rust** opens [rustup.rs](https://rustup.rs). Install Rust, then click **Check again**. |
| The Rusteal CLI | **Not found**, or a version too old for the Rust templates | **Install Rusteal** or **Update Rusteal** runs `cargo install rusteal --locked`, with its log in the window, and starts the project when it is done. |

The launcher looks for both on `PATH` and in Cargo's own directory (`~/.cargo/bin`, or `CARGO_HOME`), so a Rust installed after the editor started is found without restarting it.

## When it stops

- **Cancel** stops the setup; **Retry** starts it again.
- A setup that fails or is canceled leaves the project as it was made from the template, so opening it again starts over. Closing the editor in the middle does the same.
- A project name the CLI refuses is reported in the window before anything runs: create the project again with another name.
- Every line of the setup's log is also in the editor's log (`Saved/Logs/<Project>.log`), under `LogRustealLauncher`.

The console command `Rusteal.Launcher.Create` runs the setup again, as **Retry** does.

## Using a checkout of Rusteal

To create projects with a Rusteal CLI built from a checkout instead of the installed one, start the editor with `RUSTEAL_CLI` pointing at it (on Linux; `UnrealEditor.exe` on Windows, `UnrealEditor.app` on macOS):

```bash
RUSTEAL_CLI=<checkout>/target/debug/rusteal <engine>/Engine/Binaries/Linux/UnrealEditor
```
