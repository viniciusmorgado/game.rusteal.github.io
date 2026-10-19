# Object references

Every Unreal object Rust code touches, an actor, a component, an asset, a class's default object, is reached through a reference. Unreal owns the objects and destroys them when its garbage collector decides; Rust code holds references to them and checks them before use.

## UObjectRef

```rust
let target: UObjectRef<Actor> = self.current_target();
```

`UObjectRef<T>` is a handle to an object of class `T`: eight bytes, `Copy`, cheap to pass around and to store. It does not keep the object alive, and the object may be destroyed while you hold it.

| Method | What it does |
|---|---|
| `is_valid()` | Whether the handle points to an object (it can still be pending destruction). |
| `checked()` | Verifies the object is alive and gives access to its functions, or an `ObjectDestroyed` error. |
| `cast::<U>()` | The same object as class `U`, checked at runtime, or an `InvalidCast` error naming the object and the class expected. |
| `upcast_to::<U>()` | The same object as an ancestor class `U`, checked at compile time. |
| `is_a::<U>()` | Whether the object is of class `U` or derives from it. |
| `get_name()` | The object's name. |
| `pin()` | A `Pinned<T>` that keeps the object alive. |
| `UObjectRef::null()` | A handle to nothing, also its `Default`. |

## Calling functions: checked

Engine functions are called on a checked reference:

```rust
let me = self.as_ref().checked()?;
let location = me.k2_get_actor_location().to_dvec3();
me.k2_destroy_actor();
```

`checked()` verifies the object is still alive once, and the calls after it go straight to the engine. A call on a destroyed object would crash the engine in C++; in Rust, `checked()` returns an error first, and the code decides what to do:

```rust
let Ok(target) = self.current_target().checked() else {
    return;
};
```

## Casting

```rust
if let Ok(character) = other_actor.cast::<Character>() {
}

let as_object: UObjectRef<Object> = actor.upcast_to::<Object>();
```

`cast` is C++'s `Cast<T>`: a runtime check. `upcast_to` is free and cannot fail, because the compiler knows `Character` is an `Actor`. The engine functions that take a world context or a base class (`UObject* WorldContextObject`, `AActor*`) take the upcast:

```rust
let world_context = self.as_ref().upcast_to();
let player = GameplayStatics::get_player_pawn(world_context, 0);
```

An object's class and outer come typed as well: `actor.class()?` is a `SubclassOf<T>` of the reference's own type, and `component.outer::<Actor>()?` is the outer, checked to be an actor.

For your own Rust classes, `MyClass::from_obj(reference)` does the cast and gives the Rust class with its getters and methods. See [Declaring a class](../classes/uclass.md).

## Keeping an object alive

The garbage collector keeps an object alive while something it knows about references it. A `UObjectRef` in a Rust-only field or a local variable is invisible to it. To keep an object:

- **Hold it in a `#[uproperty]`.** The usual way, and what C++ does with `UPROPERTY()` pointers.

  ```rust
  #[uproperty]
  spawned_weapon: UObjectRef<Weapon>,
  ```

- **Pin it.** `Pinned<T>` is a garbage collection root: the object lives until the `Pinned` is dropped. For objects Rust code alone keeps, outside any class.

  ```rust
  let pinned: Pinned<Actor> = actor.pin()?;
  let location = pinned.k2_get_actor_location();
  ```

- **Watch it.** `TWeakObjectPtr<T>` does not keep the object, and tells you when it is gone:

  ```rust
  let weak = TWeakObjectPtr::from_ref(&actor);
  if let Some(actor) = weak.get() {
  }
  ```

## Engine soft and weak references

The engine's own properties and functions use the same types as your classes:

| Engine C++ | Rust | Reading it |
|---|---|---|
| `TSoftObjectPtr<T>` | `SoftObjectRef<T>` | `path()`; `get()` gives the object if it is loaded, `load_synchronous()` loads it; `checked()` gives the loaded object or an error |
| `TSoftClassPtr<T>` | `SoftClassRef<T>` | `path()`; `get()` gives the class as a `SubclassOf<T>` if it is loaded, `load_synchronous()` loads it |
| `TWeakObjectPtr<T>` | `TWeakObjectPtr<T>` | `get()` gives `Some(object)` while it exists, `None` once it is destroyed; `checked()` gives it or an error |

A soft reference keeps its path even when nothing is loaded, so functions that take one take a path:

```rust
let material = KismetSystemLibrary::load_asset_blocking(&SoftObjectRef::new(
    "/Engine/EngineMaterials/DefaultMaterial.DefaultMaterial",
));
```

A hit result's component is weak: the component may be destroyed while the result is kept.

```rust
if let Ok(component) = hit.get_component().checked() {
    let actor = component.get_owner();
}
```

## Engine interfaces

An engine function or property that takes an interface (`TScriptInterface<IBlendableInterface>` in C++) takes a `UObjectRef` of the interface class; `cast` gives one from any object that implements it, natively or in a Blueprint:

```rust
let blendable = material.cast::<BlendableInterface>()?;
volume.add_or_update_blendable(blendable, Some(1.0));
```

A function that takes any interface (`FScriptInterface`, such as `KismetSystemLibrary::conv_interface_to_object`) takes a `UObjectRef<Object>`.

## Classes: SubclassOf

A class is not an object reference. `SubclassOf<T>` is Unreal's `TSubclassOf<T>`, a class that is `T` or derives from it:

```rust
#[uproperty(EditAnywhere)]
projectile_class: SubclassOf<Projectile>,
```

| Expression | What it is |
|---|---|
| `SubclassOf::<Projectile>::base()` | The `Projectile` class itself. |
| `class.upcast_to::<Actor>()` | The same class, as a `SubclassOf<Actor>`, for engine functions that take any actor class. |
| `class.is_null()` | Whether no class is set (a Blueprint child that did not fill the property). |

Engine functions that take a class filter (`get_all_actors_of_class`, `get_overlapping_actors`) take a `SubclassOf`:

```rust
let units = GameplayStatics::get_all_actors_of_class(
    self.as_ref().upcast_to(),
    SubclassOf::<Unit>::base().upcast_to(),
);
```

## Threads

References are `Send`, so a background thread can carry them, but Unreal objects belong to the game thread: call the engine only from there. See [Work off the game thread](../systems/tasks.md).
