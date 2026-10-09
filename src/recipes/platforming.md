# Platforming

The `platforming` and `side-scrolling` variants of `third-person` are full platformer controllers. Their rules are in `model.rs`, tested without the engine, and their classes apply them.

## Use case: double jump and coyote time

**The goal:** a second jump in the air, and a jump that still works a moment after walking off a ledge.

The rules, plain Rust:

```rust
#[derive(Clone, Copy, Default)]
pub struct MoveState {
    pub has_double_jumped: bool,
    pub last_fall_time: f64,
}

pub enum AirJump {
    Coyote,
    Double,
    None,
}

pub fn air_jump(state: &MoveState, now: f64, max_coyote_time: f32) -> AirJump {
    if now - state.last_fall_time < f64::from(max_coyote_time) {
        AirJump::Coyote
    } else if !state.has_double_jumped {
        AirJump::Double
    } else {
        AirJump::None
    }
}
```

The character records when it starts falling, applies the rules on the jump press, and resets them on landing:

```rust
use bindings::engine::{CharacterExt, GameplayStatics, NavMovementComponentExt};

const MOVE_FALLING: u8 = 3;

#[uclass(parent = Character)]
pub struct Platformer {
    #[uproperty(EditAnywhere, category = "Jump", default = 0.16)]
    max_coyote_time: f32,

    state: MoveState,
}

#[uclass_impl]
impl Platformer {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        me.set_jump_max_hold_time(0.4);
        me.set_jump_max_count(3);
        Ok(())
    }

    #[ufunction(Override, name = "K2_OnMovementModeChanged")]
    fn on_movement_mode_changed(&mut self, _prev_mode: u8, new_mode: u8, _prev_custom: u8, _new_custom: u8) {
        if new_mode == MOVE_FALLING {
            let mut state = self.state();
            state.last_fall_time = GameplayStatics::get_time_seconds(self.as_ref().upcast_to());
            self.set_state(state);
        }
    }

    #[ufunction]
    fn jump_pressed(&mut self) {
        let Ok(me) = self.as_ref().checked() else {
            return;
        };

        let Ok(movement) = me.get_character_movement().checked() else {
            return;
        };

        if !movement.is_falling() {
            me.jump();
            return;
        }

        let mut state = self.state();
        let now = GameplayStatics::get_time_seconds(self.as_ref().upcast_to());

        match air_jump(&state, now, self.max_coyote_time()) {
            AirJump::Coyote => me.jump(),
            AirJump::Double => {
                state.has_double_jumped = true;
                me.jump();
            }
            AirJump::None => {}
        }

        self.set_state(state);
    }

    #[ufunction(Override)]
    fn on_landed(&mut self, _hit: UStructRef<FHitResult>) {
        let mut state = self.state();
        state.has_double_jumped = false;
        self.set_state(state);
    }
}
```

The movement mode arrives as its `u8` value: `3` is `MOVE_Falling` in the engine's `EMovementMode`.

## Use case: a wall jump

**The goal:** jumping while touching a wall bounces off it.

```rust
let me = self.as_ref().checked()?;
let start = me.k2_get_actor_location().to_dvec3();
let end = start + me.get_actor_forward_vector().to_dvec3() * 100.0;

let (hit, out_hit) = KismetSystemLibrary::sphere_trace_single(
    self.as_ref().upcast_to(),
    &FVector::from_dvec3(start),
    &FVector::from_dvec3(end),
    40.0,
    ETraceTypeQuery::TraceTypeQuery1,
    false,
    &[],
    EDrawDebugTrace::None,
    true,
    &OwnedStruct::new(),
    &OwnedStruct::new(),
    None,
);

if hit {
    let normal = out_hit.as_ref().get_impact_normal().to_dvec3();
    let impulse = normal * 800.0 + DVec3::Z * 900.0;
    me.launch_character(&FVector::from_dvec3(impulse), true, true);
}
```

`launch_character(velocity, xy_override, z_override)` replaces the character's velocity with the launch: the bounce does not depend on how fast it was falling.

