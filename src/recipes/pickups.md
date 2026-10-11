# Pickups

## Use case: a coin that counts

**The goal:** the player walks into a coin, the game mode counts it, the UI shows the count, and the Blueprint plays the effect.

```rust
use bindings::engine::{
    ECollisionChannel, ECollisionEnabled, ECollisionResponse, GameplayStatics,
    PrimitiveComponentExt, SceneComponent, SphereComponent, SphereComponentExt
};

#[uclass(parent = Actor)]
pub struct Coin {
    #[component(root)]
    root: SceneComponent,

    #[component(attach = "root", name = "Collision")]
    sphere: SphereComponent,
}

#[uclass_impl]
impl Coin {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let sphere = self.sphere()?.checked()?;
        sphere.set_sphere_radius(60.0, Some(false));
        sphere.set_collision_enabled(ECollisionEnabled::QueryOnly);
        sphere.set_collision_response_to_all_channels(ECollisionResponse::ECR_Ignore);
        sphere.set_collision_response_to_channel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_actor_begin_overlap(&mut self, other_actor: UObjectRef<Actor>) {
        if let Err(e) = self.pick_up(other_actor) {
            log::warn!("[MyGame] pickup failed: {e}");
        }
    }

    #[ufunction(BlueprintImplementableEvent, name = "BP_OnPickedUp")]
    fn bp_on_picked_up(&self) {}
}

impl Coin {
    fn pick_up(&mut self, other_actor: UObjectRef<Actor>) -> RustealResult<()> {
        let Ok(character) = other_actor.cast::<Character>() else {
            return Ok(());
        };

        if !character.checked()?.is_player_controlled() {
            return Ok(());
        }

        let game_mode = GameplayStatics::get_game_mode(self.as_ref().upcast_to());

        if let Ok(mut game_mode) = CoinGameMode::from_obj(game_mode) {
            game_mode.add_coin()?;
        }

        self.as_ref().checked()?.set_actor_enable_collision(false);
        self.bp_on_picked_up();

        Ok(())
    }
}
```

The collision only overlaps pawns, so the coin does not block anything. The Blueprint child gives it a mesh, and its **BP On Picked Up** graph plays a sound and hides or destroys it. The game mode keeps the count and updates the UI: see [UI](ui.md), "A counter on screen".

## Use case: a weapon pickup that respawns

**The goal:** a pickup gives its weapon to whoever walks in, hides, and comes back after a few seconds.

```rust
#[ufunction(Override)]
fn receive_actor_begin_overlap(&mut self, other_actor: UObjectRef<Actor>) {
    let Some(mut holder) = weapon_holder(other_actor) else {
        return;
    };

    holder.add_weapon_class(self.weapon_class());

    let Ok(me) = self.as_ref().checked() else {
        return;
    };

    me.set_actor_hidden_in_game(true);
    me.set_actor_enable_collision(false);

    KismetSystemLibrary::k2_set_timer(me.as_ref().upcast_to(), "RespawnPickup", self.respawn_time(), false, None, None, None);
}

#[ufunction]
fn respawn_pickup(&mut self) {
    if let Ok(me) = self.as_ref().checked() {
        me.set_actor_hidden_in_game(false);
        me.set_actor_enable_collision(true);
    }
}

#[ufunction(Override)]
fn receive_end_play(&mut self, _end_play_reason: u8) {
    KismetSystemLibrary::k2_clear_timer(self.as_ref().upcast_to(), "RespawnPickup");
}
```

`weapon_holder` turns the actor into a Rust trait that both the player and the NPCs implement (see [Interfaces](../classes/interfaces.md)). Which weapon the pickup gives comes from a data table row: see [Structs and data tables](../classes/structs.md).
