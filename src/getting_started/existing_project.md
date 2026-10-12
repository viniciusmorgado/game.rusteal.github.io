# Adding Rusteal to an existing project

**The goal:** Rust gameplay in a project that already exists, made from the editor or with C++.

```bash
rusteal setup /path/to/YourProject
```

`rusteal setup` installs the Rusteal plugins into the project's `Plugins/` and writes a starter `rusteal.toml`. If the project already has a `rusteal.toml`, it keeps it.

It does not write the Rust workspace. Create it as `rusteal new` does (see [Project layout](layout.md)):

1. `Rust/Cargo.toml`, the workspace, listing your crate and `bindings`, and pinning the Rusteal crates to the CLI's version (`rusteal --version`);
2. `Rust/<crate>/Cargo.toml`, a `cdylib` depending on `rusteal-runtime`, `bindings` and `glam`;
3. `Rust/<crate>/src/lib.rs`, with `rusteal_runtime::entry!();`;
4. in `rusteal.toml`, `[project] crate = "<crate>"`, the package name of your crate.

The quickest way is to create a blank project with `rusteal new` next to it and copy its `Rust/` directory and the profiles of its `Rust/Cargo.toml`.

Then build everything once:

```bash
cd /path/to/YourProject
rusteal build --all
```

The project needs a C++ module (`Source/<Project>/`) for the Unreal Build Tool to build it. A project made from the editor's Blueprint templates has none: add one from the editor (**Tools > New C++ Class**, any class) before running the build. On macOS that needs Xcode, as in [Prerequisites](prerequisites.md#macos-apple-silicon).

Your existing C++ and Blueprint classes stay as they are, and work next to the Rust ones: Blueprints derive from Rust classes, call their functions and bind their delegates the way they do with C++ classes.
