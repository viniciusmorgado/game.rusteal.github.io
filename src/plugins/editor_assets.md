# Assets and editor utilities

**The goal:** a new kind of asset the **Content Browser** creates, a file format the editor imports, and small tools the editor runs, all in an editor library's Rust.

## Use case: a new asset and an importer

```rust
use bindings::core_ue::Object;
use bindings::engine::DataAsset;
use bindings::rusteal_editor::RustealFactory;
use bindings::unreal_ed::FactoryExt;

#[uclass(parent = DataAsset)]
pub struct DialogueScript {
    #[uproperty(EditAnywhere)]
    lines: UeArray<String>,
}

#[uclass(parent = RustealFactory)]
pub struct DialogueScriptFactory {}

#[uclass_impl]
impl DialogueScriptFactory {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let factory = self.as_ref().checked()?;
        factory.set_supported_class(SubclassOf::<DialogueScript>::base().upcast_to::<Object>());
        factory.set_create_new(true);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_created_new(&mut self, new_object: UObjectRef<Object>, _context: UObjectRef<Object>) {
        if let Ok(script) = DialogueScript::from_obj(new_object) {
            let _ = script.lines().push(&"Hello.".to_string());
        }
    }
}

#[uclass(parent = RustealFactory)]
pub struct DialogueScriptImporter {}

#[uclass_impl]
impl DialogueScriptImporter {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let factory = self.as_ref().checked()?;
        factory.set_supported_class(SubclassOf::<DialogueScript>::base().upcast_to::<Object>());
        factory.set_editor_import(true);
        factory.formats().push(&"dlg;Dialogue script".to_string())?;
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_import_file(&mut self, new_object: UObjectRef<Object>, filename: String) -> bool {
        let Ok(text) = std::fs::read_to_string(&filename) else {
            return false;
        };

        let Ok(script) = DialogueScript::from_obj(new_object) else {
            return false;
        };

        for line in text.lines().filter(|line| !line.is_empty()) {
            let _ = script.lines().push(&line.to_string());
        }

        true
    }
}
```

`RustealFactory` (since 0.10) is the editor's factory, `UFactory`, for a Rust class: it creates the new object, with the name, the outer and the flags the editor asks for, and raises an event to fill it.

| Event | When |
|---|---|
| `receive_created_new(new_object, context)` | the Content Browser's **Add** menu, or `create_asset`, made a new asset |
| `receive_import_file(new_object, filename) -> bool` | the editor imports a file of one of the factory's formats; `false` refuses it, and the new object is thrown away |
| `receive_can_import(filename) -> bool` | the editor asks whether a file is the factory's; by default, by its extension |
| `receive_should_show_in_new_menu() -> bool`, `receive_get_display_name() -> Text` | the **Add** menu |

A factory that creates new assets and one that imports files are two classes: a factory with `create_new` set creates even when it imports. The factory's settings (`supported_class`, `create_new`, `editor_import`, `formats`) go in its class defaults. Any crate from crates.io can parse the file: the import runs on the game thread, as the editor's importers do.

## Use case: a tool the editor runs

```rust
use bindings::blutility::EditorUtilityObject;

#[uclass(parent = EditorUtilityObject)]
pub struct FixMaterialNames {}

#[uclass_impl]
impl FixMaterialNames {
    #[ufunction(Override)]
    fn run(&mut self) {
        rename_materials();
    }
}
```

The `editor` template binds Blutility (since 0.10), the module of the editor's utilities. An `EditorUtilityObject`'s `run` is what **Run Editor Utility** calls; `EditorUtilitySubsystem::try_run_class` runs it from code.

`AssetActionUtility`, `ActorActionUtility` (the Content Browser's and the level's **Scripted Actions** menus) and `EditorUtilityWidget` (an editor tab drawn by a Widget Blueprint) can be the parent of a Rust class too. The editor finds them as Blueprint assets: create a Blueprint whose parent is the Rust class, as for a [Blueprint child](../classes/blueprint_child.md), and put the logic in Rust. That path has not been tested yet.

## Editor Python and Rust classes

Editor Python reaches a Rust class as it reaches a Blueprint's: through reflection, by name.

```python
import unreal

note = unreal.load_asset("/Game/Dialogue/Intro")
note.get_editor_property("lines")
note.set_editor_property("speaker", "Ana")
note.call_method("Play", ())
```

| From Python | Works |
|---|---|
| `get_editor_property` and `set_editor_property`, with the Unreal name (`"Speaker"`) or the Rust one (`"speaker"`) | yes |
| `call_method("Play", args)`, with the function's Unreal name | yes |
| `call_method("play", ...)`, with the Rust name | no |
| attributes and methods (`note.speaker`, `note.play()`) | no: Python has no generated type for Rust classes, as for Blueprint ones |
