# rustrover

**The goal:** a new project that opens in RustRover with its run configurations ready, the same on Linux, macOS and Windows.

`rustrover` is a feature that adds files to the project instead of a plugin: two RustRover run configurations, stored as project files in `Rust/.run/`:

```bash
rusteal new MyGame --template third-person --feature rustrover
```

| Run configuration | Runs | For |
|---|---|---|
| **Rusteal Watch** | `cargo rusteal watch` | Building and reloading the library every time a Rust file is saved, with the editor open. |
| **Rusteal Test** | `cargo test -p <game crate>` | The game crate's unit tests (see [Testing without the engine](../iteration/testing.md)). |

1. In RustRover, choose **File > Open** and open the project's `Rust` folder.
2. Pick **Rusteal Watch** or **Rusteal Test** in the run configurations next to the **Run** button, and click **Run**.

They are files of the project, so they go into version control with it and the rest of the team gets them too.

> **Screenshot:** RustRover's run configurations list with Rusteal Watch and Rusteal Test. *(images/getting_started/rustrover_features.png)*
