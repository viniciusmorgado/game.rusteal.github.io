# Calling the engine

The bindings give Rust every function Blueprints can call, for the engine modules the project binds. This page covers how they look from Rust.

## Finding a function

A Blueprint node is a function of some class. Its Rust name is its C++ name in snake_case, on the class's `Ext` trait:

| Blueprint node | C++ | Rust |
|---|---|---|
| **Get Actor Location** | `AActor::K2_GetActorLocation` | `actor.k2_get_actor_location()` (`ActorExt`) |
| **Set Max Walk Speed** | `UCharacterMovementComponent::MaxWalkSpeed` | `movement.set_max_walk_speed(600.0)` (`CharacterMovementComponentExt`) |
| **Launch Character** | `ACharacter::LaunchCharacter` | `character.launch_character(&v, true, true)` (`CharacterExt`) |
| **Get Player Controller** | `UGameplayStatics::GetPlayerController` | `GameplayStatics::get_player_controller(ctx, 0)` |

The C++ name is in the node's tooltip in the Blueprint editor, and in the engine's headers. Many Blueprint nodes are `K2_` functions in C++ (`K2_GetActorLocation`, `K2_SetTimer`, `K2_DestroyActor`).

Import the class and its `Ext` trait from its module:

```rust
use bindings::engine::{Character, CharacterExt, CharacterMovementComponentExt};
```

rust-analyzer completes the methods on a checked reference, which is the quickest way to browse what a class offers.

## Static functions

Function libraries (`UGameplayStatics`, `UKismetSystemLibrary`, `UKismetMathLibrary`) are called on the class, as in C++:

```rust
let now = GameplayStatics::get_time_seconds(self.as_ref().upcast_to());
let roll = KismetMathLibrary::random_integer_in_range(0, 100);
```

## Properties of engine classes

The properties Blueprints can read have a getter, `get_<name>()`, and the ones they can write a setter, `set_<name>()`: `get_max_walk_speed()`, `set_max_walk_speed(600.0)`. An array property's getter is the property's name alone (`tags()`). Private engine properties that Blueprints can read (`ACharacter`'s `Mesh`, `CharacterMovement` and `CapsuleComponent`) have getters, without setters when they are read-only:

```rust
let mesh = me.get_mesh().checked()?;
let movement = me.get_character_movement().checked()?;
let tags = me.tags();
```

An array property returns a view of the array inside the object (`UeArray`), changed in place:

```rust
me.tags().push(&FName::new("Dead").handle())?;
```

## Optional parameters

A parameter with a default value in C++ is an `Option` in Rust. `None` keeps the default:

```rust
anim_instance.montage_play(montage, Some(1.0), None, Some(0.0), Some(true));
capsule.set_capsule_size(42.0, 96.0, Some(false));
```

## Out parameters

A function with out parameters returns them in a tuple, after its return value:

```rust
let (hit, out_hit) = KismetSystemLibrary::line_trace_single(...);
let (projected, screen_position) = controller.project_world_location_to_screen(&location, Some(true));
let (found, x, y) = controller.get_mouse_position();
```

## Names, strings and text

| C++ | Rust argument | Rust return value |
|---|---|---|
| `FName` | `FName::new("Socket").handle()` | `FNameHandle`, `FName::from(handle)` |
| `FString` | `&str` | `String` |

```rust
let socket = FName::new("HandGrip_R");
let location = mesh.get_socket_location(socket.handle()).to_dvec3();
```

## Math

Engine structs (`FVector`, `FRotator`, `FTransform`, `FQuat`, `FLinearColor`) are Unreal's memory, held in an `OwnedStruct`. Rust math uses [glam](https://docs.rs/glam) (`DVec3`, `DQuat`, `DVec2`) and Rusteal's own `Rotator`, `Transform` and `LinearColor`. Convert at the boundary:

| From | To |
|---|---|
| `FVector` returned by the engine | `.to_dvec3()` |
| a `DVec3` passed to the engine | `&FVector::from_dvec3(v)` |
| `FRotator` | `.to_rotator()`, `FRotator::from_rotator(Rotator::new(pitch, yaw, roll))` |
| `FTransform` | `.to_transform()`, `FTransform::from_transform(Transform::new(rotation, translation, scale))` |
| `FQuat` | `.to_dquat()`, `FQuat::from_dquat(q)` |
| `FVector2D` | `.to_dvec2()`, `FVector2D::from_dvec2(v)` |
| `FLinearColor` | `FLinearColor::from_linear_color(LinearColor::new(r, g, b, a))` |

```rust
let me = self.as_ref().checked()?;
let forward = me.get_actor_forward_vector().to_dvec3();
let target = me.k2_get_actor_location().to_dvec3() + forward * 500.0;
me.k2_set_actor_location(&FVector::from_dvec3(target), false, true);
```

The conversions and `FVector`, `FRotator`, `FTransform` come with `use bindings::prelude::*`. Unreal's units are kept: centimeters, degrees for rotators, Z up.

## Structs: reading and writing fields

An engine struct has a getter and a setter per field, on its `Ext` trait (`FHitResultExt`, `FVectorExt`):

```rust
use bindings::engine::{ActorComponentExt, FHitResultExt};

let hit = out_hit.as_ref();
let impact = hit.get_impact_point().to_dvec3();
let normal = hit.get_impact_normal().to_dvec3();
let other = hit.get_component().checked()?.get_owner();
```

`OwnedStruct<T>` is a struct Rust owns; `as_ref()` gives the `UStructRef<T>` the field accessors work on. `OwnedStruct::new()` is a struct with Unreal's default values, the usual way to pass "no options":

```rust
subsystem.add_mapping_context(context, 0, &OwnedStruct::new());
```

To change a struct an engine object holds, read it, change the copy and write it back:

```rust
let nav = movement.get_nav_agent_props();
nav.as_ref().set_agent_radius(42.0);
movement.set_nav_agent_props(&nav);
```

## Enums

Engine enums are Rust enums with the C++ names: `ECollisionChannel::ECC_Pawn`, `ETriggerEvent::Triggered`, `EAttachmentRule::SnapToTarget`. Events receive some enums as their `u8` value (`receive_end_play(&mut self, end_play_reason: u8)`). Your own enums are declared with `#[uenum]`; see [Enums](../classes/enums.md).

## Functions that are not in the bindings

A function that reflection exposes but the bindings leave out (a Blueprint's own function, a module the project does not bind) can still be called by name. See [Dynamic calls](dynamic_calls.md).
