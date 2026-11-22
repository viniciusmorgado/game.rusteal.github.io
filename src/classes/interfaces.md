# Interfaces

Two different tools answer "different classes that respond to the same call": Unreal interfaces, which Blueprints understand, and Rust traits, which only Rust code sees. A `#[uinterface]` trait is both.

## Use case: an interface declared in Rust

`#[uinterface]` exists since 0.8.0. A property may hold an object implementing one: `ScriptInterface<dyn Interactable>` (0.10).

**The goal:** doors, chests and levers the player interacts with, some written in Rust and some in Blueprints, and one call that reaches any of them.

### In Rust

```rust
use rusteal_runtime::{uclass, uclass_impl, uinterface};

#[uinterface]
pub trait Interactable {
    fn interact(&mut self, instigator: UObjectRef<Actor>) -> bool;
    fn highlight(&mut self, on: bool);
}
```

`#[uinterface]` makes the trait an Unreal interface, `Interactable`, whose functions are Blueprint events: **Interact** returns a Boolean, **Highlight** returns nothing. Its functions take `&self` or `&mut self`, then what a `#[ufunction]` takes (see [Functions](functions.md)), and return what it returns. They have no body: each class implementing the interface writes its own.

A Rust class implements it with `#[uclass_impl]` on the trait's implementation. Its methods are the interface's functions, with no `#[ufunction]`:

```rust
#[uclass(parent = Actor)]
pub struct Door {
    #[uproperty(EditAnywhere)]
    b_locked: bool,
    #[uproperty(VisibleAnywhere)]
    b_open: bool,
}

#[uclass_impl]
impl Interactable for Door {
    fn interact(&mut self, _instigator: UObjectRef<Actor>) -> bool {
        if self.b_locked() {
            return false;
        }

        self.set_b_open(!self.b_open());
        true
    }

    fn highlight(&mut self, on: bool) {
        log::info!("Door highlighted: {on}");
    }
}
```

The class keeps its other `#[uclass_impl]` block for its own functions and events.

### In the editor

A Blueprint implements it as it implements a Blueprint Interface:

1. Open the Blueprint (a chest, say), and click **Class Settings** in the toolbar.
2. In the **Details** panel, under **Interfaces**, click **Add** next to **Implemented Interfaces** and pick **Interactable**.
3. In the **My Blueprint** panel, under **Interfaces**, double-click **Interact**: a function graph opens, since it returns a value. Connect its logic to the **Return Node**.
4. Right-click the **Event Graph**, search for **Event Highlight**, and place it: functions with no return value are events.
5. Click **Compile**, then **Save**.

> **Screenshot:** a chest Blueprint's Class Settings, with Interactable under Implemented Interfaces and the Interact function in My Blueprint. *(images/classes/uinterface_bp.png)*

A Blueprint child of `Door` has the interface too: double-clicking **Interact** in its **My Blueprint** panel overrides `Door`'s, as a C++ `BlueprintNativeEvent` is overridden, and **Add call to parent function** in the graph's context menu runs the Rust one.

A Rust class that leaves the functions to its Blueprint children lists the interface instead of implementing it:

```rust
#[uclass(parent = Actor, implements = [Interactable])]
pub struct Lever {}
```

### Calling it

From Rust, on any object, whichever language implements it:

```rust
if let Ok(mut target) = hit_actor.interface::<dyn Interactable>() {
    target.interact(self.as_ref().upcast_to());
}
```

`interface::<dyn Interactable>()` turns an object into the interface, as C++'s `Cast<IInteractable>`: an error when its class does not implement it. `implements::<dyn Interactable>()` only tells. The `ScriptInterface<dyn Interactable>` it gives (C++'s `TScriptInterface`) calls the object's own implementation, Rust or Blueprint; a function the object's class does not implement returns the default value (`false`, `0`, a null reference), as a Blueprint's interface message does.

From a Blueprint: the **Interact (Message)** node on any actor, or **Interact** on an **Interactable** reference; **Does Implement Interface** with **Interactable** finds out whether an actor has it.

> **Screenshot:** a Blueprint graph calling Interact (Message) on a line trace's Hit Actor. *(images/classes/uinterface_message.png)*

### Notes

