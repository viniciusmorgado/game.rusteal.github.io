# Getting ready for consoles

**The goal:** a game that ships on PC today and can go to consoles later, when a publisher gets it access, without rewriting its Rust code.

Rusteal does not build for consoles yet: that needs each console's SDK, which is private and given only to studios with access. What can be done before is done, and most of it is automatic. What is left to the game is a few habits.

## What Rusteal already does

- **The Rust code links into the game.** Consoles run native code linked into the game, and `rusteal package --static` packages a PC game the same way (see [Linking the Rust code into the game](packaging.md#linking-the-rust-code-into-the-game)). A game that packages and plays with `--static` has passed the part of a console build that is about Rust.
- **The runtime uses the engine, not the operating system.** Rusteal's background tasks run on the engine's task system, its clock is the engine's, and in a static build Rust's memory comes from the engine's allocator. The engine is already ported to every console it supports, so these work there too.
- **A platform is a file.** A console's definition (its Rust target, toolchain and libraries) is a file the studio keeps privately and gives to `rusteal package --platform`; nothing in the game changes.

## What the game can do

Rust's standard library talks to the operating system for threads, clocks, files and the network. On a console, those parts of the standard library need a port to that console, which may be incomplete. Code that goes through Rusteal and the engine does not depend on it:

| Instead of | Use |
|---|---|
| `std::thread::spawn` | `task::spawn` (see [Work off the game thread](../systems/tasks.md)) |
| `std::time::Instant`, `SystemTime` | `rusteal_runtime::runtime::time::Instant`, or the world's time |
| `std::fs` for saves | the engine's save games (see [Saving the game](../recipes/save_game.md)) |
| A crate that starts its own threads (`rayon`, `tokio`) | `task::spawn` for the work; keep the crate on PC only if you need it (below) |

None of this is required on PC: `std`, `rayon` and any crate keep working there. It is a choice for code that should reach consoles without changes.

## Checking a project

```toml
[consoles]
check = true
```

In `rusteal.toml`, this makes every `rusteal build` list, as warnings, the lines of your crates that use `std::thread` or the `std` clock, and the dependencies known to start their own threads or use the operating system's network and files. Nothing is refused: a dependency may well work once a console's standard library is ported. The list is what to look at first when a console build comes.

## A dependency only on PC

Cargo can make a dependency depend on the platform, and Rust code follow it with `cfg`:

```toml
[target.'cfg(any(target_os = "linux", target_os = "windows", target_os = "macos"))'.dependencies]
rayon = "1"
```

```rust
#[cfg(any(target_os = "linux", target_os = "windows", target_os = "macos"))]
fn plan_paths(agents: &[Agent], grid: &Grid) -> Vec<Path> {
    use rayon::prelude::*;
    agents.par_iter().map(|agent| plan_path(agent, grid)).collect()
}

#[cfg(not(any(target_os = "linux", target_os = "windows", target_os = "macos")))]
fn plan_paths(agents: &[Agent], grid: &Grid) -> Vec<Path> {
    agents.iter().map(|agent| plan_path(agent, grid)).collect()
}
```

Both versions run inside a `task::spawn`, so the game thread never waits for either.

## The engine allocator

In a `--static` package, Rust allocates through the engine's `FMemory`, so its memory shows in the engine's memory tools and follows the platform's allocator. A game that sets its own `#[global_allocator]` turns it off in `rusteal.toml`:

```toml
[static]
engine_allocator = false
```
