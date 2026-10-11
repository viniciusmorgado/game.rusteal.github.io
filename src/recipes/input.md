# Character movement and input

**The goal:** a character that walks, looks around and jumps with the keyboard and mouse, a gamepad or touch, with Enhanced Input.

Input in Rusteal works as in the engine's C++ templates:

1. the player controller adds the **Input Mapping Contexts** when the game starts;
2. the pawn binds each **Input Action** to one of its functions once its input component exists;
3. a Blueprint child of each assigns the assets.

## The player controller

```rust
use bindings::engine::{ControllerExt, PlayerController};
use bindings::enhanced_input::{EnhancedInputLocalPlayerSubsystemExt, InputMappingContext};
use bindings::prelude::*;
use rusteal_runtime::runtime::{OwnedStruct, RustealResult, UObjectRef, UeArray };
use rusteal_runtime::{uclass, uclass_impl};

#[uclass(parent = PlayerController)]
pub struct MyPlayerController {
    #[uproperty(EditAnywhere)]
    default_mapping_contexts: UeArray<UObjectRef<InputMappingContext>>,
}

#[uclass_impl]
impl MyPlayerController {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        if let Err(e) = self.add_mapping_contexts() {
            log::warn!("[MyGame] input mapping contexts failed: {e}");
        }
    }
}

impl MyPlayerController {
    fn add_mapping_contexts(&self) -> RustealResult<()> {
        let me: UObjectRef<PlayerController> = self.as_ref();

        if !me.checked()?.is_local_controller() {
            return Ok(());
        }

        let subsystem = enhanced_input_subsystem(me)?.checked()?;

        for context in self.default_mapping_contexts().to_vec()? {
            subsystem.add_mapping_context(context, 0, &OwnedStruct::new());
        }

        Ok(())
    }
}
```

`enhanced_input_subsystem(controller)` is the controller's **Enhanced Input Local Player Subsystem**, where mapping contexts are added. A player controller binds its own actions in `receive_begin_play` too, which is where C++ does it (`SetupInputComponent`).

## The character

```rust
use bindings::engine::{
    Character, CharacterExt, CharacterMovementComponentExt, ControllerExt, PawnExt
};
use bindings::enhanced_input::{ETriggerEvent, FInputActionValue, InputAction};
use bindings::prelude::*;
use rusteal_runtime::runtime::{RustealResult, UObjectRef, UStructRef };
use rusteal_runtime::{uclass, uclass_impl};

#[uclass(parent = Character)]
pub struct MyCharacter {
    #[uproperty(EditAnywhere, category = "Input")]
    jump_action: UObjectRef<InputAction>,

    #[uproperty(EditAnywhere, category = "Input")]
    move_action: UObjectRef<InputAction>,

    #[uproperty(EditAnywhere, category = "Input")]
    look_action: UObjectRef<InputAction>,
}

#[uclass_impl]
impl MyCharacter {
    #[ufunction(Override)]
    fn receive_restarted(&mut self) {
        if let Err(e) = self.bind_input() {
            log::warn!("[MyGame] input binding failed: {e}");
        }
    }

    #[ufunction]
    fn r#move(&mut self, value: UStructRef<FInputActionValue>) {
        let axis = value.axis2d();

        let Ok(me) = self.as_ref().checked() else {
            return;
        };

        let Ok(controller) = me.get_controller().checked() else {
            return;
        };

        let yaw = controller.get_control_rotation().to_rotator().yaw.to_radians();
        let forward = DVec3::new(yaw.cos(), yaw.sin(), 0.0);
        let right = DVec3::new(-yaw.sin(), yaw.cos(), 0.0);

        me.add_movement_input(&FVector::from_dvec3(forward), Some(axis.y as f32), Some(false));
        me.add_movement_input(&FVector::from_dvec3(right), Some(axis.x as f32), Some(false));
    }

    #[ufunction]
    fn look(&mut self, value: UStructRef<FInputActionValue>) {
        let axis = value.axis2d();

        if let Ok(me) = self.as_ref().checked() {
            me.add_controller_yaw_input(axis.x as f32);
            me.add_controller_pitch_input(axis.y as f32);
        }
    }
}

impl MyCharacter {
    fn bind_input(&self) -> RustealResult<()> {
        let me = self.as_ref();
        let pawn = me.checked()?;

        if !pawn.is_player_controlled() || !pawn.is_locally_controlled() {
            return Ok(());
        }

        bind_action(&me, self.jump_action(), ETriggerEvent::Started, "Jump")?;
        bind_action(&me, self.jump_action(), ETriggerEvent::Completed, "StopJumping")?;
        bind_action(&me, self.move_action(), ETriggerEvent::Triggered, "Move")?;
        bind_action(&me, self.look_action(), ETriggerEvent::Triggered, "Look")?;

        Ok(())
    }
}
```

