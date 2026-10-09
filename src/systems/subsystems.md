# Subsystems

**The goal:** a system with one instance per game, per level or per player, created and destroyed by the engine, that any code can find: a wave director, a quest log, a score keeper, a music manager.

The engine creates one subsystem object for every class deriving from a subsystem base. A Rust subsystem derives from one of Rusteal's bases, which turn the engine's C++ virtuals into events:

| Parent | One per | Events |
|---|---|---|
| `RustealGameInstanceSubsystem` | game session (survives level changes) | `receive_initialize`, `receive_deinitialize`, `receive_should_create_subsystem` |
| `RustealWorldSubsystem` | world: each level, in the game and in Play In Editor | the same, plus `receive_post_initialize`, `receive_world_begin_play`, and `receive_tick` when it wants to tick |
| `RustealLocalPlayerSubsystem` | local player | the same as the game's, plus `receive_player_controller_changed` |
| `RustealEngineSubsystem` | process | the same as the game's |

They come from `bindings::rusteal`.

## Use case: a wave director

**The goal:** a system that starts enemy waves when the level begins and counts down between them.

```rust
use bindings::rusteal::{RustealWorldSubsystem, RustealWorldSubsystemExt};

#[uclass(parent = RustealWorldSubsystem)]
pub struct WaveDirector {
    #[uproperty]
    wave: i32,

    countdown: f32,
}

#[uclass_impl]
impl WaveDirector {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        self.as_ref().checked()?.set_wants_tick(true);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_world_begin_play(&mut self) {
        self.set_wave(1);
        self.set_countdown(10.0);
    }

    #[ufunction(Override)]
    fn receive_tick(&mut self, delta_seconds: f32) {
        let countdown = self.countdown() - delta_seconds;

        if countdown <= 0.0 {
            self.set_wave(self.wave() + 1);
            self.set_countdown(30.0);
        } else {
            self.set_countdown(countdown);
        }
    }

    #[ufunction(BlueprintPure)]
    pub fn current_wave(&self) -> i32 {
        self.wave()
    }
}
```

## Finding a subsystem

From Rust, through the engine's `SubsystemBlueprintLibrary`, with any object in the world as context:

```rust
use bindings::engine::{SubsystemBlueprintLibrary, WorldSubsystem};

impl WaveDirector {
    pub fn get(context: UObjectRef<Object>) -> RustealResult<Self> {
        let class = SubclassOf::<WaveDirector>::base().upcast_to::<WorldSubsystem>();
        Self::from_obj(SubsystemBlueprintLibrary::get_world_subsystem(context, class))
    }
}

let mut director = WaveDirector::get(self.as_ref().upcast_to())?;
```

`get_game_instance_subsystem`, `get_local_player_subsystem` and `get_engine_subsystem` find the other kinds.

## Choosing the kind

- State of the current level, gone when the level changes: **world**.
- State that lives across levels (the player's progress, settings loaded once, a connection): **game instance**.
- State per player in split screen: **local player**.

A world subsystem exists in the editor's own world too only when it asks for it (`bCreateInEditorWorlds`); by default it exists in the game and in Play In Editor.
