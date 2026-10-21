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

- The first closure runs as a background task of the engine's task system, on one of its worker threads, like the engine's own background work (it shows up in Unreal Insights). It must not call the engine.
- The second runs on the game thread, on a frame after the first finishes, with its result. There, the engine can be called again.

Capture `UObjectRef`s, not Rust class values, in the second closure, and turn them back into the class with `from_obj`: the object may have been destroyed while the work ran.

In async code, `task::run_blocking(move || generate_dungeon(seed)).await` does the same in one line, and goes on with the result where it left off. See [Async code](../engine_api/async.md).

## Using every core

`rayon` spreads work over all cores. Rusteal runs it on the engine's own threads: turn on the runtime's `rayon` feature in the game crate's `Cargo.toml`,

```toml
rusteal-runtime = { workspace = true, features = ["rayon"] }
```

and run the parallel work through `rusteal_runtime::rayon::install`, inside the first closure:

```rust
use rusteal_runtime::rayon::{self, prelude::*};

task::spawn(
    move || {
        rayon::install(|| agents.par_iter().map(|agent| plan_path(agent, &grid)).collect::<Vec<_>>())
    },
    move |paths| {
        if let Ok(paths) = paths {
            apply_paths(paths);
        }
    },
);
```

`install` runs the closure in your library's pool: one per library, with as many threads as the engine gives its own workers, started by the engine (they show up as `Rusteal rayon` threads in Unreal Insights) the first time it is used, and stopped and waited for when the library unloads. The prelude is `rayon`'s own, re-exported, so the game needs no `rayon` dependency of its own; `rusteal_runtime::rayon::ThreadPool::new(threads)` makes a pool of another size, which stops when it is dropped.

Avoid `rayon`'s global pool (a `par_iter()` outside `install`): its threads are never stopped, so a [hot reload](../iteration/hot_reload.md) unloads the code they run, and the editor can crash. On a console they would also be the operating system's threads, not the engine's (see [Getting ready for consoles](../shipping/consoles.md)).

## A thread of its own

For work that runs as long as the game does, such as a streaming loader or a network loop, `task::spawn_thread` starts a dedicated engine thread:

```rust
let loader = task::spawn_thread("Level streamer", move || stream_levels(requests))?;
```

`join()` waits for it; a thread whose handle is dropped is waited for when the library unloads, so a thread must end when what it waits on closes. Engine calls still belong on the game thread: send results back with `task::run_on_game_thread`.

## Other tools

| Function | What it does |
|---|---|
| `task::run_on_game_thread(closure)` | Runs a closure on the game thread, on the next frame, from any thread. |
| `task::on_tick(closure)` | Runs a closure every frame, the editor's frames too, until the returned handle's `remove()`. |
| `task::is_game_thread()` | Whether the code runs on the game thread (the engine's `IsInGameThread`). |
| `task::worker_count()` | How many worker threads the engine uses on this machine. |

```rust
let handle = task::on_tick(|delta_seconds| {
});

handle.remove();
```

## Measuring time

`rusteal_runtime::runtime::time` reads the engine's clock, the one the engine's own profiling uses:

```rust
use rusteal_runtime::runtime::time::Instant;

let started = Instant::now();
let dungeon = generate_dungeon(seed);
log::info!("generated in {:?}", started.elapsed());
```

`Instant` has `now`, `elapsed`, `duration_since`, `checked_duration_since`, and adds and subtracts `Duration`s, like `std::time::Instant`. For gameplay timing, the world's time (`GameplayStatics::get_time_seconds`) follows pause and time dilation; `Instant` does not.

## Hot reload and shutdown

A library waits for its spawned work before it unloads (a hot reload, the editor closing), so no thread runs code that is gone. A result still waiting for the game thread at that moment is dropped. Keep spawned work short, or split it, so a reload does not wait for long; a task that never ends also holds one of the engine's worker threads.

A panic in the work is caught and logged, and the second closure does not run.
