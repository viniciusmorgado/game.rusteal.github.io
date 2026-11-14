# Object references

Every Unreal object Rust code touches, an actor, a component, an asset, a class's default object, is reached through a reference. Unreal owns the objects and destroys them when its garbage collector decides; Rust code holds references to them and checks them before use.

## UObjectRef

```rust
let target: UObjectRef<Actor> = self.current_target();
```

`UObjectRef<'a, T>` is a reference to an object of class `T`: eight bytes, `Copy`, cheap to pass around. It holds the object's index and serial number in the engine's object array, as `TWeakObjectPtr` does, never its address. So it never reads freed memory, and an object the engine allocates in the place of a destroyed one is never taken for it: once the object is gone, every check on the reference says so.

It does not keep the object alive, and the object may be destroyed while you hold it.

| Method | What it does |
|---|---|
| `is_valid()` | Whether the object still exists and is not being destroyed. |
| `checked()` | Verifies the object is alive and gives access to its functions, or an `ObjectDestroyed` error. |
| `cast::<U>()` | The same object as class `U`, checked at runtime, or an `InvalidCast` error naming the object and the class expected. |
| `upcast_to::<U>()` | The same object as an ancestor class `U`, checked at compile time. |
| `is_a::<U>()` | Whether the object is of class `U` or derives from it. |
| `get_name()` | The object's name. |
| `pin()` | A `Pinned<T>` that keeps the object alive. |
| `with_scope_of(&other)` | The same reference, for as long as `other` may live (see below). |
| `UObjectRef::null()` | A reference to nothing, also its `Default`. |

### A reference lives as long as the call that gave it

The `'a` of `UObjectRef<'a, T>` is the call that handed the reference out: the engine event your `#[ufunction]` runs for, the callback a delegate or a timer calls, the engine function that returned it. The reference cannot outlive it. You rarely write `'a`: in a function's parameters the compiler fills it in, and a method returning an object takes it from `&self`.

```rust
#[ufunction(Override)]
fn receive_actor_begin_overlap(&mut self, other: UObjectRef<Actor>) {
    let Ok(other) = other.checked() else {
        return;
    };

    let owner = other.get_owner(); // lives as long as `other`
}
```

