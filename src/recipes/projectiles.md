# Projectiles and weapons

## Use case: a projectile

**The goal:** a bullet that flies, bounces, damages what it hits, and is removed after a while.

```rust
use bindings::engine::{
    Character, DamageType, ECollisionEnabled, ECollisionResponse, FHitResult, FHitResultExt,
    GameplayStatics, PawnExt, PrimitiveComponent, PrimitiveComponentExt,
    ProjectileMovementComponent, ProjectileMovementComponentExt, SphereComponent,
    SphereComponentExt
};

#[uclass(parent = Actor)]
pub struct Bullet {
    #[component(root, name = "Collision Component")]
    collision: SphereComponent,

    #[component(name = "Projectile Movement")]
    movement: ProjectileMovementComponent,

    #[uproperty(EditAnywhere, category = "Hit", default = 25.0)]
    hit_damage: f32,

    #[uproperty(EditAnywhere, category = "Hit", default = 100.0)]
    physics_force: f32,

    hit: bool,
}

#[uclass_impl]
impl Bullet {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let collision = self.collision()?.checked()?;
        collision.set_sphere_radius(16.0, Some(true));
        collision.set_collision_enabled(ECollisionEnabled::QueryAndPhysics);
        collision.set_collision_response_to_all_channels(ECollisionResponse::ECR_Block);

        let movement = self.movement()?.checked()?;
        movement.set_initial_speed(3000.0);
        movement.set_max_speed(3000.0);
        movement.set_should_bounce(true);

        self.as_ref().checked()?.set_initial_life_span(5.0);

        Ok(())
    }

    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        if let (Ok(collision), Ok(me)) = (self.collision().and_then(|c| c.checked()), self.as_ref().checked()) {
            collision.ignore_actor_when_moving(me.get_instigator().upcast_to(), true);
        }
    }

    #[ufunction(Override)]
    #[allow(clippy::too_many_arguments)]
    fn receive_hit(
        &mut self,
        _my_comp: UObjectRef<PrimitiveComponent>,
        other: UObjectRef<Actor>,
        other_comp: UObjectRef<PrimitiveComponent>,
        _self_moved: bool,
        _hit_location: UStructRef<FVector>,
        _hit_normal: UStructRef<FVector>,
        _normal_impulse: UStructRef<FVector>,
        hit: UStructRef<FHitResult>,
    ) {
        if self.hit() {
            return;
        }

        self.set_hit(true);

        let Ok(me) = self.as_ref().checked() else {
            return;
        };

        if other.cast::<Character>().is_ok() {
            let instigator = me.get_instigator().checked().map(|pawn| pawn.get_controller()).unwrap_or_default();
            GameplayStatics::apply_damage(other, self.hit_damage(), instigator, me.as_ref().upcast_to(), SubclassOf::<DamageType>::base());
        }

        if let Ok(component) = other_comp.checked()
            && component.is_simulating_physics(None)
        {
            let direction = -hit.get_impact_normal().to_dvec3();
            component.add_impulse_at_location(
                &FVector::from_dvec3(direction * f64::from(self.physics_force())),
                &FVector::from_dvec3(hit.get_impact_point().to_dvec3()),
                None,
            );
        }

        self.bp_on_hit(&hit.to_owned());
        me.k2_destroy_actor();
    }

    #[ufunction(BlueprintImplementableEvent, name = "BP_OnProjectileHit")]
    fn bp_on_hit(&self, hit: &OwnedStruct<FHitResult>) {}
}
```

- The `ProjectileMovementComponent` moves it; the root sphere collides.
- `set_initial_life_span` removes it after five seconds if it hits nothing.
- The Blueprint child gives it a mesh, and implements **BP On Projectile Hit** for the impact effect and sound.

The weapon spawns it deferred, so the instigator is set before `BeginPlay` runs (see [Spawning actors](../engine_api/spawning.md)).

## Use case: a weapon with a magazine and a refire rate

**The goal:** a weapon that fires while the trigger is held (or once per press), counts its bullets, and tells the HUD.

The shooter template splits it in two: the rules in `model.rs`, tested without the engine, and the weapon class that applies them.

```rust
pub enum TriggerPull {
    FireNow,
    FireIn(f32),
    Wait,
}

pub fn pull_trigger(time_since_last_shot: f32, refire_rate: f32, full_auto: bool) -> TriggerPull {
    if time_since_last_shot > refire_rate {
        TriggerPull::FireNow
    } else if full_auto {
        TriggerPull::FireIn(refire_rate - time_since_last_shot)
    } else {
        TriggerPull::Wait
    }
}

pub fn bullets_after_shot(bullets: i32, magazine_size: i32) -> i32 {
    let left = bullets - 1;
    if left <= 0 { magazine_size } else { left }
}
```

```rust
pub fn start_firing(&mut self) {
    self.set_is_firing(true);

    let now = GameplayStatics::get_time_seconds(self.as_ref().upcast_to()) as f32;

    match model::pull_trigger(now - self.time_of_last_shot(), self.refire_rate(), self.b_full_auto()) {
        TriggerPull::FireNow => self.fire(),
        TriggerPull::FireIn(delay) => self.set_refire_timer(method(self, Self::fire), delay),
        TriggerPull::Wait => {}
    }
}

pub fn stop_firing(&mut self) {
    self.set_is_firing(false);
    self.clear_refire_timer();
}

fn set_refire_timer(&mut self, callback: impl FnMut() + Send + 'static, delay: f32) {
    self.clear_refire_timer();

    let refire_handle = timer::set_timer(&self.as_ref(), Duration::from_secs_f32(delay), false, callback).ok();
    self.set_refire_handle(refire_handle);
}

fn clear_refire_timer(&self) {
    if let Some(refire_handle) = self.refire_handle() {
        refire_handle.clear();
    }
}
```

`refire_handle` is a Rust-private field, `Option<TimerHandle>`: one timer at a time waits for the next shot, and `stop_firing` clears it (see [Timers](../engine_api/timers.md)).

`fire` spawns the projectile, plays the firing montage on the holder, adds recoil (`add_controller_pitch_input`), updates the bullet count and sets the timer for the next shot. See `shooter/weapon.rs` in a `first-person --variant shooter` project.

## Use case: attaching a weapon to a hand

**The goal:** the weapon actor follows the character's hand.

```rust
use bindings::engine::{EAttachmentRule, SceneComponentExt};

let snap = EAttachmentRule::SnapToTarget;

weapon.checked()?.k2_attach_to_actor(me.as_ref().upcast_to(), FName::NONE.handle(), snap, snap, snap, false);

weapon_mesh.checked()?.k2_attach_to_component(
    character_mesh.upcast_to(),
    FName::new("HandGrip_R").handle(),
    snap,
    snap,
    snap,
    false,
);
```

The weapon is spawned with the character as its owner, and destroys itself with it:

```rust
let weapon = self.as_ref();

me.get_owner()
    .checked()?
    .on_destroyed()
    .add(move |_destroyed| {
        if let Ok(weapon) = weapon.checked() {
            weapon.k2_destroy_actor();
        }
    })?
    .detach();
```

## Use case: telling the AI a shot was heard

```rust
me.make_noise(
    Some(1.0),
    Some(pawn_owner),
    &pawn.k2_get_actor_location(),
    Some(3000.0),
    Some(FName::new("Shot").handle()),
);
```

`make_noise` is heard by AI perception's hearing sense, with the tag the AI filters on. See [AI](ai.md).