## Use case: a dash

**The goal:** a quick dash in the air, driven by a montage with root motion, with gravity off while it lasts.

```rust
use bindings::engine::{AnimInstanceExt, MovementComponentExt, SkeletalMeshComponentExt};

fn start_dash(&mut self) -> RustealResult<()> {
    let me = self.as_ref().checked()?;
    let movement = me.get_character_movement().checked()?;
    movement.set_gravity_scale(0.0);
    movement.set_velocity(&FVector::from_dvec3(DVec3::ZERO));

    let anim_instance = me.get_mesh().checked()?.get_anim_instance();
    anim_instance.checked()?.montage_play(self.dash_montage(), Some(1.0), None, Some(0.0), Some(true));

    Ok(())
}

pub fn end_dash(&mut self) {
    if let Ok(movement) = self.as_ref().checked().and_then(|me| me.get_character_movement().checked()) {
        movement.set_gravity_scale(1.0);
    }
}
```

`end_dash` runs from an anim notify at the end of the dash montage (see [Animation](animation.md)), and from the montage's end as a fallback.

## Use case: a jump pad

See [Delegates](../engine_api/delegates.md): a box that overlaps pawns and calls `launch_character` on whoever steps on it.

## Use case: a platform you can drop through

**The goal:** a platform the character lands on, and drops through when pressing down and jump.

The platform's mesh blocks a custom object channel; the character ignores that channel while dropping:

```rust
pub fn set_soft_collision(&self, enabled: bool) -> RustealResult<()> {
    let response = if enabled { ECollisionResponse::ECR_Ignore } else { ECollisionResponse::ECR_Block };

    self.as_ref()
        .checked()?
        .get_capsule_component()
        .checked()?
        .set_collision_response_to_channel(self.soft_collision_object_type(), response);

    Ok(())
}
```

`soft_collision_object_type` is an `EditAnywhere` `ECollisionChannel` property, set in the Blueprint child to the channel the platforms use. The platform's own `receive_actor_end_overlap` turns the collision back on once the character has passed.

### In the editor

1. **Edit > Project Settings**, **Collision**, **New Object Channel**: name it `SoftPlatform`, **Default Response** **Block**.
2. In the platform Blueprint, set its mesh's **Collision Presets** to **Custom** and its **Object Type** to **SoftPlatform**.
3. In the character Blueprint's **Class Defaults**, set **Soft Collision Object Type** to the same channel.

> **Screenshot:** the Collision page of Project Settings with the SoftPlatform object channel. *(images/recipes/soft_platform_channel.png)*

## Use case: a moving platform

**The goal:** a platform that moves to a target when the player interacts with it.

The side-scrolling template leaves the motion to the Blueprint, where a **Timeline** is the natural tool, and keeps the rules in Rust:

```rust
#[uclass(parent = Actor)]
pub struct MovingPlatform {
    #[component(root)]
    root: SceneComponent,

    #[uproperty(EditAnywhere, BlueprintReadOnly)]
    platform_target: OwnedStruct<FVector>,

    #[uproperty(EditAnywhere, BlueprintReadOnly, default = 2.0)]
    move_duration: f32,

    moving: bool,
}

#[uclass_impl]
impl MovingPlatform {
    #[ufunction(BlueprintCallable)]
    fn reset_interaction(&mut self) {
        self.set_moving(false);
    }

    #[ufunction(BlueprintImplementableEvent, name = "BP_MoveToTarget")]
    fn bp_move_to_target(&self) {}
}

impl MovingPlatform {
    pub fn interact(&mut self) {
        if self.moving() {
            return;
        }

        self.set_moving(true);
        self.bp_move_to_target();
    }
}
```

The Blueprint child implements **BP Move To Target** with a **Timeline** that moves the platform to **Platform Target** over **Move Duration**, and calls **Reset Interaction** when it finishes.

To move it from Rust instead, interpolate in `receive_tick` with `k2_set_actor_location`, using the engine's interpolation as plain Rust (`current + (target - current) * (delta * speed).clamp(0.0, 1.0)`).
