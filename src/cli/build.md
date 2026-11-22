# Building

A Rusteal build has five steps:

| Step | Name | What it does |
|---|---|---|
| 1 | UE build | Builds the project with the Unreal Build Tool, which exports the engine's reflection to `Intermediate/Rusteal/uht/`. |
| 2 | Codegen | Reads the reflection and generates the `bindings` crate and the C++ wrappers. |
| 3 | UE rebuild | Builds the generated C++ wrappers into the Rusteal plugin. |
| 4 | Cargo build | Builds your game crate (and each Rust plugin's) with Cargo. |
| 5 | Deploy | Copies the libraries to the plugins' `Binaries/<Platform>/`, where the editor loads them. |

## Which build to run

| You changed | Run |
|---|---|
| Rust code in your crate | `rusteal build` |
| Rust code, many times while you work | `rusteal watch` |
| `rusteal.toml` (engine modules, blocklist) | `rusteal build --all` |
| The engine version | `rusteal build --all` |
| A cloned project, built for the first time on this machine | `rusteal build --all` |

`rusteal build` runs steps 4 and 5: that is all a change to Rust code needs, and it takes seconds. `rusteal build --all` runs the five, and takes as long as the engine build and the bindings generation take.

## Flags

```bash
rusteal build
rusteal build --all
rusteal build --from 2
rusteal build --step 3
rusteal build --release
rusteal build --plugin Inventory
rusteal build /path/to/MyGame
```

| Flag | What it does |
|---|---|
| `--all` | Runs every step. |
| `--from <n>` | Runs step `n` and every step after it. |
| `--step <n>` | Runs only step `n`. |
| `--release` | Builds the libraries with the release profile, for shipping. |
| `--plugin <Name>` | Limits codegen, Cargo and deploy to one Rust plugin of the project, both of its libraries when it has two. |

`--all` cannot be combined with `--from` or `--step`.

## Generating the bindings alone

```bash
rusteal generate
```

Regenerates the `bindings` crate and the C++ wrappers from the reflection already exported, without building anything.

## When the build fails

The build prints the output of the tool that failed: Cargo's errors for your code, the Unreal Build Tool's for the engine steps. A Rust compile error leaves the library the editor runs as it was: fix the code and build again.

The Unreal Build Tool runs one build at a time per engine. If another build is running, from the editor or from another project, wait for it to finish.

If `rusteal build` says the bindings are missing, the project has never been built on this machine, or `Rust/bindings/` was deleted: run `rusteal build --all`.
