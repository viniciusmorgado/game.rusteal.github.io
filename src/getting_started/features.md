# Adding features

A feature is a piece of gameplay any template can take: a Rusteal plugin with its Rust classes, and the engine content they need. `--feature` adds it when the project is created, as many times as wanted:

```bash
rusteal new MyGame --template third-person --feature mannequin
```

| Feature | Plugin | What it is |
|---|---|---|
| `mannequin` | `Mannequin` | `CopycatAnimInstance`, a parent for Animation Blueprints that copies a character's movement, and `BP_CopycatMannequin`, the Intro To Unreal template's mannequin that walks, runs and jumps with the player; `ReachingMannequin`, whose right hand reaches for a `ReachTarget` placed in the level. |
| `checkpoints` | `Checkpoints` | `BP_SpawnPoint`s that save where the player got to, and a subsystem that starts the level there. |
| `interactables` | `Interactables` | A trigger (`BP_DoorTrigger`) and what it drives: lights, a door it unlocks, a confetti burst; a trigger that plays a level sequence, a MetaSound with a seed, and blocks dangling from physics constraints. |
| `tour` | `Tour` | The pieces of the Intro To Unreal tour: texts that change in game, station titles, animated keyboard keys and mouse, gizmo demos, template cubes, decal arrows along a spline and editor panels shown in game. |

Each feature becomes a plugin in `Plugins/<Plugin>/`, enabled in the `.uproject` and built with the project: its Rust classes are in `Plugins/<Plugin>/Rust/`, and `rusteal build` builds them with the game's. When the project lacks the engine content the feature uses, it is copied in: `mannequin` brings the `Characters` pack (`Content/Characters/Mannequins`), which the third person and first person templates already have, and `interactables` and `tour` the `LevelPrototyping` pack. Each feature's Blueprints are in its plugin's content: in the **Content Browser**, click **Settings** and check **Show Plugin Content**, then open the plugin's folder, **Plugins > Mannequin Content** for the `mannequin` feature.

The `intro` template is made of all four: `rusteal new MyTour --template intro` adds them by itself, and the Blueprints of its tour level are children of their Rust classes.

An existing project takes a feature as a plugin:

```bash
rusteal plugin new Mannequin --template mannequin
```

A feature whose plugin carries assets keeps its plugin name, since its assets live under `/<Plugin>/`.

## Use case: the IDE set up from the start

**The goal:** a new project that opens in RustRover with its run configurations ready, the same on Linux and Windows.

Some features add files to the project instead of a plugin. `rustrover` adds two RustRover run configurations, stored as project files in `Rust/.run/`:

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

## A mannequin that copies the player

1. Create a project with `--feature mannequin`, or add the plugin to an existing one.
2. In the **Content Browser**, right-click and choose **Animation > Animation Blueprint**; pick the **SK_Mannequin** skeleton and, as the **Parent Class**, **CopycatAnimInstance**.
3. Open it. In the **My Blueprint** panel, click the gear icon and check **Show Inherited Variables**: **Ground Speed**, **Direction**, **Should Move**, **Is Falling**, **Velocity** and **Character** appear under **Copycat**.
4. In its **AnimGraph**, blend the idle, walk and run animations by **Ground Speed** and **Direction**, and switch to a jump loop when **Is Falling** is set.
5. Drag **SKM_Manny_Simple** from `Content/Characters/Mannequins/Meshes` into the level.
6. In its **Details** panel, set **Animation Mode** to **Use Animation Blueprint**.
7. Set **Anim Class** to the new Animation Blueprint.
8. Press **Play** and walk: the mannequin walks, runs and jumps with you.

> **Screenshot:** a copycat mannequin running beside the player in the third person level. *(images/getting_started/feature_copycat.png)*

The copying is plain Rust in `Plugins/Mannequin/Rust/mannequin/src/copycat.rs`: the speed ignores the vertical axis, the mannequin moves only when the speed is above 0.01 and the character accelerates, and the direction is kept within 45 degrees when the character turns toward its movement.
