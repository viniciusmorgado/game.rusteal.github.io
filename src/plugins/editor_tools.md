# Editor tools

An editor plugin's library runs in the editor only, and binds the editor's modules, which the `editor` template lists. It reaches the engine's editor-only functions as well, such as `set_actor_label`.

## Use case: a menu entry

**The goal:** **Tools > LevelTools > Count assets** runs Rust code.

A menu entry is a class whose parent is `ToolMenuEntryScript`, with `execute` overridden:

```rust
use bindings::tool_menus::{FToolMenuContext, ToolMenuEntryScript};

#[uclass(parent = ToolMenuEntryScript)]
pub struct CountAssetsEntry {}

#[uclass_impl]
impl CountAssetsEntry {
    #[ufunction(Override)]
    fn execute(&mut self, _context: UStructRef<FToolMenuContext>) {
        count_assets("/Game");
    }
}
```

An editor subsystem creates it when the editor starts and removes it when it closes:

```rust
use bindings::manual::world_ext::new_object;
use bindings::rusteal_editor::RustealEditorSubsystem;
use bindings::tool_menus::{ToolMenuEntryScript, ToolMenuEntryScriptExt};

#[uclass(parent = RustealEditorSubsystem)]
pub struct LevelToolsSubsystem {
    #[uproperty]
    menu_entry: UObjectRef<ToolMenuEntryScript>,
}

#[uclass_impl]
impl LevelToolsSubsystem {
    #[ufunction(Override)]
    fn receive_initialize(&mut self) {
        if let Err(e) = self.add_menu_entry() {
            ulog!(LOG_WARNING, "[LevelTools] no menu entry: {e}");
        }
    }

    #[ufunction(Override)]
    fn receive_deinitialize(&mut self) {
        if let Ok(entry) = self.menu_entry().checked() {
            entry.unregister_menu_entry();
        }
    }
}

impl LevelToolsSubsystem {
    fn add_menu_entry(&mut self) -> RustealResult<()> {
        let entry = new_object::<CountAssetsEntry>(&self.as_ref())?.upcast_to::<ToolMenuEntryScript>();
        let script = entry.checked()?;

        script.init_entry(
            FName::new("LevelTools").handle(),
            FName::new("LevelEditor.MainMenu.Tools").handle(),
            FName::new("LevelTools").handle(),
            FName::new("LevelToolsCountAssets").handle(),
            Some("Count assets"),
            Some("Count the project's assets by type, off the game thread"),
        );

        script.register_menu_entry();
        self.set_menu_entry(entry);

        Ok(())
    }
}
```

`init_entry` takes the owner, the menu (`LevelEditor.MainMenu.Tools` is the **Tools** menu), the section, the entry's name, its label and its tooltip.

## Use case: work on many assets, off the game thread

**The goal:** list the project's assets and count them by name prefix without freezing the editor.

```rust
use bindings::unreal_ed::{EditorAssetSubsystem, EditorAssetSubsystemExt};
use rusteal_runtime::runtime::task;

pub fn count_assets(path: &str) {
    let assets = match editor_subsystem::<EditorAssetSubsystem>().and_then(|s| s.checked()) {
        Ok(subsystem) => subsystem.list_assets(path, Some(true), Some(false)),
        Err(e) => {
            ulog!(LOG_ERROR, "[LevelTools] no asset subsystem: {e}");
            return;
        }
    };

    let path = path.to_string();

    task::spawn(
        move || {
            let mut by_prefix = std::collections::BTreeMap::<String, usize>::new();

            for asset in &assets {
                let name = asset.rsplit('/').next().unwrap_or(asset);
                let prefix = name.split_once('_').map_or("(no prefix)", |(p, _)| p);
                *by_prefix.entry(prefix.to_string()).or_default() += 1;
            }

            (assets.len(), by_prefix)
        },
        move |(total, by_prefix)| {
            ulog!(LOG_DISPLAY, "[LevelTools] {total} assets under {path}");

            for (prefix, count) in by_prefix {
                ulog!(LOG_DISPLAY, "[LevelTools]   {prefix}: {count}");
            }
        },
    );
}
```

The engine call that lists the assets runs on the game thread; the counting, plain Rust, runs on a thread of its own (see [Work off the game thread](../systems/tasks.md)).

The editor's own subsystems (`EditorActorSubsystem`, `EditorAssetSubsystem`, `LevelEditorSubsystem`) come from `EditorSubsystemBlueprintLibrary::get_editor_subsystem`, which the template wraps:

```rust
use bindings::editor_subsystem::EditorSubsystem;
use bindings::unreal_ed::EditorSubsystemBlueprintLibrary;

pub fn editor_subsystem<T: UeClass>() -> RustealResult<UObjectRef<T>> {
    let class = unsafe { SubclassOf::<EditorSubsystem>::from_raw(T::static_class()) };
    EditorSubsystemBlueprintLibrary::get_editor_subsystem(class).cast::<T>()
}
```

## Use case: validating assets on save

**The goal:** saving an asset whose name has no type prefix (`SM_`, `T_`, `BP_`) fails, with a message.

```rust
use bindings::core_ue::Object;
use bindings::data_validation::{EditorValidatorBase, EditorValidatorBaseExt};

const INVALID: u8 = 0;
const VALID: u8 = 1;

#[uclass(parent = EditorValidatorBase)]
pub struct NameValidator {}

#[uclass_impl]
impl NameValidator {
    #[ufunction(Override, name = "K2_CanValidateAsset")]
    fn can_validate_asset(&self, asset: UObjectRef<Object>) -> bool {
        asset.is_valid()
    }

    #[ufunction(Override, name = "K2_ValidateLoadedAsset")]
    fn validate_loaded_asset(&mut self, asset: UObjectRef<Object>) -> u8 {
        let Ok(name) = asset.get_name() else {
            return VALID;
        };

        let Ok(me) = self.as_ref().checked() else {
            return VALID;
        };

        if name.contains('_') {
            me.asset_passes(asset);
            VALID
        } else {
            me.asset_fails(asset, &format!("'{name}' has no type prefix (SM_, T_, BP_...)"));
            INVALID
        }
    }
}
```

A class whose parent is `EditorValidatorBase` runs when assets are saved, and on **Validate Assets** in the **Content Browser**'s right-click menu. It returns an `EDataValidationResult` as its `u8` value. Validation results show in the **Message Log**.

## Use case: one undo step for many changes

`KismetSystemLibrary::begin_transaction`, `transact_object` and `end_transaction` group changes into one step of **Edit > Undo**: begin the transaction, call `transact_object` on each object before changing it, and end the transaction.