- The interface is `/Script/Rusteal.Interactable`, a plugin's `/Script/<Plugin>.Interactable`. The classes of another library (the game, for a plugin's interface) implement it by that path, as below.
- Changing it during a hot reload keeps the interface and gives it the new functions; the Blueprints implementing or calling it are recompiled, as when a Blueprint Interface changes. See [Hot reload](../iteration/hot_reload.md).

## Use case: implementing a Blueprint Interface

**The goal:** a Rust door that a Blueprint can **Use** through a Blueprint Interface, like any Blueprint door.

### In the editor

1. In the **Content Browser**, right-click an empty area and choose **Blueprint > Blueprint Interface**.
2. Name it `BPI_Usable` and open it.
3. In the **My Blueprint** panel, rename the first function to `Use`.
4. With `Use` selected, in the **Details** panel, add an **Input** named `Instigator` of type **Actor** (**Object Reference**), and an **Output** named `ReturnValue` of type **Boolean**.
5. Click **Compile**, then **Save**.

> **Screenshot:** BPI_Usable open, with the Use function's Instigator input and ReturnValue output in the Details panel. *(images/classes/bpi_usable.png)*

### In Rust

```rust
#[uclass(parent = Actor, implements = ["/Game/Interaction/BPI_Usable.BPI_Usable_C"])]
pub struct Door {
    #[uproperty(EditAnywhere)]
    b_locked: bool,
}

#[uclass_impl]
impl Door {
    #[ufunction]
    fn r#use(&mut self, _instigator: UObjectRef<Actor>) -> bool {
        !self.b_locked()
    }
}
```

`implements` lists interfaces by class path:

- a Blueprint Interface asset: its path with `_C` after the asset name, `/Game/Interaction/BPI_Usable.BPI_Usable_C` for `BPI_Usable` in `Content/Interaction/`;
- a C++ interface: `/Script/<Module>.<Interface>` without the `U`, `/Script/Engine.NavAgentInterface`.

Each interface function is a `#[ufunction]` named as the interface's function (in snake_case, or with `name`), taking the same parameters in the same order and returning its return value.

### Calling it

A Blueprint calls it with the **Use (Message)** node on any actor, and **Does Implement Interface** finds out whether an actor has it. C++ calls it with `IUsable::Execute_Use`. They reach the Rust class as they reach a Blueprint that implements the interface.

### Notes

- The interface loads once the engine is initialized, so the class implements it from then on.
- An interface whose functions are not Blueprint events (one marked `CannotImplementInterfaceInBlueprint`) can only be implemented in C++, and is refused, as Blueprints refuse it.
- To declare the interface in Rust instead, see the first use case.

## Use case: Rust traits between Rust classes

**The goal:** several Rust classes take damage, and the code that deals it does not care which one it hit.

When only Rust code calls it, a Rust trait is the lighter tool. The templates declare the trait, implement it on each class, and turn an actor into the trait with `from_obj`:

```rust
pub trait Damageable {
    fn apply_damage(&mut self, damage: f32, causer: UObjectRef<Actor>);
}

impl Damageable for Enemy {
    fn apply_damage(&mut self, damage: f32, _causer: UObjectRef<Actor>) {
        self.set_current_hp(self.current_hp() - damage);
    }
}

impl Damageable for DamageableBox {
    fn apply_damage(&mut self, damage: f32, _causer: UObjectRef<Actor>) {
    }
}

pub fn damageable(actor: UObjectRef<Actor>) -> Option<Box<dyn Damageable>> {
    if let Ok(enemy) = Enemy::from_obj(actor) {
        Some(Box::new(enemy))
    } else if let Ok(damageable_box) = DamageableBox::from_obj(actor) {
        Some(Box::new(damageable_box))
    } else {
        None
    }
}
```

The caller:

```rust
if let Some(mut target) = damageable(other_actor) {
    target.apply_damage(self.damage(), self.as_ref().upcast_to());
}
```

A trait is invisible to Blueprints: a Blueprint class cannot implement it, and a Blueprint cannot call it. When a Blueprint must take part, put `#[uinterface]` on the trait: the classes implement it with `#[uclass_impl]`, and `interface::<dyn Damageable>()` replaces the `damageable` function.
