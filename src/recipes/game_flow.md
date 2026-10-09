# Game flow

## Use case: the game mode, pawn and controller in Rust

**The goal:** the level runs your Rust game mode, which spawns your character and controller.

```rust
use bindings::engine::GameModeBase;

#[uclass(parent = GameModeBase)]
pub struct MyGameMode {}

#[uclass_impl]
impl MyGameMode {}
```

The game mode's Blueprint child names the classes: **Default Pawn Class**, **Player Controller Class**, **HUD Class**. The level, or the project, uses that Blueprint. See [Blueprint children](../classes/blueprint_child.md), "Game mode, pawn and controller".

A game mode with rules overrides its events (`receive_begin_play`, `choose_player_start`) and holds the match's state, as the templates do with scores and counters. Any actor reaches it:

```rust
let game_mode = GameplayStatics::get_game_mode(self.as_ref().upcast_to());

if let Ok(mut game_mode) = MyGameMode::from_obj(game_mode) {
}
```

## Use case: choosing where each player starts

**The goal:** player one starts at the start tagged `Player0`, player two at `Player1`, and any start if none is tagged.

```rust
use bindings::engine::{Controller, KismetMathLibrary, PlayerStart};

#[uclass(parent = GameModeBase)]
pub struct MyGameMode {
    next_player_start: i32,
}

#[uclass_impl]
impl MyGameMode {
    #[ufunction(Override)]
    fn choose_player_start(&mut self, _player: UObjectRef<Controller>) -> UObjectRef<Actor> {
        let world = self.as_ref().upcast_to();
        let class = SubclassOf::<PlayerStart>::base().upcast_to::<Actor>();
        let tag = FName::new(&format!("Player{}", self.next_player_start()));

        let mut starts = GameplayStatics::get_all_actors_of_class_with_tag(world, class, tag.handle());
        self.set_next_player_start(self.next_player_start() + 1);

        if starts.is_empty() {
            starts = GameplayStatics::get_all_actors_of_class(world, class);
        }

        if starts.is_empty() {
            return UObjectRef::null();
        }

        let index = KismetMathLibrary::random_integer_in_range(0, starts.len() as i32 - 1);
        starts[index as usize]
    }
}
```

Tag a **Player Start** in the level by adding the tag to **Tags** in its **Details** panel, under **Actor**.

## Use case: local multiplayer

**The goal:** two to four players on one screen, split.

```rust
#[uclass(parent = GameModeBase)]
pub struct CoopGameMode {
    #[uproperty(EditDefaultsOnly, default = 2)]
    number_of_local_players: i32,
}

#[uclass_impl]
impl CoopGameMode {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        for _ in 2..=self.number_of_local_players() {
            GameplayStatics::create_player(self.as_ref().upcast_to(), Some(-1), Some(true));
        }
    }
}
```

`create_player` adds a local player with its own controller and pawn; the engine splits the screen. Each player controller adds its own mapping contexts and widgets (`add_to_player_screen` puts a widget on that player's part of the screen). `GameplayStatics::get_num_local_player_controllers` and `get_player_pawn(world, index)` reach each one.

## Use case: a respawn

See [Health, damage and death](combat.md), "Respawning the player".

## Use case: changing level

```rust
GameplayStatics::open_level(self.as_ref().upcast_to(), FName::new("Lvl_Arena").handle(), Some(true), None);
```

Everything in the old level is destroyed, the game mode included; what must survive a level change lives in a game instance subsystem (see [Subsystems](../systems/subsystems.md)) or a save.
