# Declaring a class

**The goal:** a new Unreal class, written in Rust, that the editor and Blueprints treat like any other.

```rust
use bindings::engine::{StaticMeshComponent, SceneComponent};
use bindings::prelude::*;
use rusteal_runtime::runtime::{LOG_DISPLAY, ulog};
use rusteal_runtime::{uclass, uclass_impl};

#[uclass(parent = Actor)]
pub struct Barrel {
    #[component(root)]
    root: SceneComponent,

    #[component(attach = "root")]
    mesh: StaticMeshComponent,

    #[uproperty(EditAnywhere, default = 50)]
    health: i32,
}

#[uclass_impl]
impl Barrel {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        ulog!(LOG_DISPLAY, "[MyGame] crate with {} health", self.health());
    }
}
```

A class is two parts:

- **The struct**, marked `#[uclass(parent = ...)]`: its fields are the class's properties (`#[uproperty]`), its components (`#[component]`) and fields only Rust sees (no attribute).
- **The impl block**, marked `#[uclass_impl]`: its methods marked `#[ufunction]` are the class's Unreal functions and events, and `#[class_defaults]` sets its default values. A class can split them over several `#[uclass_impl]` blocks, in several files. Plain `impl` blocks next to them hold the methods only Rust calls.

When the library loads, Rusteal creates the class in the engine under the struct's name: `Barrel` above. It can be placed in a level, used as a Blueprint's parent and picked in any class field of the editor.

## Choosing a parent

The parent is an engine class from the bindings, or another Rust class. What the class is follows from it:

| Parent | What the class is |
|---|---|
| `Actor` | Anything placed in or spawned into a level: props, pickups, volumes, spawners, projectiles. |
| `Pawn`, `Character` | Something a player or an AI controls. `Character` brings a capsule, a skeletal mesh and walking movement. |
| `PlayerController`, `AIController` | What possesses a pawn: input for a player, logic for an AI. |
| `GameModeBase`, `GameStateBase`, `PlayerState` | The rules of a level and their state. |
| `PlayerCameraManager` | How the player's camera is placed every frame. |
| `HUD` | A canvas drawn over the game. |
| `ActorComponent`, `SceneComponent` | Behavior added to actors: an inventory, a health system. |
| `UserWidget` | A UMG widget whose layout a Widget Blueprint child draws. |
| `AnimNotify` | Code run at a frame of an animation. |
| `SaveGame` | Data written to and read from a save slot. |
| `PrimaryDataAsset` | A kind of asset, created and edited in the Content Browser. |
| `DeveloperSettings` | A page of Project Settings. |
| `StateTreeTaskBlueprintBase`, `StateTreeConditionBlueprintBase` | StateTree tasks and conditions. |
| `EnvQueryContext_BlueprintBase` | An EQS context. |
| `RustealGameInstanceSubsystem`, `RustealWorldSubsystem`... | A subsystem. See [Subsystems](../systems/subsystems.md). |

The engine classes keep their C++ names without the prefix (`AActor` is `Actor`), and come from the module they belong to (`bindings::engine::Character`, `bindings::ai::AIController`, `bindings::umg::UserWidget`).

## Reaching the object

Inside the class, `self` is the Rust side of the object. Three methods connect it to the engine:

| Method | What it returns |
|---|---|
| `self.as_ref()` | A reference to the object as its parent class (`UObjectRef<Actor>` for `Barrel`), to call engine functions on it. |
| `MyClass::from_obj(object)` | The Rust class of any object reference, or an error if the object is not of that class. |
| `MyClass::get_default()` | The class default object, which every instance starts from. |

```rust
let me = self.as_ref().checked()?;
let location = me.k2_get_actor_location().to_dvec3();
```

```rust
if let Ok(mut barrel) = Barrel::from_obj(other_actor) {
    barrel.set_health(barrel.health() - 10);
}
```

`from_obj` is how one Rust class reaches another: an overlap gives an `UObjectRef<Actor>`, and `from_obj` turns it into the Rust class, with its getters, setters and methods, when that is what it is. See [Object references](../engine_api/object_refs.md).

## Where the class lives

The game's classes are in the `/Script/Rusteal` package: `/Script/Rusteal.Barrel` is the class path of `Barrel`, the path an ini file, a redirect or editor Python uses. A plugin's classes are in its own package (see [Plugins](../plugins/overview.md)).

## What a class cannot do yet

- Replicate properties or call RPCs over the network: `Replicated` and the RPC specifiers are not accepted yet.

See [Known limits](../reference/limits.md).
