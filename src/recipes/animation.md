# Animation

## Use case: playing a montage and knowing when it ends

**The goal:** an attack plays its animation montage, and the character can attack again once it ends.

```rust
use bindings::engine::{AnimInstance, AnimInstanceExt, AnimMontage, Character, CharacterExt, SkeletalMeshComponentExt};

#[uclass(parent = Character)]
pub struct Fighter {
    #[uproperty(EditAnywhere, category = "Combat")]
    attack_montage: UObjectRef<AnimMontage>,

    is_attacking: bool,
    montage_end_bound: bool,
}

impl Fighter {
    fn attack(&mut self) -> RustealResult<()> {
        if self.is_attacking() {
            return Ok(());
        }

        let anim_instance = self.as_ref().checked()?.get_mesh().checked()?.get_anim_instance();
        self.bind_montage_end(anim_instance)?;

        anim_instance.checked()?.montage_play(self.attack_montage(), Some(1.0), None, Some(0.0), Some(true));
        self.set_is_attacking(true);

        Ok(())
    }

    fn bind_montage_end(&mut self, anim_instance: UObjectRef<AnimInstance>) -> RustealResult<()> {
        if self.montage_end_bound() {
            return Ok(());
        }

        let character: UObjectRef<Character> = self.as_ref();

        anim_instance
            .checked()?
            .on_montage_ended()
            .add(move |montage, _interrupted| {
                if let Ok(mut me) = Fighter::from_obj(character)
                    && montage == me.attack_montage()
                {
                    me.set_is_attacking(false);
                }
            })?
            .detach();

        self.set_montage_end_bound(true);

        Ok(())
    }
}
```

- `montage_play(montage, play_rate, return_value_type, time_to_start, stop_all_montages)` plays it; `None` keeps a default.
- `on_montage_ended` is the anim instance's delegate for every montage that ends, interrupted or not: the closure checks which one ended.
- A Rust-only flag keeps the closure from being bound twice.

Other montage controls on the anim instance:

```rust
anim_instance.montage_jump_to_section(FName::new("Attack2").handle(), Some(montage));
anim_instance.montage_stop(0.1, Some(montage));
```

## Use case: code at a frame of an animation

**The goal:** the sword's damage trace runs at the exact frame of the swing, set by an animator in the montage.

```rust
use bindings::engine::{ActorComponentExt, AnimNotify, AnimSequenceBase, FAnimNotifyEventReference, SkeletalMeshComponent};

#[uclass(parent = AnimNotify)]
pub struct AnimNotifyAttackTrace {
    #[uproperty(EditAnywhere)]
    attack_bone_name: FName,
}

#[uclass_impl]
impl AnimNotifyAttackTrace {
    #[ufunction(Override, name = "Received_Notify")]
    fn notify(
        &self,
        mesh_comp: UObjectRef<SkeletalMeshComponent>,
        _animation: UObjectRef<AnimSequenceBase>,
        _event_reference: UStructRef<FAnimNotifyEventReference>,
    ) -> bool {
        let Ok(owner) = mesh_comp.checked().map(|mesh| mesh.get_owner()) else {
            return false;
        };

        if let Ok(mut fighter) = Fighter::from_obj(owner) {
            fighter.do_attack_trace(&self.attack_bone_name().to_string_lossy());
        }

        true
    }
}
```

A class whose parent is `AnimNotify` is a notify animators place on a timeline. It receives the skeletal mesh component that plays the animation; its owner is the character.

### In the editor

1. Open the montage (`AM_Attack`).
2. In the **Notifies** track, right-click at the frame of the hit and choose **Add Notify > Anim Notify Attack Trace**.
3. Select the notify, and in the **Details** panel set **Attack Bone Name** (`hand_r`).
4. **Save**.

> **Screenshot:** a montage's Notifies track with the Rust notify placed at the hit frame, and its Attack Bone Name in the Details panel. *(images/recipes/anim_notify.png)*

## Use case: gameplay state for the Animation Blueprint

An Animation Blueprint reads the character's state through `BlueprintPure` functions:

```rust
#[ufunction(BlueprintPure)]
fn has_double_jumped(&self) -> bool {
    self.state().has_double_jumped
}
```

In the Animation Blueprint's **Event Graph**, cast **Try Get Pawn Owner** to the character's class, call **Has Double Jumped** on it, and store the result in a variable the **AnimGraph** uses.

## Use case: the Animation Blueprint's logic in Rust

**The goal:** the logic of an Animation Blueprint's **Event Graph** (reading the owner, easing values toward their targets) in Rust, and the **AnimGraph** left to the Animation Blueprint, which reads the values the Rust class keeps.

The Rust class is the Animation Blueprint's parent, an `AnimInstance`:

```rust
use bindings::engine::{AnimInstance, AnimInstanceExt, SkeletalMeshComponent};

#[uclass(parent = SkeletalMeshComponent)]
pub struct MannequinsXR {
    #[uproperty(EditAnywhere, BlueprintReadWrite, category = "Hand", default = false)]
    b_mirror: bool,
}

#[uclass(parent = AnimInstance)]
pub struct HandAnimInstance {
    #[uproperty(BlueprintReadWrite, category = "Pose", default = 0.0)]
    pose_alpha_point: f32,

    #[uproperty(BlueprintReadOnly, category = "Pose", default = 0.0)]
    current_pose_alpha_point: f64,

    #[uproperty(BlueprintReadOnly, category = "Pose", default = false)]
    b_mirror: bool,
}

#[uclass_impl]
impl HandAnimInstance {
    #[ufunction(Override)]
    fn blueprint_initialize_animation(&mut self) {
        let mirror = self
            .as_ref()
            .checked()
            .ok()
            .and_then(|anim| MannequinsXR::from_obj(anim.get_owning_component()).ok())
            .is_some_and(|hand| hand.b_mirror());

        self.set_b_mirror(mirror);
    }

    #[ufunction(Override)]
    fn blueprint_update_animation(&mut self, delta_time_x: f32) {
        self.set_current_pose_alpha_point(model::f_interp_to(
            self.current_pose_alpha_point(),
            f64::from(self.pose_alpha_point()),
            f64::from(delta_time_x),
            13.0,
        ));
    }
}
```

