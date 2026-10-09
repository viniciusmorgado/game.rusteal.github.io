# Saving the game

**The goal:** keep the player's progress between sessions.

A save game is a Rust class whose parent is `SaveGame`, written to and read from a slot with the engine's `GameplayStatics`:

```rust
use bindings::engine::{GameplayStatics, SaveGame};

#[uclass(parent = SaveGame)]
pub struct Progress {
    #[uproperty(SaveGame)]
    unlocked_level: i32,

    #[uproperty(SaveGame)]
    best_times: UeArray<f32>,

    #[uproperty(SaveGame)]
    player_name: String,
}

#[uclass_impl]
impl Progress {}

const SLOT: &str = "progress";

pub fn save_progress(level: i32) -> RustealResult<()> {
    let save = GameplayStatics::create_save_game_object(SubclassOf::<Progress>::base().upcast());
    let mut progress = Progress::from_obj(save)?;
    progress.set_unlocked_level(level);

    if !GameplayStatics::save_game_to_slot(save, SLOT, 0) {
        return Err(RustealError::InvalidOperation("the save failed".into()));
    }

    Ok(())
}

pub fn load_progress() -> Option<Progress> {
    if !GameplayStatics::does_save_game_exist(SLOT, 0) {
        return None;
    }

    Progress::from_obj(GameplayStatics::load_game_from_slot(SLOT, 0)).ok()
}
```

- `create_save_game_object` makes an empty save of the class; `save_game_to_slot` writes all its properties to the slot; `load_game_from_slot` reads it back as a new object.
- The second argument of each is the user index, `0` for a single player.
- Saves go to `Saved/SaveGames/<slot>.sav` in the project during development, and to the platform's save location in a packaged game.

## The SaveGame specifier

`save_game_to_slot` writes a `SaveGame` object whole, flagged or not. The `SaveGame` specifier matters when the object is serialized some other way: an archive that is a save game archive (the usual way to save actors' state, for example in a save system that walks the level's actors) writes only the properties that carry it. Marking the fields keeps them right in both cases.

```rust
#[uclass(parent = Actor)]
pub struct Chest {
    #[uproperty(EditAnywhere, SaveGame)]
    b_opened: bool,
}
```

## Hot reload

A hot reload keeps the save game class's properties, as for any class. A save written with an older shape of the class loads the properties that still exist under the same name, as in C++.
