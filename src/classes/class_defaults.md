# Class defaults

**The goal:** set what a C++ constructor sets: inherited properties, component settings, object defaults, classes to use.

```rust
#[uclass_impl]
impl Hero {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        me.set_use_controller_rotation_yaw(false);

        let movement = me.get_character_movement().checked()?;
        movement.set_orient_rotation_to_movement(true);
        movement.set_max_walk_speed(500.0);
        movement.set_jump_z_velocity(700.0);

        let boom = self.camera_boom()?.checked()?;
        boom.set_target_arm_length(400.0);
        boom.set_use_pawn_control_rotation(true);

        Ok(())
    }
}
```

A method marked `#[class_defaults]` in the `#[uclass_impl]` block runs once, on the class default object, when the class is created. Every instance and every Blueprint child starts from that object, so what it sets is the class's default. It returns `RustealResult<()>`, so `?` works, or nothing.

## What goes there

| You want | In class defaults |
|---|---|
| A default for an inherited property | `me.set_use_controller_rotation_yaw(false)` |
| A component's settings | `self.mesh()?.checked()?.set_simulate_physics(true)` |
| A default for one of your properties that `default =` cannot hold | `self.set_ragdoll_profile(FName::new("Ragdoll"))` |
| The class an engine object uses | `me.set_player_camera_manager_class(SubclassOf::<MyCameraManager>::base().upcast_to())` |
| An AI controller for a pawn | `me.set_ai_controller_class(SubclassOf::<MyAIController>::base().upcast_to())` |
| A subsystem that ticks | `self.as_ref().checked()?.set_wants_tick(true)` |

An actor needs nothing to tick: overriding `receive_tick` turns its tick on.

## What does not go there

Class defaults run on the class default object, not on an instance in a level, and before any game starts. Do not spawn actors, look for other actors or read the world there: do that in `receive_begin_play`.

## Blueprint children win

A Blueprint child that changes a value in its **Class Defaults** keeps its value: class defaults set the starting point, and the Blueprint's changes are saved on top of it. When a change in `class_defaults` does not show on a Blueprint child, check whether that Blueprint overrides the same value (a value changed from its default shows a **Reset to Default** arrow next to it in the **Details** panel).

## A Rust parent's defaults

A class whose parent is a Rust class starts from the parent's defaults: the parent's `class_defaults` runs first, then the child's.
