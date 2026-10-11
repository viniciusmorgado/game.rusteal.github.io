# ar

The engine's Handheld AR template on ARCore and ARKit: its pawn reads touch gestures to pick a detected plane, place the object on it, then move, rotate and scale it, and applies the light estimation; the plane and placeable actors and the styled menus are Rust classes. It runs in the editor; Rusteal does not build for Android or iOS yet (see [Known limits](../reference/limits.md)).

```bash
rusteal new MyGame --template ar
```

From `TP_HandheldARBP`, the engine template it is made of.

## Playing it

Open the project and press **Play**: tap Begin Scan, tap a plane, tap it again to place the object, drag with one finger to move it, twist or pinch with two to rotate or scale it.

> **Screenshot:** the ar template's start menu in the editor. *(images/templates/ar.png)*

## Its classes

The Rust classes are in `Rust/<crate>/src/`; their Blueprint children, under the engine template's names, are in `Content/`.

| Class | Parent | File |
|---|---|---|
| `CapsuleButton` | `StylizedUI` | `src/buttons.rs` |
| `ToggleButton` | `StylizedUI` | `src/buttons.rs` |
| `ARGameMode` | `GameModeBase` | `src/game_mode.rs` |
| `ARPlayerController` | `PlayerController` | `src/game_mode.rs` |
| `StartMenu` | `StylizedUI` | `src/menus.rs` |
| `BottomMenu` | `StylizedUI` | `src/menus.rs` |
| `InfoMenu` | `StylizedUI` | `src/menus.rs` |
| `OptionMenu` | `StylizedUI` | `src/menus.rs` |
| `MainMenu` | `UserWidget` | `src/menus.rs` |
| `InteractionHUD` | `UserWidget` | `src/menus.rs` |
| `ARPawn` | `Pawn` | `src/pawn.rs` |
| `ProductAssetRef` | `ActorComponent` | `src/placeable.rs` |
| `Placeable` | `Actor` | `src/placeable.rs` |
| `Plane` | `Actor` | `src/plane.rs` |
| `StylizedUI` | `UserWidget` | `src/styles.rs` |
| `StructIcon` | a struct | `src/styles.rs` |
| `StructStyle` | a struct | `src/styles.rs` |
Recipes taken from it: [Augmented reality](../recipes/ar.md).
