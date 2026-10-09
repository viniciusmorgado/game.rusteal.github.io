# Interfaces

Two different tools answer "different classes that respond to the same call": Unreal interfaces, which Blueprints understand, and Rust traits, which only Rust code sees.

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
- Declaring a new interface from Rust is not possible yet: create it as a Blueprint Interface.

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

A trait is invisible to Blueprints: a Blueprint class cannot implement it, and a Blueprint cannot call it. When a Blueprint must take part, use an Unreal interface.
