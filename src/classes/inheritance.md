# Rust classes deriving from Rust classes

**The goal:** a class that extends another Rust class: a horror character built on the first-person character, an elite enemy built on the enemy.

```rust
use crate::character::FirstPersonCharacter;

#[uclass(parent = FirstPersonCharacter)]
pub struct HorrorCharacter {
    #[component(attach = "first_person_camera_component", name = "SpotLight")]
    spot_light: SpotLightComponent,

    #[uproperty(EditAnywhere, category = "Sprint", default = 600.0)]
    sprint_speed: f32,
}
```

The child has the parent's properties, components and functions, and adds its own:

- its components can attach to the parent's by their field names;
- its class defaults start from the parent's: the parent's `class_defaults` runs first;
- Blueprints see one class with everything.

## Calling the parent

An event both classes override runs the child's code. To run the parent's too, as C++ calls `Super::`, reach the parent class with `from_obj` and call it:

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    if let Ok(mut parent) = FirstPersonCharacter::from_obj(self.as_ref()) {
        parent.receive_begin_play();
    }
}
```

The parent's methods must be visible to the child's module: declare the ones the child calls `pub(crate)`.

```rust
#[ufunction(Override)]
pub(crate) fn receive_begin_play(&mut self) {
}
```

The same reaches the parent's components and Rust methods. The templates keep a helper for it:

```rust
fn base(&self) -> RustealResult<FirstPersonCharacter> {
    FirstPersonCharacter::from_obj(self.as_ref())
}

let camera = self.base()?.first_person_camera_component()?.checked()?;
```

## Replacing a parent's function

A function the child declares again with the same name replaces the parent's, for Blueprints and for the engine:

```rust
#[ufunction(BlueprintCallable)]
fn do_move(&mut self, right: f32, forward: f32) {
    if !self.is_dead()
        && let Ok(mut parent) = FirstPersonCharacter::from_obj(self.as_ref())
    {
        parent.do_move(right, forward);
    }
}
```

## Hot reload

When a parent class changes shape, its Rust children are created again with it, and their objects move to the new classes. See [Hot reload](../iteration/hot_reload.md).
