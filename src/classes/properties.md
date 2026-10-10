# Properties

**The goal:** values that designers tune in the editor, that Blueprints read and write, and that the engine saves with the object.

```rust
use bindings::engine::{AnimMontage, SoundBase};
use bindings::prelude::*;
use rusteal_runtime::runtime::{FName, SubclassOf, UObjectRef, UeArray};
use rusteal_runtime::{uclass, uclass_impl};

#[uclass(parent = Actor)]
pub struct Turret {
    #[uproperty(EditAnywhere, category = "Combat", default = 25.0)]
    damage: f32,

    #[uproperty(EditAnywhere, category = "Combat", default = 3)]
    burst_size: i32,

    #[uproperty(EditAnywhere, BlueprintReadWrite, default = true)]
    b_active: bool,

    #[uproperty(EditAnywhere, category = "Effects")]
    fire_sound: UObjectRef<SoundBase>,

    #[uproperty(EditAnywhere, category = "Effects")]
    fire_montage: UObjectRef<AnimMontage>,

    #[uproperty(EditAnywhere, category = "Combat")]
    projectile_class: SubclassOf<Actor>,

    #[uproperty(EditAnywhere)]
    target_tags: UeArray<FName>,

    #[uproperty]
    current_target: UObjectRef<Actor>,

    shots_fired: u32,
}

#[uclass_impl]
impl Turret {}
```

A field marked `#[uproperty]` is a real Unreal property. It lives in the object's memory, the editor shows it according to its specifiers, Blueprints reach it, it is saved with the level or the Blueprint, and object references in it keep the object alive for the garbage collector.

## Getters and setters

Each property gets a getter named after the field and a setter with `set_`:

```rust
let damage = self.damage();
self.set_burst_size(5);
self.set_current_target(target);
```

The getter of a scalar, an object, a class or a name returns a copy. The getter of a `UeArray` returns a view of the array inside the object, changed in place:

```rust
let tags = self.target_tags();
tags.push(&FName::new("Player"))?;
let all: Vec<_> = tags.to_vec()?;
```

The getter of a struct property (`OwnedStruct<FVector>`) returns a copy; change it and set it back.

## Specifiers

| Specifier | What it does |
|---|---|
| `EditAnywhere` | Editable in the **Details** panel, on the class defaults and on each instance in a level. |
| `EditDefaultsOnly` | Editable on the class defaults only (a Blueprint's **Class Defaults**), not per instance. |
| `VisibleAnywhere` | Shown, read-only, in the **Details** panel. |
| `BlueprintReadWrite` | Blueprints get **Get** and **Set** nodes for it. |
| `BlueprintReadOnly` | Blueprints get a **Get** node only. Rust code still has the setter. |
| `SaveGame` | Written by a save game archive. See [Saving the game](../recipes/save_game.md). |
| `Config` | Read from and saved to the ini files of the class's config category. See [Project settings](../systems/settings.md). |

Several specifiers can be combined. A property with none (`#[uproperty]`) is hidden from the editor and from Blueprints, but is still a real property: use it for object references the class holds, so the garbage collector sees them.

Any other argument is a compile error.

## Named arguments

| Argument | What it does |
|---|---|
| `default = <value>` | The property's default value, for scalars and engine enums: `default = 25.0`, `default = true`, `default = ECollisionChannel::ECC_Pawn`, or a constant (`default = model::MAX_HP`). |
| `category = "..."` | The category it is listed under in the **Details** panel. `"Combat|Ranged"` makes a subcategory. |
| `name = "..."` | The Unreal name, when the field's name in PascalCase is not the one you want. |

`default` is for scalars and enums. Objects, classes, arrays and structs get their defaults in [Class defaults](class_defaults.md) or in a Blueprint child.

## Names

The Unreal name is the field's name in PascalCase: `burst_size` is `BurstSize`. A `bool` whose field starts with `b_` follows Unreal's `b` convention: `b_active` is `bActive`. When PascalCase gets it wrong, `name` fixes it:

```rust
#[uproperty(EditAnywhere, name = "MaxHP", default = 100.0)]
max_hp: f32,

#[uproperty(EditAnywhere, name = "NPCClass")]
npc_class: SubclassOf<Actor>,
```

The Unreal name is what Blueprints, ini files, data tables and editor Python see. Changing it on a class that already has Blueprint children or placed instances loses the values they saved under the old name.

## Types

| Rust type | Unreal type |
|---|---|
| `bool`, `u8`, `i32`, `i64`, `f32`, `f64` | `bool`, `uint8`, `int32`, `int64`, `float`, `double` |
| `String` | `FString` |
| `FName` | `FName` |
| An engine enum (`ECollisionChannel`, `ETraceTypeQuery`...) or a `#[uenum]` of yours | that enum |
| `UObjectRef<T>` | `TObjectPtr<T>`: an object or an asset |
| `SubclassOf<T>` | `TSubclassOf<T>`: a class, such as a Blueprint to spawn |
| `SoftObjectRef<T>` | `TSoftObjectPtr<T>`: an asset by path, loaded on demand |
| `OwnedStruct<T>` | a struct: `FVector`, `FLinearColor`, `FTransform`, `FDataTableRowHandle`, a `#[ustruct]` of yours |
| `UeArray<T>` of the scalars, objects, classes or `FName` | `TArray<T>` |

Maps and sets (`TMap`, `TSet`) cannot be declared yet; two arrays side by side, or a Rust-only `HashMap`, do the job meanwhile.

## Fields only Rust sees

A field without an attribute is Rust's alone: any Rust type, kept outside the Unreal object. It gets the same getter and setter, plus a `<field>_mut()` that changes it in place:

```rust
self.set_shots_fired(self.shots_fired() + 1);
self.cooldowns_mut().insert(weapon, 2.0);
```

The getter of a Rust-only field returns a clone. Use these fields for state the editor does not need: caches, timers, a state machine, a `HashMap`. Two things to keep in mind:

- They start over from `Default` on every [hot reload](../iteration/hot_reload.md).
- An `UObjectRef` kept only there does not keep its object alive: the garbage collector does not see it. Hold the objects you need to keep in a `#[uproperty]`, or check them with `checked()` before use (see [Object references](../engine_api/object_refs.md)).

The templates keep a class's transient state in one plain struct field, read, changed and written back:

```rust
#[derive(Clone, Copy, Default)]
pub struct MoveState {
    pub has_double_jumped: bool,
    pub last_fall_time: f64,
}

let mut state = self.state();
state.has_double_jumped = true;
self.set_state(state);
```

## In the editor

Open the class's Blueprint child, click **Class Defaults** in the toolbar, and the **Details** panel lists the `EditAnywhere` and `EditDefaultsOnly` properties under their categories. Select an instance in the level to see the `EditAnywhere` ones per instance.

> **Screenshot:** the Details panel of a Blueprint child showing the Combat and Effects categories of a Rust class. *(images/classes/details_categories.png)*
