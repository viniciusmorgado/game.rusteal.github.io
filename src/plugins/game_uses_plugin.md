# A plugin's classes in the game

**The goal:** the game uses the Rust classes of a runtime plugin as it uses the engine's: spawn them, read and write their properties, call their functions, and write game classes that derive from them.

```rust
use bindings::plugins::inventory::{InventoryComponent, InventoryComponentExt};

#[uclass(parent = InventoryComponent)]
pub struct HeroInventory {
    #[uproperty(EditAnywhere, default = 3)]
    potion_slots: i32,
}

#[uclass_impl]
impl HeroInventory {
    #[ufunction(Override)]
    fn on_item_added(&mut self, item: FName) {
        log::info!("the hero picked up {item}");
    }
}

fn give_potion(hero: UObjectRef<HeroInventory>) -> RustealResult<()> {
    let inventory = hero.checked()?;

    if inventory.capacity() > inventory.count()? {
        inventory.add_item(FName::new("Potion"))?;
    }

    Ok(())
}
```

Since 0.10, `rusteal build` reads the Rust sources of every runtime plugin of the project, and the game's bindings get its classes under `bindings::plugins::<plugin>`, in snake_case (`Inventory` is `inventory`, `RustealProbe` is `rusteal_probe`). Each class is there as an engine class is:

| What the plugin's class has | What the game sees |
|---|---|
| A `#[uproperty]` | `charge()` and `set_charge(value)` on `<Class>Ext`, a copy for containers (`Vec`, `HashMap`, `HashSet`) |
| A `#[component]` | a getter of its `UObjectRef<T>` |
| A `#[ufunction]`, an event | the same method on `<Class>Ext`, returning `RustealResult<T>`; a static one on the class itself |
| Its parent | `Deref` to the parent's methods, and `cast`, `upcast_to` and `SubclassOf<T>` as for any class |
| Rust-only fields and methods | nothing: they belong to the plugin's library |

A call crosses into the plugin's library through the engine, as a Blueprint's does, so it returns a `RustealResult`: it fails with an error, not a crash, when the plugin is not loaded or its class changed under a stale build.

## A game class with a plugin parent

`#[uclass(parent = InventoryComponent)]` derives from the plugin's class, overrides its events (`#[ufunction(Override)]`), and adds properties and Rust fields of its own. Each library keeps its own Rust data on the object: the plugin's fields stay the plugin's, the game's the game's.

The game's library loads before the plugins'. A game class with a plugin parent waits until the plugin's library has registered, and is created right then, still before any level or asset loads. The Output Log says it:

```text
LogRusteal: Display: [Rusteal] my_game: one class waits for a parent class in another library: HeroInventory
LogRusteal: Display: [Rusteal] my_game: created HeroInventory, whose parent is in another library
```

A class whose properties or components use a waiting class waits with it. When the plugin never loads (it is disabled, or its library failed), the waiting classes are reported as errors once the engine has started.

## Hot reload

Reloading the plugin reloads the game too when it has to: when a plugin class the game derives from changed its properties, its functions or its parent, the game's library is reloaded right after the plugin's, and the editor reinstances both together. A change that only touches the bodies of the plugin's functions reloads the plugin alone.

## What is not read

- Classes in a module declared with `#[path]`, and items behind a macro of the plugin's own.
- The plugin's `#[ustruct]`s and `#[uenum]`s, and members that use them: the game reaches them through dynamic calls.
- The plugin's `#[udelegate]`s: bind them by name with `add_ufunction`, or with a [dynamic call](../engine_api/dynamic_calls.md).
- Another plugin's classes from a plugin, and an editor library's classes: only the game's bindings get plugin classes, from runtime libraries.
