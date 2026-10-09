# An inventory

**The goal:** items defined as assets, an inventory component any actor can have, a delegate that tells the UI when it changes, and limits set in Project Settings.

This is what the `inventory` plugin template builds:

```bash
rusteal plugin new Inventory --template inventory
```

The same code works in a game's crate. The pieces:

## Items as data assets

```rust
use bindings::engine::PrimaryDataAsset;

#[uclass(parent = PrimaryDataAsset)]
pub struct InventoryItem {
    #[uproperty(EditAnywhere, BlueprintReadOnly, category = "Item")]
    display_name: String,

    #[uproperty(EditAnywhere, BlueprintReadOnly, category = "Item", default = 1)]
    max_stack: i32,

    #[uproperty(EditAnywhere, BlueprintReadOnly, category = "Item", default = 0.0)]
    weight: f32,
}
```

Each item is an asset of this class: see [Data assets](../systems/data_assets.md) for creating them in the editor.

## The component

```rust
use bindings::engine::ActorComponent;

#[uclass(parent = ActorComponent)]
pub struct InventoryComponent {
    #[uproperty(VisibleAnywhere, BlueprintReadOnly, category = "Inventory")]
    items: UeArray<UObjectRef<InventoryItem>>,

    #[uproperty(VisibleAnywhere, BlueprintReadOnly, category = "Inventory")]
    counts: UeArray<i32>,
}

#[uclass_impl]
impl InventoryComponent {
    #[udelegate]
    pub fn on_inventory_changed(&self, item: UObjectRef<InventoryItem>, count: i32) {}

    #[ufunction]
    pub fn add_item(&mut self, item: UObjectRef<InventoryItem>, count: i32) -> i32 {
        let added = self.try_add(item, count).unwrap_or(0);

        if added > 0 {
            self.on_inventory_changed(item, self.count_of(item));
        }

        added
    }

    #[ufunction(BlueprintPure)]
    pub fn count_of(&self, item: UObjectRef<InventoryItem>) -> i32 {
        self.stacks()
            .map(|stacks| stacks.iter().filter(|(i, _)| *i == item).map(|(_, c)| c).sum())
            .unwrap_or(0)
    }
}

impl InventoryComponent {
    pub fn stacks(&self) -> RustealResult<Vec<(UObjectRef<InventoryItem>, i32)>> {
        let items = self.items().to_vec()?;
        let counts = self.counts().to_vec()?;
        Ok(items.into_iter().zip(counts).collect())
    }

    fn try_add(&mut self, item: UObjectRef<InventoryItem>, count: i32) -> RustealResult<i32> {
        let definition = InventoryItem::from_obj(item)?;
        let settings = InventorySettings::get_default()?;
        let max_stack = definition.max_stack().max(1);
        let (items, counts) = (self.items(), self.counts());
        let mut left = count.max(0);

        for index in 0..items.len()? {
            if left == 0 {
                break;
            }

            let held = counts.get(index)?;

            if items.get(index)? == item && held < max_stack {
                let put = left.min(max_stack - held);
                counts.set(index, &(held + put))?;
                left -= put;
            }
        }

        while left > 0 && (items.len()? as i32) < settings.max_slots() {
            let put = left.min(max_stack);
            items.push(&item)?;
            counts.push(&put)?;
            left -= put;
        }

        Ok(count.max(0) - left)
    }
}
```

- Two arrays side by side hold the stacks: properties cannot be maps yet.
- `on_inventory_changed` is a delegate a UMG widget binds to redraw (see [Delegates](../engine_api/delegates.md)).
- `add_item` and `count_of` are callable from Blueprints.

## Giving any actor an inventory

A world subsystem finds an actor's inventory, adding the component when asked:

```rust
use bindings::engine::{ActorComponent, ActorExt, SubsystemBlueprintLibrary, WorldSubsystem};
use bindings::rusteal::RustealWorldSubsystem;

#[uclass(parent = RustealWorldSubsystem)]
pub struct InventorySubsystem {}

#[uclass_impl]
impl InventorySubsystem {
    #[ufunction]
    pub fn inventory_of(&mut self, actor: UObjectRef<Actor>, create: bool) -> UObjectRef<InventoryComponent> {
        find_or_add(actor, create).unwrap_or_default()
    }
}

fn find_or_add(actor: UObjectRef<Actor>, create: bool) -> RustealResult<UObjectRef<InventoryComponent>> {
    let actor = actor.checked()?;
    let class = SubclassOf::<InventoryComponent>::base().upcast_to::<ActorComponent>();
    let mut component = actor.get_component_by_class(class);

    if !component.is_valid() && create {
        let identity = FTransform::from_transform(Transform::IDENTITY);
        component = actor.add_component_by_class(class, false, &identity, false);
    }

    component.cast::<InventoryComponent>()
}
```

See [Subsystems](../systems/subsystems.md) for finding the subsystem itself.

## Limits in Project Settings

```rust
use bindings::developer_settings::DeveloperSettings;

#[uclass(parent = DeveloperSettings, config = "Game")]
pub struct InventorySettings {
    #[uproperty(Config, EditAnywhere, category = "Inventory", default = 20)]
    max_slots: i32,
}
```

See [Project settings](../systems/settings.md).

## Trying it from the console

The template registers `give`, `take` and `list` console commands (see [Console commands and variables](../systems/console.md)): press **Play**, open the console and type the command with an item's path.
