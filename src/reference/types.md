# Types

## Unreal types in Rust

| C++ | Rust | Notes |
|---|---|---|
| `bool`, `int8`, `int16`, `int32`, `int64`, `uint8`, `uint16`, `uint32`, `uint64`, `float`, `double` | `bool`, `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64` | the integers other than `u8`, `i32` and `i64` in your classes since 0.10 |
| `FString` | `String` (properties, returns, your functions' parameters), `&str` (engine arguments) | |
| `FName` | `FName` (properties, Rust code), `FNameHandle` (engine arguments and returns) | `FName::new("x").handle()`, `FName::from(handle)` |
| `FText` | `Text` (since 0.10; arguments take `impl Into<Text>`, so a `&str` works) | keeps its localization; see [Text and localization](../engine_api/text.md) |
| `UObject*`, `TObjectPtr<T>` | `UObjectRef<T>` | a handle; see [Object references](../engine_api/object_refs.md) |
| `TSubclassOf<T>` | `SubclassOf<T>` | a class |
| `TScriptInterface<IFoo>` | `ScriptInterface<dyn Foo>` | an object implementing a `#[uinterface]`; `object.interface::<dyn Foo>()` (after 0.7.0) |
| `TScriptInterface<IBlendableInterface>` (an engine interface) | `UObjectRef<BlendableInterface>` | `object.cast::<BlendableInterface>()`; `FScriptInterface` is `UObjectRef<Object>` (0.10) |
| `TSoftObjectPtr<T>` | `SoftObjectRef<T>` | an asset by path; the engine's too since 0.10 |
| `TSoftClassPtr<T>` | `SoftClassRef<T>` | a class by path, such as a Blueprint not loaded yet; engine properties and functions (0.10) |
| `TWeakObjectPtr<T>` | `TWeakObjectPtr<T>` | the engine's too since 0.10 |
| `TLazyObjectPtr<T>` | `TWeakObjectPtr<T>` | engine properties (0.10) |
| `TOptional<T>` | `Option<T>` | engine properties and struct fields of bools, numbers, enums, structs and objects (0.10) |
| A struct (`FVector`, `FHitResult`, `FIntPoint`...) | `OwnedStruct<T>` (owned), `UStructRef<T>` (a reference into Unreal memory) | fields through the `<Struct>Ext` trait; every engine struct since 0.10 |
| `FInstancedStruct` | `OwnedStruct<FInstancedStruct>` | holds any struct: `FInstancedStruct::make(&value)`, `get::<T>()`, `set(&value)`, `reset()`, `script_struct()` (0.10) |
| An enum (`ECollisionChannel`) | the Rust enum of the same name, or its `u8` in some events | |
| Your enum | a `#[uenum]` | |
| `TArray<T>` | `UeArray<T>` (a property), `Vec<T>` (your functions and delegates; engine returns), `&[T]` (engine arguments) | |
| `TMap<K, V>` | `UeMap<K, V>` (a property), `HashMap<K, V>` (your functions and delegates) | yours since 0.10 |
| `TSet<T>` | `UeSet<T>` (a property), `HashSet<T>` (your functions and delegates) | yours since 0.10 |
| An output parameter | engine functions: part of the returned tuple; your functions: `&mut T` (0.10); an engine event's: `UStructRef<T>` or `OutRef<T>` | |
| A dynamic delegate (`FScriptDelegate`, `DECLARE_DYNAMIC_DELEGATE`) | a closure, as an engine argument (0.10); yours: `#[udelegate(single)]` | see [Delegates](../engine_api/delegates.md) |
| A dynamic multicast delegate, sparse or not | the engine's: `on_x().add(closure)`, `on_x().add_ufunction(&target, "Function")`; yours: `#[udelegate]`, `#[udelegate(sparse)]` | |
| `TFieldPath<T>` | not bound | one engine function takes it (`KismetSystemLibrary::SetFieldPathPropertyByName`) |

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
