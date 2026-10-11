# Assets and objects

## Assets assigned in the editor

Most assets reach Rust code through properties a Blueprint child fills: a `UObjectRef<StaticMesh>`, a `UObjectRef<SoundBase>`, a `SubclassOf<Actor>`. That is the recommended way: the cooker sees the reference and packages the asset.

```rust
#[uproperty(EditAnywhere)]
explosion_sound: UObjectRef<SoundBase>,
```

## Loading an asset by path

```rust
let mesh: UObjectRef<StaticMesh> = load_object("/Game/Props/SM_Crate.SM_Crate")?;
let cube: UObjectRef<StaticMesh> = load_object("/Engine/BasicShapes/Cube")?;
```

A class, a Blueprint's included, loads with `load_class`, checked against the class it must derive from:

```rust
let class: SubclassOf<AnimInstance> = load_class("/Game/Characters/ABP_Hero.ABP_Hero_C")?;
mesh.set_anim_instance_class(class.upcast_to());
```

A Blueprint's class path ends in `_C`. `load_object` loads the asset if it is not loaded yet, and returns an error if there is none at that path. `find_object` finds it only if it is already loaded. The path is the asset's object path, which **Copy Reference** in the **Content Browser**'s right-click menu puts on the clipboard.

An asset that only `load_object` uses is not found by the cooker, because nothing references it: list its directory in **Project Settings > Packaging > Additional Asset Directories to Cook**, or the packaged game will not have it. See [Packaging a game](../shipping/packaging.md).

## Soft references: loading on demand

```rust
#[uproperty(EditAnywhere)]
boss_music: SoftObjectRef<SoundBase>,
```

`SoftObjectRef<T>` is Unreal's `TSoftObjectPtr<T>`: an asset picked in the editor but not loaded with the object. It keeps big or rarely used assets out of memory until needed:

```rust
let music = self.boss_music().load_synchronous()?;
let loaded_already = self.boss_music().get();
```

`load_synchronous` loads it now (a hitch for a large asset); `get` returns it only if it is already loaded, or a null reference.

## Creating objects

```rust
use bindings::manual::world_ext::new_object;

let entry = new_object::<MenuEntry>(&self.as_ref())?;
```

`new_object::<T>(outer)` creates an object of class `T` inside `outer`, as C++'s `NewObject<T>(Outer)`. For actors, spawn instead (see [Spawning actors](spawning.md)); for widgets, see [UI](../recipes/ui.md).

Keep the new object in a `#[uproperty]`, or the garbage collector may collect it.
