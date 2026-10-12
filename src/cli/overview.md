# The rusteal CLI

Every Rusteal task is a `rusteal` command. Commands that act on a project take its directory as an argument, or find it by walking up from the current directory, so inside a project you can leave the path out.

| Command | What it does | Page |
|---|---|---|
| `rusteal new <Name>` | Creates a project from a template and builds it. | [Your first project](../getting_started/first_project.md) |
| `rusteal setup <path>` | Installs the Rusteal plugins into an existing project. | [Adding Rusteal to an existing project](../getting_started/existing_project.md) |
| `rusteal build` | Builds the Rust libraries and deploys them. | [Building](build.md) |
| `rusteal build --all` | Runs the whole pipeline: engine build, bindings, Rust. | [Building](build.md) |
| `rusteal watch` | Builds every time a Rust source file changes. | [rusteal watch](../iteration/watch.md) |
| `rusteal generate` | Regenerates the bindings crate and the C++ wrappers. | [Building](build.md) |
| `rusteal upgrade` | Moves the project to the CLI's Rusteal version. | [Versions and upgrades](../getting_started/versions.md) |
| `rusteal package` | Builds the release libraries and packages the game; `--static` links the Rust code into it, `--platform` targets another platform. | [Packaging a game](../shipping/packaging.md) |
| `rusteal platform list`, `show <name>` | Lists the platforms a game can be packaged for, or prints one's definition. | [Packaging for another platform](../shipping/packaging.md#packaging-for-another-platform) |
| `rusteal plugin new <Name>` | Creates a Rust plugin in the project. | [A new plugin](../plugins/new_plugin.md) |
| `rusteal plugin package <Name>` | Copies a plugin out, ready for another project. | [Packaging a plugin](../plugins/packaging.md) |

`rusteal help <command>` and `rusteal <command> --help` list the flags of each command.

## A day with the CLI

A typical session, once the project exists:

```bash
cd ~/Projects/MyGame
rusteal watch
```

Open the editor, change Rust code, save: `rusteal watch` builds, the editor reloads the library, and you press **Play**. When you change `rusteal.toml` or the engine version, stop the watch and run `rusteal build --all` once. When the game is ready to be tried outside the editor, run `rusteal package`.

## Use case: running the CLI from the IDE

**The goal:** start `rusteal watch` from the IDE, with one run configuration that works on Linux, macOS and Windows.

Every command is also a Cargo subcommand: `cargo rusteal watch` is `rusteal watch` (`cargo install rusteal` installs both). In RustRover:

1. Open **Run > Edit Configurations...**.
2. Click **+** and choose **Cargo**.
3. Name it `rusteal watch`, and set **Command** to `rusteal watch`.
4. Set **Working directory** to the project's `Rust` directory: the command finds the project by walking up from it.
5. Tick **Store as project file** to share it with the rest of the team, then click **OK**.

> **Screenshot:** RustRover's Run/Debug Configurations with a Cargo configuration whose command is rusteal watch. *(images/cli/rustrover_watch.png)*