- `receive_restarted` is where a pawn binds input: it runs when a controller possesses the pawn, once the input component exists. It replaces C++'s `SetupPlayerInputComponent`.
- `bind_action(&pawn, action, trigger, "Function")` binds an action's trigger event to a function by its Unreal name. Engine functions work as well: `"Jump"` and `"StopJumping"` are `ACharacter`'s.
- The value of an action reads as `get_bool()`, `axis1d()`, `axis2d()` or `axis3d()`, by its **Value Type**.

### What a bound function can take

The function takes nothing, or a leading part of these, in order:

| Parameter | Type | What it is |
|---|---|---|
| value | `UStructRef<FInputActionValue>` | the action's value |
| elapsed seconds | `f32` | how long the action has been evaluated |
| triggered seconds | `f32` | how long it has been triggered |
| source action | `UObjectRef<InputAction>` | the action itself |

```rust
#[ufunction]
fn charge(&mut self, _value: UStructRef<FInputActionValue>, elapsed_seconds: f32) {
    let charged = elapsed_seconds >= 1.5;
}
```

## In the editor

The engine templates already have the input actions (`IA_Move`, `IA_Look`, `IA_Jump`) and a mapping context (`IMC_Default`). For a project without them:

1. In the **Content Browser**, right-click and choose **Input > Input Action**. Create `IA_Move` and, in its **Details**, set **Value Type** to **Axis2D (Vector2D)**. Do the same for `IA_Look`, and create `IA_Jump` with **Digital (bool)**.
2. Right-click and choose **Input > Input Mapping Context**. Name it `IMC_Default`, open it, and add a mapping for each action with the **+** next to **Mappings**: `IA_Jump` to **Space Bar**, `IA_Look` to **Mouse XY 2D-Axis**, `IA_Move` to **W**, **S**, **A** and **D** with the **Swizzle Input Axis Values** and **Negate** modifiers as the engine templates do.
3. Create a Blueprint child of `MyPlayerController` (see [Blueprint children](../classes/blueprint_child.md)), and in its **Class Defaults** add `IMC_Default` to **Default Mapping Contexts**.
4. Create a Blueprint child of `MyCharacter`, and in its **Class Defaults**, under **Input**, set **Jump Action**, **Move Action** and **Look Action**.
5. Set both Blueprints in the game mode (see [Game flow](game_flow.md)).

> **Screenshot:** the Class Defaults of the character Blueprint with the three input actions assigned. *(images/recipes/input_actions_assigned.png)*

> **Screenshot:** IMC_Default with the W, A, S, D mappings and their modifiers. *(images/recipes/imc_default.png)*

If nothing moves, the Output Log usually says why: no mapping contexts added, or a function name that does not exist.

## Touch controls

The templates add a virtual joystick widget on touch screens:

```rust
use rusteal_runtime::runtime::input::should_display_touch_interface;

if should_display_touch_interface() {
    let widget = create_widget_of_class(&me, self.mobile_controls_widget_class())?;
    widget.checked()?.add_to_player_screen(Some(0));
}
```

`should_display_touch_interface()` is the engine's own check for a touch screen. The widget class is a `SubclassOf<UserWidget>` property, which the template's player controller Blueprint sets to the engine template's mobile controls widget.

## Movement settings

The movement itself is the character movement component's, tuned in [Class defaults](../classes/class_defaults.md):

```rust
let movement = me.get_character_movement().checked()?;
movement.set_orient_rotation_to_movement(true);
movement.set_max_walk_speed(500.0);
movement.set_jump_z_velocity(500.0);
movement.set_air_control(0.35);
```
