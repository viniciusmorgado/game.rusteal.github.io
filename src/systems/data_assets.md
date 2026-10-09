# Data assets

**The goal:** a kind of asset of your own, items, recipes, enemy types, dialogue lines, that designers create and edit in the **Content Browser**.

```rust
use bindings::engine::{PrimaryDataAsset, StaticMesh, Texture2D};

#[uclass(parent = PrimaryDataAsset)]
pub struct EnemyType {
    #[uproperty(EditAnywhere, BlueprintReadOnly, category = "Enemy")]
    display_name: String,

    #[uproperty(EditAnywhere, BlueprintReadOnly, category = "Enemy", default = 100.0)]
    max_health: f32,

    #[uproperty(EditAnywhere, BlueprintReadOnly, category = "Enemy")]
    mesh: UObjectRef<StaticMesh>,

    #[uproperty(EditAnywhere, BlueprintReadOnly, category = "Enemy")]
    icon: UObjectRef<Texture2D>,
}

#[uclass_impl]
impl EnemyType {}
```

A class whose parent is `PrimaryDataAsset` is a data asset class: each asset of it is a set of values, edited in the editor and loaded by the game.

## In the editor

1. In the **Content Browser**, right-click and choose **Miscellaneous > Data Asset**.
2. In the **Pick Class For Data Asset Instance** window, choose **EnemyType** and click **Select**.
3. Name the asset (`DA_Grunt`), open it, fill in its values, and **Save**.

> **Screenshot:** the Pick Class For Data Asset Instance window with the Rust class selected. *(images/systems/data_asset_class.png)*

## Using them

A property holds the asset a spawner or a level actor uses:

```rust
#[uproperty(EditAnywhere)]
enemy_type: UObjectRef<EnemyType>,
```

```rust
let enemy_type = EnemyType::from_obj(self.enemy_type())?;
let health = enemy_type.max_health();
```

Or the game loads one by path:

```rust
let grunt: UObjectRef<EnemyType> = load_object("/Game/Enemies/DA_Grunt.DA_Grunt")?;
```

An asset loaded only by path must be in a directory the cooker cooks: see [Packaging a game](../shipping/packaging.md).

## Creating them in bulk

Editor Python creates and fills them, using the classes' paths and the properties' Unreal names:

```python
import unreal

enemy_class = unreal.load_class(None, "/Script/Rusteal.EnemyType")
factory = unreal.DataAssetFactory()
factory.set_editor_property("data_asset_class", enemy_class)

tools = unreal.AssetToolsHelpers.get_asset_tools()
grunt = tools.create_asset("DA_Grunt", "/Game/Enemies", enemy_class, factory)
grunt.set_editor_property("MaxHealth", 80.0)
```
