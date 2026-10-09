# Project settings and config

**The goal:** values designers tune in **Project Settings**, saved in the project's ini files, and read by the game.

```rust
use bindings::developer_settings::DeveloperSettings;

#[uclass(parent = DeveloperSettings, config = "Game")]
pub struct CombatSettings {
    #[uproperty(Config, EditAnywhere, category = "Combat", default = 3)]
    max_combo: i32,

    #[uproperty(Config, EditAnywhere, category = "Combat", default = 0.4)]
    combo_window: f32,
}

#[uclass_impl]
impl CombatSettings {}
```

- A class whose parent is `DeveloperSettings` and that names a config category shows in **Project Settings**.
- `config = "Game"` saves its `Config` properties in the **Game** category's files: `Config/DefaultGame.ini` for the project's defaults.
- The values live on the class default object, which `get_default()` returns:

```rust
let settings = CombatSettings::get_default()?;
let window = settings.combo_window();
```

## Binding the module

`DeveloperSettings` is its own engine module. A game binds it by adding it to `[codegen.modules]` in `rusteal.toml`, then running `rusteal build --all`:

```toml
DeveloperSettings = { module = "developer_settings", feature = "engine" }
```

Plugin templates already bind it.

## In the editor

Open **Edit > Project Settings**. The class has a page of its own, named after the class, listed under **Game** by default. Changing a value there writes it to `Config/DefaultGame.ini`:

```ini
[/Script/Rusteal.CombatSettings]
MaxCombo=4
ComboWindow=0.35
```

The section is the class's path, and the keys the properties' Unreal names. A plugin's settings use the plugin's package (`[/Script/Inventory.InventorySettings]`).

> **Screenshot:** the Project Settings page of a Rust DeveloperSettings class. *(images/systems/project_settings_page.png)*

## Config on other classes

Any class can read properties from ini files: name a category in `#[uclass(config = "...")]` and mark the properties `Config`. The values are read into the class default object when the class is created, as in C++. `Game`, `Engine`, `Input` and `GameUserSettings` are the engine's categories; any other name makes a category of its own (`config = "Combat"` reads `Config/DefaultCombat.ini`).
