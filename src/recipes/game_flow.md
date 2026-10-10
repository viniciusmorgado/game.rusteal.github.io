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

## Use case: one actor switching others

**The goal:** stepping on a trigger switches the lights wired to it and unlocks a door, which the next step locks again.

The trigger declares an event and broadcasts it on every overlap:

```rust
#[uclass_impl]
impl TriggerSwitch {
    #[udelegate(name = "Trigger")]
    pub fn trigger(&self) {}

    #[ufunction(Override)]
    fn receive_actor_begin_overlap(&mut self, _other_actor: UObjectRef<Actor>) {
        self.trigger();
        self.set_is_on(!self.is_on());
    }
}
```

What it drives holds a reference to it, set on each placed actor in the **Details** panel, and binds one of its functions when play begins:

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    let me = self.as_ref();

    if let Ok(trigger) = TriggerSwitch::from_obj(self.trigger()) {
        let _ = trigger.trigger_add_ufunction(&me, "Unlock");
    }
}

#[ufunction]
fn unlock(&mut self) {
    self.set_unlocked(!self.unlocked());

    if self.unlocked() {
        self.open_door();
    } else {
        self.close_door();
    }
}
```

The `interactables` feature has the trigger (`BP_DoorTrigger`), lights (`BP_Light`), a door (`BP_DoorFrame_Unlockable`) and a confetti burst (`BP_ConfettiTrigger`) wired this way.

## Use case: a door that slides open

**The goal:** a door that opens in 0.3 seconds as a player comes near and closes when the last one leaves, eased like the Blueprint timeline it replaces.

The timeline's curve is a few keys with their tangents, evaluated in Rust, and its playhead moves in the actor's tick:

```rust
const DOOR_CONTROL: [Key; 2] = [
    curve::key(0.0, 0.0, 0.0, 7.314_739),
    curve::key(0.3, 1.0, 0.0, 0.0),
];

#[ufunction(Override)]
fn receive_tick(&mut self, delta_seconds: f32) {
    let mut timeline = self.timeline();

    if !timeline.is_playing() {
        return;
    }

    let position = timeline.advance(f64::from(delta_seconds), DOOR_CONTROL_LENGTH);
    self.set_timeline(timeline);
    self.move_doors(curve::eval(&DOOR_CONTROL, position));
}
```

`timeline.play()` runs it forward from where it is, `timeline.reverse()` back, so a player who turns around mid-way closes the door from there. The door's frame and leaves are built in the construction script (`user_construction_script`) from its **Door Size**, **Split Door** and **Door Thickness**, with `add_component_by_class`.

## Use case: changing level

```rust
GameplayStatics::open_level(self.as_ref().upcast_to(), FName::new("Lvl_Arena").handle(), Some(true), None);
```

Everything in the old level is destroyed, the game mode included; what must survive a level change lives in a game instance subsystem (see [Subsystems](../systems/subsystems.md)) or a save.
