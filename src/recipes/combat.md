# Health, damage and death

## Use case: health with the engine's damage system

**The goal:** a character that loses health when anything damages it, through the engine's `ApplyDamage`, which Blueprints and other systems also use.

The dealer applies damage:

```rust
use bindings::engine::{DamageType, GameplayStatics, PawnExt};

let instigator_controller = me.get_instigator().checked().map(|pawn| pawn.get_controller()).unwrap_or_default();

GameplayStatics::apply_damage(
    hit_actor,
    25.0,
    instigator_controller,
    me.as_ref().upcast_to(),
    SubclassOf::<DamageType>::base(),
);
```

The damaged character overrides `ReceiveAnyDamage`, the event the engine raises for it:

```rust
use bindings::engine::{Controller, DamageType};

#[uclass(parent = Character)]
pub struct Soldier {
    #[uproperty(EditAnywhere, name = "MaxHP", category = "Health", default = 100.0)]
    max_hp: f32,

    #[uproperty(name = "CurrentHP")]
    current_hp: f32,
}

#[uclass_impl]
impl Soldier {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        self.set_current_hp(self.max_hp());
    }

    #[ufunction(Override)]
    fn receive_any_damage(
        &mut self,
        damage: f32,
        _damage_type: UObjectRef<DamageType>,
        _instigated_by: UObjectRef<Controller>,
        _damage_causer: UObjectRef<Actor>,
    ) {
        if self.current_hp() <= 0.0 {
            return;
        }

        self.set_current_hp(self.current_hp() - damage);

        if self.current_hp() <= 0.0 {
            self.die();
        }
    }

    #[ufunction(BlueprintImplementableEvent, name = "BP_OnDeath")]
    fn bp_on_death(&self) {}
}
```

For damage in an area, `GameplayStatics::apply_radial_damage` damages every actor in a radius that is not blocked by a wall. The shooter template does it by hand instead, with an overlap and `apply_damage` per actor, to push physics bodies too (see [Projectiles](projectiles.md)).

## Use case: dying as a ragdoll

**The goal:** on death the character stops, falls as a ragdoll, and is removed a few seconds later.

```rust
use bindings::engine::{
    CharacterExt, CharacterMovementComponentExt, ECollisionEnabled, KismetSystemLibrary,
    MovementComponentExt, PrimitiveComponentExt,
};

impl Soldier {
    fn die(&mut self) {
        let Ok(me) = self.as_ref().checked() else {
            return;
        };

        let _ = me.tags().push(&FName::new("Dead").handle());

        if let Ok(movement) = me.get_character_movement().checked() {
            movement.stop_movement_immediately();
            movement.disable_movement();
        }

        if let Ok(capsule) = me.get_capsule_component().checked() {
            capsule.set_collision_enabled(ECollisionEnabled::NoCollision);
        }

        if let Ok(mesh) = me.get_mesh().checked() {
            mesh.set_collision_profile_name(FName::new("Ragdoll").handle(), Some(true));
            mesh.set_simulate_physics(true);
        }

        self.bp_on_death();

        KismetSystemLibrary::k2_set_timer(me.as_ref().upcast_to(), "RemoveBody", 5.0, false, None, None, None);
    }
}
```

`RemoveBody` is a `#[ufunction]` that calls `k2_destroy_actor`. The Blueprint child implements **BP On Death** for the sound and the effects.

## Use case: respawning the player

**The goal:** when the player's character is destroyed, a new one spawns at a player start and the controller possesses it.

```rust
use bindings::engine::{ControllerExt, Pawn, PlayerController, PlayerStart};

#[uclass(parent = PlayerController)]
pub struct SoldierController {
    #[uproperty(EditAnywhere)]
    character_class: SubclassOf<Soldier>,
}

#[uclass_impl]
impl SoldierController {
    #[ufunction(Override)]
    fn receive_possess(&mut self, possessed_pawn: UObjectRef<Pawn>) {
        let controller: UObjectRef<PlayerController> = self.as_ref();

        let bound = possessed_pawn.checked().and_then(|pawn| {
            pawn.on_destroyed().add(move |_destroyed| {
                if let Ok(mut me) = SoldierController::from_obj(controller)
                    && let Err(e) = me.respawn()
                {
                    ulog!(LOG_WARNING, "[MyGame] respawn failed: {e}");
                }
            })
        });

        match bound {
            Ok(binding) => binding.detach(),
            Err(e) => ulog!(LOG_WARNING, "[MyGame] cannot watch the pawn: {e}"),
        }
    }
}

impl SoldierController {
    fn respawn(&mut self) -> RustealResult<()> {
        let me = self.as_ref();

        let starts = GameplayStatics::get_all_actors_of_class(
            me.upcast_to(),
            SubclassOf::<PlayerStart>::base().upcast_to(),
        );

        let Some(start) = starts.first() else {
            return Ok(());
        };

        let transform = start.checked()?.get_transform();
        let character = me.get_world()?.spawn_actor_of_class(self.character_class(), &transform)?;
        me.checked()?.possess(character.upcast_to());

        Ok(())
    }
}
```

The Blueprint child of the controller sets **Character Class** to the character's Blueprint.

## Use case: damage between your own classes

When only Rust classes deal and take the damage, and they need more than a number (a hit location, a knockback impulse), a Rust trait is simpler than the engine's damage system. See [Interfaces](../classes/interfaces.md), "Rust traits between Rust classes", and the `combat` variant of `third-person`, whose `interfaces.rs` does exactly this.
