# intro

The engine's Intro To Unreal template: a guided tour level whose stations show the editor and the engine at work. The stations are the four features any template can take, [mannequin](../features/mannequin.md), [checkpoints](../features/checkpoints.md), [interactables](../features/interactables.md) and [tour](../features/tour.md), which `rusteal new` adds by itself; the level's Blueprints are children of their Rust classes. The first person character is the [first-person](first-person.md) template's, with `IntroGameMode` as the level's game mode; the checkpoints' subsystem brings the player back to the last spawn point.

```bash
rusteal new MyGame --template intro
```

From `TP_UEIntro_BP`, the engine template it is made of.

## Playing it

Open the project and press **Play**: WASD / left stick move, mouse / right stick look, Space / A jump; follow the tour from the first station.

> **Screenshot:** the intro template's tour level, at its first station. *(images/templates/intro.png)*

## Its classes

The Rust classes are in `Rust/<crate>/src/`; their Blueprint children, under the engine template's names, are in `Content/`.

| Class | Parent | File |
|---|---|---|
| `FirstPersonCameraManager` | `PlayerCameraManager` | `src/camera_manager.rs` |
| `FirstPersonCharacter` | `Character` | `src/character.rs` |
| `FirstPersonGameMode` | `GameModeBase` | `src/game_mode.rs` |
| `IntroGameMode` | `GameModeBase` | `src/game_mode.rs` |
| `FirstPersonPlayerController` | `PlayerController` | `src/player_controller.rs` |
Recipes taken from it: [Saving the game](../recipes/save_game.md), [Game flow](../recipes/game_flow.md), [Animation](../recipes/animation.md).