What it forbids is keeping a reference past its call, where the garbage collector cannot see it and the object can be destroyed: a struct field, a static, a closure or a future that runs in a later frame. Those are compile errors. Keep an object one of the ways [below](#keeping-an-object), and let a closure or a task carry a `TWeakObjectPtr` it resolves when it runs:

```rust
let target = TWeakObjectPtr::from_ref(&target);

timer::set_timer(&me, Duration::from_secs(1), false, move || {
    if let Ok(target) = target.checked() {
        target.k2_destroy_actor();
    }
})?;
```

`method(self, Self::fire)` does the same for a method of your class: it finds the object again on every call, and does nothing once it is gone. A delegate whose parameters are objects binds a method with `add_method(self, Self::on_hit)`, which hands the method objects that live as long as each call (see [Delegates](delegates.md#binding-a-method)).

A function that returns an object it did not get from its parameters or from `self` names the lifetime it borrows, or re-scopes the object to one it has: `subsystem.hint_widget().with_scope_of(&me)` returns a property of a subsystem found for the call, for as long as `me` may live.

The same holds for what is read through a temporary value: the Rust class `from_obj` gives, a hit result, a weak pointer a getter returns. An object read from it lives as long as that value, so keep the value in a `let` while the object is used, or re-scope the object to the reference the value came from:

```rust
let component = hit.get_component();          // a TWeakObjectPtr, kept in a let
let actor = component.checked()?.get_owner();

let root = TestActor::from_obj(target)?.root()?.with_scope_of(&target);
```

An object found from a path, or made with no outer, borrows a `GameThread`, the proof that the code runs on the game thread:

```rust
let gt = GameThread::current()?;
let mesh = load_object::<StaticMesh>(&gt, "/Game/Meshes/Crate.Crate")?;
let save = GameplayStatics::create_save_game_object(&gt, SubclassOf::<MySave>::base().upcast());
```

Engine functions with no object parameter that return one take it first, the same way.

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

## Keeping an object

The garbage collector keeps an object alive while something it knows about references it. A `UObjectRef` is not one of those things, and the compiler keeps it out of the places that would need to be: a `#[uclass]` refuses a Rust-only field that holds one (inside an `Option`, a `Vec` or a map too), and the lifetime keeps it out of statics, closures and tasks. To keep an object:

- **Hold it in a `#[uproperty]`.** The usual way, and what C++ does with `UPROPERTY()` pointers: the class owns the object or must keep it, and the engine sees it.

  ```rust
  #[uproperty]
  spawned_weapon: UObjectRef<Weapon>,

  #[uproperty]
  squad: UeArray<UObjectRef<Unit>>,
  ```

- **Watch it.** `TWeakObjectPtr<T>` does not keep the object, and tells you when it is gone: for what the class only observes (a target, the player that triggered something, a selection). It is a plain value, kept in Rust-only fields, closures and tasks.

  ```rust
  target: TWeakObjectPtr<Actor>,
  selection: Vec<TWeakObjectPtr<Unit>>,

  if let Some(target) = self.target().get() {
  }
  ```

- **Pin it.** `Pinned<T>` keeps the object alive until the `Pinned` is dropped, for objects Rust code alone keeps, outside any class. Each `Pinned` (and each `clone()` of one) keeps it on its own, and an object destroyed explicitly (`k2_destroy_actor`) reads as gone through it.

  ```rust
  let pinned: Pinned<Actor> = actor.pin()?;
  let location = pinned.checked()?.k2_get_actor_location();
  ```

An engine struct Rust owns (`OwnedStruct<T>`, such as a hit result) keeps the objects it holds alive the same way a `#[uproperty]` does, while it exists.

## Engine soft and weak references

The engine's own properties and functions use the same types as your classes:

| Engine C++ | Rust | Reading it |
|---|---|---|
| `TSoftObjectPtr<T>` | `SoftObjectRef<T>` | `path()`; `get()` gives the object if it is loaded, `load_synchronous()` loads it; `checked()` gives the loaded object or an error. The object lives as long as the soft reference is borrowed: keep the soft reference in a `let` first (`let mesh = data.mesh(); let mesh = mesh.load_synchronous()?;`). `load_async()` gives a `TWeakObjectPtr`, because it is used after the `.await` |
| `TSoftClassPtr<T>` | `SoftClassRef<T>` | `path()`; `get()` gives the class as a `SubclassOf<T>` if it is loaded, `load_synchronous()` loads it |
| `TWeakObjectPtr<T>` | `TWeakObjectPtr<T>` | `get()` gives `Some(object)` while it exists, `None` once it is destroyed; `checked()` gives it or an error |

A soft reference keeps its path even when nothing is loaded, so functions that take one take a path:

```rust
let gt = GameThread::current()?;
let material = KismetSystemLibrary::load_asset_blocking(&gt, &SoftObjectRef::new(
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

Unreal objects belong to the game thread, and so do Rust's references to them: `UObjectRef`, `Checked`, `Pinned`, `ScriptInterface` and `SubclassOf` are not `Send`, so the compiler keeps them out of `task::spawn`'s work and out of rayon. Work in the background with plain data, or with a `TWeakObjectPtr` (which is `Send`) resolved back on the game thread, in `task::spawn`'s `done` or in `task::run_on_game_thread`:

```rust
let target = TWeakObjectPtr::from_ref(&target);

task::spawn(
    move || plan_path(&grid),          // a worker thread: no engine calls
    move |path| {                      // the game thread
        if let Some(target) = target.get() {
            follow(target, path);
        }
    },
);
```

Development builds check every engine call: one made off the game thread panics with the name of the call, where it would race with the game thread in C++. A `#[ufunction(ThreadSafe)]` may read and write properties from the worker thread the engine runs it on. Shipping builds leave the check out. See [Work off the game thread](../systems/tasks.md).
