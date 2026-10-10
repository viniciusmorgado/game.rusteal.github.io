# Types

## Unreal types in Rust

| C++ | Rust | Notes |
|---|---|---|
| `bool`, `uint8`, `int32`, `int64`, `float`, `double` | `bool`, `u8`, `i32`, `i64`, `f32`, `f64` | |
| `FString` | `String` (as a property or a return value), `&str` (as an argument) | |
| `FName` | `FName` (properties, Rust code), `FNameHandle` (engine arguments and returns) | `FName::new("x").handle()`, `FName::from(handle)` |
| `FText` | `&str` (as an argument), `String` (as a return value) | |
| `UObject*`, `TObjectPtr<T>` | `UObjectRef<T>` | a handle; see [Object references](../engine_api/object_refs.md) |
| `TSubclassOf<T>` | `SubclassOf<T>` | a class |
| `TScriptInterface<IFoo>` | `ScriptInterface<dyn Foo>` | an object implementing a `#[uinterface]`; `object.interface::<dyn Foo>()` |
| `TSoftObjectPtr<T>` | `SoftObjectRef<T>` | an asset by path |
| `TWeakObjectPtr<T>` | `TWeakObjectPtr<T>` | |
| A struct (`FVector`, `FHitResult`...) | `OwnedStruct<T>` (owned), `UStructRef<T>` (a reference into Unreal memory) | fields through the `<Struct>Ext` trait |
| An enum (`ECollisionChannel`) | the Rust enum of the same name, or its `u8` in some events | |
| Your enum | a `#[uenum]` | |
| `TArray<T>` | `UeArray<T>` (a property), `Vec<T>` or `&[T]` (engine arguments and returns) | |
| `TMap<K, V>` | `UeMap<K, V>` | engine properties only |
| `TSet<T>` | `UeSet<T>` | engine properties only |
| An output parameter | part of the returned tuple; in an event, `UStructRef<T>` or `OutRef<T>` | |

## Math

| Rusteal and glam | Unreal | Conversion |
|---|---|---|
| `glam::DVec3` | `FVector` | `FVector::from_dvec3(v)`, `fvector.to_dvec3()` |
| `glam::DVec2` | `FVector2D` | `FVector2D::from_dvec2(v)`, `.to_dvec2()` |
| `glam::DQuat` | `FQuat` | `FQuat::from_dquat(q)`, `.to_dquat()` |
| `Rotator` | `FRotator` | `FRotator::from_rotator(r)`, `.to_rotator()` |
| `Transform` | `FTransform` | `FTransform::from_transform(t)`, `.to_transform()` |
| `LinearColor` | `FLinearColor` | `FLinearColor::from_linear_color(c)` |

`Rotator`, `Transform`, `LinearColor` and `Color` come from `rusteal_runtime::runtime`; glam's types from `glam`, re-exported as `rusteal_runtime::glam`.

## Containers

| Method | `UeArray<T>` | `UeMap<K, V>` | `UeSet<T>` |
|---|---|---|---|
| Size | `len()`, `is_empty()` | `len()`, `is_empty()` | `len()`, `is_empty()` |
| Read | `get(i)`, `iter()`, `to_vec()` | `find(&k)`, `iter()`, `to_hash_map()` | `contains(&v)`, `iter()`, `to_hash_set()` |
| Write | `set(i, &v)`, `push(&v)`, `remove(i)`, `clear()`, `set_all(&[..])` | `add(&k, &v)`, `remove(&k)`, `clear()` | `add(&v)`, `remove(&v)`, `clear()` |

Every method returns a `RustealResult`: the object holding the container may be gone.

## Errors

`RustealResult<T>` is `Result<T, RustealError>`. See [Errors](../engine_api/errors.md).
