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
