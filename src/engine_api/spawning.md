# Spawning actors

**The goal:** put actors into the level from code: projectiles, enemies, pickups, effects.

## Spawning a Blueprint chosen in the editor

The usual case: the class to spawn is a Blueprint child (`BP_Projectile`), picked in a `SubclassOf` property.

```rust
use bindings::engine::{ESpawnActorCollisionHandlingMethod, GameplayStatics, KismetMathLibrary};
use bindings::prelude::*;

#[uclass(parent = Character)]
pub struct Shooter {
    #[uproperty(EditAnywhere, category = "Projectile")]
    projectile_class: SubclassOf<Projectile>,
}

impl Shooter {
    fn spawn_projectile(&self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        let location = me.k2_get_actor_location().to_dvec3()
            + me.get_actor_forward_vector().to_dvec3() * 100.0;

        let transform = KismetMathLibrary::make_transform(
            &FVector::from_dvec3(location),
            &me.k2_get_actor_rotation(),
            &FVector::from_dvec3(DVec3::ONE),
        );

        let world = self.as_ref().get_world()?;
        world.spawn_actor_of_class(self.projectile_class(), &transform)?;

        Ok(())
    }
}
```

`get_world()` gives the world an object lives in, and `spawn_actor_of_class` spawns an actor of a class held in a `SubclassOf`. It returns an `UObjectRef<T>` to the new actor, or an error if the class is not set or the spawn failed.

## Deferred spawn: setting values before BeginPlay

To set an actor up before its `BeginPlay` runs (an owner, an instigator, a team), spawn it deferred and finish the spawn after:

```rust
let spawned = GameplayStatics::begin_deferred_actor_spawn_from_class(
    me.as_ref().upcast_to(),
    self.projectile_class().upcast_to(),
    &transform,
    Some(ESpawnActorCollisionHandlingMethod::AlwaysSpawn),
    Some(me.as_ref().upcast_to()),
    None,
);

if let Ok(projectile) = spawned.checked() {
    projectile.set_instigator(me.as_ref().upcast_to());
}

let spawned = GameplayStatics::finish_spawning_actor(spawned, &transform, None);

if let Ok(projectile) = Projectile::from_obj(spawned) {
    projectile.set_damage(self.damage());
}
```

The fifth argument is the owner. The collision handling method decides what happens when the spawn point is blocked:

| Method | What happens |
|---|---|
| `AlwaysSpawn` | Spawns where asked, overlapping or not. |
| `AdjustIfPossibleButAlwaysSpawn` | Moves the actor out of the way if it can, spawns anyway. |
| `AdjustIfPossibleButDontSpawnIfColliding` | Moves the actor out of the way, or does not spawn. |
| `DontSpawnIfColliding` | Does not spawn if the point is blocked: the returned reference is not valid. |

## Spawning a Rust class directly

A Rust class with no Blueprint child, or an engine class, is spawned by type:

```rust
let transform = FTransform::from_transform(Transform::new(DQuat::IDENTITY, location, DVec3::ONE));
let gem: UObjectRef<Gem> = world.spawn_actor(&transform)?;
```

Without a Blueprint child, no assets are assigned: give the actor its mesh in code, or prefer a `SubclassOf` property and a Blueprint.

An actor class without a root component ignores the spawn transform; Rusteal logs a warning when that happens.

## Destroying

```rust
me.k2_destroy_actor();
me.set_life_span(5.0);
```

`k2_destroy_actor` destroys the actor at the end of the frame. `set_life_span` destroys it after that many seconds.

## Finding actors

```rust
let world_context = self.as_ref().upcast_to();

let enemies = GameplayStatics::get_all_actors_of_class(
    world_context,
    SubclassOf::<Enemy>::base().upcast_to(),
);

let starts = GameplayStatics::get_all_actors_of_class_with_tag(
    world_context,
    SubclassOf::<PlayerStart>::base().upcast_to(),
    FName::new("Player1").handle(),
);

let player_pawn = GameplayStatics::get_player_pawn(world_context, 0);
let player_controller = GameplayStatics::get_player_controller(world_context, 0);
let game_mode = GameplayStatics::get_game_mode(world_context);
```

`get_all_actors_of_class` walks every actor of the level: fine on a key press or at the start of a level, too slow every frame for a big level. Keep the actors you need in a property, or let them register themselves with a subsystem.

Your Rust game mode, from any actor:

```rust
if let Ok(mut game_mode) = MyGameMode::from_obj(GameplayStatics::get_game_mode(world_context)) {
    game_mode.add_score(10);
}
```
