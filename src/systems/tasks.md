# Work off the game thread

**The goal:** heavy work, pathfinding for hundreds of agents, procedural generation, parsing a big file, done on other cores, with the result applied to the game when it is ready.

Unreal's objects belong to the game thread: engine calls are made from there. Everything else is plain Rust, and runs anywhere: threads, `rayon`, `tokio`. `task::spawn` connects the two:

```rust
use rusteal_runtime::runtime::task;

let seed = self.seed();

task::spawn(
    move || generate_dungeon(seed),
    move |dungeon| {
        if let Ok(mut level) = LevelBuilder::from_obj(builder) {
            level.place_rooms(&dungeon);
        }
    },
);
```

- The first closure runs on a thread of its own. It must not call the engine.
- The second runs on the game thread, on a frame after the first finishes, with its result. There, the engine can be called again.

Capture `UObjectRef`s, not Rust class values, in the second closure, and turn them back into the class with `from_obj`: the object may have been destroyed while the work ran.

## Using every core

`rayon` spreads work over all cores inside the first closure:

```rust
use rayon::prelude::*;

task::spawn(
    move || agents.par_iter().map(|agent| plan_path(agent, &grid)).collect::<Vec<_>>(),
    move |paths| apply_paths(paths),
);
```

Add `rayon` to the game crate's `Cargo.toml` like any dependency.

## Other tools

| Function | What it does |
|---|---|
| `task::run_on_game_thread(closure)` | Runs a closure on the game thread, on the next frame, from any thread. |
| `task::on_tick(closure)` | Runs a closure every frame, the editor's frames too, until the returned handle's `remove()`. |
| `task::is_game_thread()` | Whether the code runs on the game thread. |

```rust
let handle = task::on_tick(|delta_seconds| {
});

handle.remove();
```

## Hot reload and shutdown

A library waits for its spawned work before it unloads (a hot reload, the editor closing), so no thread runs code that is gone. A result still waiting for the game thread at that moment is dropped. Keep spawned work short, or split it, so a reload does not wait for long.

A panic in the work is caught and logged, and the second closure does not run.
