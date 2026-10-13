# Adding features

A feature is a piece of gameplay any template can take: a Rusteal plugin with its Rust classes, and the engine content they need. `--feature` adds it when the project is created, as many times as wanted:

```bash
rusteal new MyGame --template third-person --feature mannequin
```

| Feature | Plugin | What it is |
|---|---|---|
| [`mannequin`](../features/mannequin.md) | `Mannequin` | `CopycatAnimInstance`, a parent for Animation Blueprints that copies a character's movement, and `BP_CopycatMannequin`, the Intro To Unreal template's mannequin that walks, runs and jumps with the player; `ReachingMannequin`, whose right hand reaches for a `ReachTarget` placed in the level. |
| [`checkpoints`](../features/checkpoints.md) | `Checkpoints` | `BP_SpawnPoint`s that save where the player got to, and a subsystem that starts the level there. |
| [`interactables`](../features/interactables.md) | `Interactables` | A trigger (`BP_DoorTrigger`) and what it drives: lights, a door it unlocks, a confetti burst; a trigger that plays a level sequence, a MetaSound with a seed, and blocks dangling from physics constraints. |
| [`tour`](../features/tour.md) | `Tour` | The pieces of the Intro To Unreal tour: texts that change in game, station titles, animated keyboard keys and mouse, gizmo demos, template cubes, decal arrows along a spline and editor panels shown in game. |

Each feature becomes a plugin in `Plugins/<Plugin>/`, enabled in the `.uproject` and built with the project: its Rust classes are in `Plugins/<Plugin>/Rust/`, and `rusteal build` builds them with the game's. When the project lacks the engine content the feature uses, it is copied in: `mannequin` brings the `Characters` pack (`Content/Characters/Mannequins`), which the third person and first person templates already have, and `interactables` and `tour` the `LevelPrototyping` pack. Each feature's Blueprints are in its plugin's content: in the **Content Browser**, click **Settings** and check **Show Plugin Content**, then open the plugin's folder, **Plugins > Mannequin Content** for the `mannequin` feature.

The `intro` template is made of all four: `rusteal new MyTour --template intro` adds them by itself, and the Blueprints of its tour level are children of their Rust classes.

An existing project takes a feature as a plugin:

```bash
rusteal plugin new Mannequin --template mannequin
```

A feature whose plugin carries assets keeps its plugin name, since its assets live under `/<Plugin>/`.

## Features that add files

Some features add files to the project instead of a plugin: the setup of a tool around it.

| Feature | Adds | What it is |
|---|---|---|
| [`rustrover`](../features/rustrover.md) | `Rust/.run/` | RustRover run configurations, stored as project files: **Rusteal Watch** and **Rusteal Test**, the same on Linux, macOS and Windows. |
