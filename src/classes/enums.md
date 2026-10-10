# Enums

**The goal:** an enum declared in Rust that Unreal understands: a property a designer picks from a dropdown, a function parameter a Blueprint passes, a value Blueprints switch on.

```rust
use rusteal_runtime::uenum;

#[uenum]
pub enum EGrabType {
    None,
    Snap,
    Custom = 3,
    Free,
}
```

A `#[uenum]` is Unreal's `UENUM(BlueprintType)`. Its variants carry no data, and their values go up to 254, as a Blueprint enum's `uint8` does (Unreal adds the `_MAX` entry after the last). Values follow Rust's rules: the first is 0, each next one adds 1, and `= 3` sets one. The macro sets the representation, `u8`, and derives `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash` and `Debug`, so the enum is a plain Rust value too:

```rust
match self.grab_type() {
    EGrabType::Free | EGrabType::Snap => self.attach_to(controller),
    EGrabType::Custom => self.set_b_is_held(true),
    EGrabType::None => {}
}
```

## Use case: a grab type designers choose

**The goal:** a grab component whose kind of grab each object sets in its Blueprint, from a dropdown.

```rust
#[uclass(parent = SceneComponent)]
pub struct GrabComponent {
    #[uproperty(EditAnywhere, BlueprintReadWrite, category = "Grab", default = EGrabType::Free)]
    grab_type: EGrabType,
}
```

`default` takes a variant, as it does for engine enums. A Rust class that places the component can choose it in its class defaults:

```rust
#[class_defaults]
fn class_defaults(&mut self) -> RustealResult<()> {
    GrabComponent::from_obj(self.grab_component_snap()?)?.set_grab_type(EGrabType::Snap);
    Ok(())
}
```

### In the editor

1. Open the Blueprint that has the component (a `BP_Pistol` child of the Rust pistol).
2. In the **Components** panel, select **GrabComponentSnap**.
3. In the **Details** panel, under **Grab**, **Grab Type** lists **None**, **Snap**, **Custom** and **Free**.

> **Screenshot:** the Details panel of a grab component, with the Grab Type dropdown open. *(images/classes/uenum_dropdown.png)*

In a Blueprint graph, the enum has the usual nodes: **Switch on EGrabType**, **== (Equal)**, **Literal enum EGrabType**.

## Enums in functions

`#[uenum]`s and engine enums are `#[ufunction]` parameters and return values:

```rust
#[ufunction(BlueprintPure)]
pub fn get_held_by_hand(&self) -> EControllerHand {
    match model::hand_of_motion_source(&self.motion_source_name()) {
        model::Hand::Left => EControllerHand::Left,
        model::Hand::Right => EControllerHand::Right,
    }
}

#[ufunction(BlueprintCallable)]
pub fn set_vr_spectator_mode(&mut self, new_vr_spectator_mode: EVRSpectatorMode) {
    self.set_spectator_mode(new_vr_spectator_mode);
}
```

The enum must be a one-byte one, as every Blueprint enum is; an engine enum stored on more bytes is refused when the crate compiles.

## Notes

- The enum's Unreal name is the Rust name. Its entries are `EGrabType::Snap` to C++ and ini files, **Snap** in the editor.
- An enum created by a hot reload keeps its first values: adding a variant takes an editor restart, as changing a `#[ustruct]` does.
- An enum asset made in the editor (**Blueprint > Enumeration**) is not a Rust type. To use one from Rust, declare it again with `#[uenum]` and point the Blueprints at the Rust one.
