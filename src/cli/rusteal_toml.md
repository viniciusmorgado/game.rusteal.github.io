# Binding more of the engine

**The goal:** use an engine module the project does not bind yet, such as AI, Niagara, StateTree or Gameplay Tags.

The bindings crate only has the engine modules `rusteal.toml` asks for. A new project binds the core of the engine, input, UMG and Enhanced Input. The other modules are listed in `rusteal.toml`, each behind a feature, and a feature is turned on by adding it to `features`.

## The file

```toml
[project]
crate = "my-game"

[codegen]
features = ["core", "engine", "input", "slate", "umg", "enhanced-input"]

[codegen.modules]
CoreUObject = { module = "core_ue", feature = "core" }
Engine = { module = "engine", feature = "engine" }
AIModule = { module = "ai", feature = "ai" }
GameplayTasks = { module = "gameplay_tasks", feature = "ai" }
NavigationSystem = { module = "navigation", feature = "ai" }
StateTreeModule = { module = "state_tree", feature = "state-tree", plugin = "StateTree" }
GameplayStateTreeModule = { module = "gameplay_state_tree", feature = "state-tree", plugin = "GameplayStateTree" }
Niagara = { module = "niagara", feature = "niagara", plugin = "Niagara" }
```

| Key | What it is |
|---|---|
| `[project] crate` | The package name of your game crate in `Rust/`. |
| `[codegen] features` | The features whose modules are bound. |
| `[codegen.modules]` | Each engine module (by its package name), the Rust module it becomes in `bindings`, the feature it belongs to, and the engine plugin it comes from when it is not part of the engine itself. |
| `[codegen.blocklist]` | Engine classes, structs and functions left out of the bindings, because the generated code cannot handle them. |

## Turning a module on

1. Add the feature to `features`:

   ```toml
   features = ["core", "engine", "input", "slate", "umg", "enhanced-input", "ai", "state-tree"]
   ```

2. Run the whole pipeline:

   ```bash
   rusteal build --all
   ```

3. Use the module from `bindings`:

   ```rust
   use bindings::ai::{AIController, AIControllerExt};
   use bindings::navigation::NavigationSystemV1;
   ```

When a module comes from an engine plugin (`plugin = "StateTree"`), the build enables that plugin for Rusteal, so the engine loads it with your game.

## The modules a new project lists

| Feature | Modules | What it brings |
|---|---|---|
| `core` | `core_ue` | `UObject`, math structs (`FVector`, `FRotator`, `FTransform`...) |
| `engine` | `engine` | actors, components, characters, game framework, gameplay statics, physics, audio, animation |
| `physics-core` | `physics_core` | physical materials |
| `input` | `input_core` | keys (`FKey`) |
| `enhanced-input` | `enhanced_input` | input actions, mapping contexts, triggers |
| `slate`, `umg` | `slate_core`, `slate`, `umg` | UMG widgets |
| `ai` | `ai`, `gameplay_tasks`, `navigation` | AI controllers, perception, behavior trees, EQS, navigation |
| `state-tree` | `state_tree`, `gameplay_state_tree` | StateTree components, tasks, conditions |
| `gameplay-tags` | `gameplay_tags` | gameplay tags and containers |
| `niagara` | `niagara` | Niagara systems and components |
| `chaos-vehicles` | `chaos_vehicles` | Chaos wheeled vehicles: pawns, movement, wheels, engine and gearbox setups |
| `gameplay-abilities` | `gameplay_abilities` | the Gameplay Ability System |
| `level-sequence`, `cinematic`, `movie` | `level_sequence`, `cinematic_camera`, `movie_scene`, `movie_scene_tracks` | Sequencer, cinematic cameras |

`engine` needs `input`, `slate` and `umg`, because the Rusteal plugin links them: keep them in the list.

## Binding another module

Any reflected engine module can be added to `[codegen.modules]` the same way, with a feature of your choice:

```toml
DeveloperSettings = { module = "developer_settings", feature = "engine" }
```

## Engine virtuals

`[[codegen.shims]]` declares C++ bases that raise an engine class's virtual functions as events a Rust class overrides, built with the project by `rusteal build --all`. See [Engine virtuals](../classes/virtuals.md#bases-of-your-own).

## The blocklist

A few engine functions are reflected but not exported by the engine, so their wrapper compiles but does not link. The blocklist a new project gets already lists the ones known for the engine version Rusteal targets. If a `rusteal build --all` fails at link time on a wrapper of a new module, add the function to `[codegen.blocklist] functions` as `"Class.Function"` and build again.

## Bigger bindings, longer builds

Every module added makes the bindings crate larger, and its first build longer. The bindings are compiled once and kept, so a change to your own code does not rebuild them. Bind what the game uses.