`blueprint_initialize_animation` runs once, when the mesh creates the instance; `blueprint_update_animation` every animation update, on the game thread, with the frame's time. What the AnimGraph reads are the Rust properties, `BlueprintReadOnly` or `BlueprintReadWrite`. The pawn sets the targets on the instance its hand mesh has:

```rust
let anim = hand.checked()?.get_anim_instance();

if let Ok(mut hand_anim) = HandAnimInstance::from_obj(anim) {
    hand_anim.set_pose_alpha_point(1.0);
}
```

### In the editor

1. Open the Animation Blueprint (`ABP_MannequinsXR`), and in the toolbar click **Class Settings**.
2. In the **Details** panel, set **Parent Class** to **HandAnimInstance**.
3. In the **My Blueprint** panel, **Variables** lists the Rust properties under **Pose**; drag **Current Pose Alpha Point** into the **AnimGraph** and connect it where the old variable was.
4. Delete the old variables and the **Event Graph** nodes the Rust class replaces, then click **Compile**.

> **Screenshot:** the AnimGraph of ABP_MannequinsXR, with Current Pose Alpha Point coming from the Rust parent. *(images/recipes/anim_rust_parent.png)*

The variable types follow the Blueprint's: a Blueprint **Float** that is double precision is `f64`, one stored in single precision is `f32`. When they match, the AnimGraph's nodes take the Rust property in place of a variable of the same name.

Template: `vr`.

## Use case: a mannequin that copies the player

**The goal:** a mannequin in the level that walks, runs, jumps and lands when the player does, as the Intro To Unreal template's copycat.

The `mannequin` feature's `CopycatAnimInstance` reads player 0's character every animation update and keeps what an AnimGraph needs:

```rust
#[ufunction(Override)]
fn blueprint_update_animation(&mut self, _delta_time_x: f32) {
    if !self.character().is_valid() || !self.movement_component().is_valid() {
        self.find_character();
    }

    let (Ok(anim), Ok(character), Ok(movement)) = (
        self.as_ref().checked(),
        self.character().checked(),
        self.movement_component().checked(),
    ) else {
        return;
    };

    let velocity = movement.get_velocity();
    let ground_speed = ground_speed(velocity.to_dvec3());
    let direction = f64::from(anim.calculate_direction(&velocity, &character.k2_get_actor_rotation()));

    self.set_ground_speed(ground_speed);
    self.set_direction(clamp_direction(direction, movement.get_orient_rotation_to_movement()));
    self.set_should_move(should_move(ground_speed, movement.get_current_acceleration().to_dvec3()));
    self.set_is_falling(movement.is_falling());
}
```

`ground_speed`, `should_move` and `clamp_direction` are plain functions, tested with `cargo test`: the speed ignores the vertical axis, the mannequin moves only above 0.01 cm/s and while the character accelerates, and the direction stays within 45 degrees when the character turns toward its movement. `ABP_Copycat` is a child of `CopycatAnimInstance` whose AnimGraph blends idle, walk and run by **Ground Speed** and switches to a jump on **Is Falling**. Setting the instance's **Source Character** copies another character than the player.

Any skeletal mesh of the `SK_Mannequin` skeleton takes it: Manny, Quinn, or a mannequin of your own on that skeleton.

## Use case: a hand that reaches for a target

**The goal:** a mannequin plays its idle and, when a target comes within arm's length, its right hand goes to it.

The animation plays on a hidden skeletal mesh; every tick, a poseable mesh copies that pose and bends the arm with a two-bone IK solved in Rust:

```rust
let pose = self.pose()?.checked()?;
pose.copy_pose_from_skeletal_component(self.body()?);

let chain = Chain {
    root: location("upperarm_r"),
    joint: location("lowerarm_r"),
    end: location("hand_r"),
};

let solved = ik::reach(chain, target, down);

for (bone, rotation, location) in [
    ("upperarm_r", solved.root_delta * rotation("upperarm_r"), chain.root),
    ("lowerarm_r", solved.joint_delta * solved.root_delta * rotation("lowerarm_r"), solved.joint),
    ("hand_r", rotation("hand_r"), solved.end),
] {
    pose.set_bone_transform_by_name(
        FName::new(bone).handle(),
        &FTransform::from_transform(Transform::new(rotation, location, DVec3::ONE)),
        EBoneSpaces::WorldSpace,
    );
}
```

`ik::reach` keeps both bone lengths, places the elbow in the plane of its current bend (nudged by a pole, here down), and returns the rotations that carry each bone onto the solution. Each bone gets its whole world transform: `set_bone_rotation_by_name` would read the bone's location from before its parent moved, and pull the elbow off the arm.

The hidden mesh keeps animating with **Visibility Based Anim Tick Option** set to **Always Tick Pose and Refresh Bones**, and plays an animation asset without an Animation Blueprint:

```rust
body.play_animation(self.idle_animation().upcast_to(), true);
```

In the editor: place a **ReachingMannequin** and a **ReachTarget** from the **Place Actors** panel, select the mannequin and set its **Target** to the target, then move the target around it while playing.

> **Screenshot:** the reaching mannequin with its right hand on the target sphere. *(images/recipes/reach_target.png)*
