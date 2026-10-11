# vr

The engine's Virtual Reality template on OpenXR: its pawn teleports along the navigation mesh, snap turns and poses the hands; grabbable objects, a pistol and its projectiles, a 3D menu, passthrough and a spectator camera are Rust classes. Play it with **Play > VR Preview** and an OpenXR headset.

```bash
rusteal new MyGame --template vr
```

From `TP_VirtualRealityBP`, the engine template it is made of.

## Playing it

Open the project and press **Play**: right thumbstick forward aims a teleport, left thumbstick snaps the view, grips grab, triggers shoot the pistol, the menu button opens the menu.

> **Screenshot:** the vr template's hands and teleport arc in VR Preview. *(images/templates/vr.png)*

## Its classes

The Rust classes are in `Rust/<crate>/src/`; their Blueprint children, under the engine template's names, are in `Content/`.

| Class | Parent | File |
|---|---|---|
| `XRGameMode` | `GameModeBase` | `src/game_mode.rs` |
| `GrabComponent` | `SceneComponent` | `src/grab_component.rs` |
| `GrabbableSmallCube` | `Actor` | `src/grabbable.rs` |
| `MannequinsXR` | `SkeletalMeshComponent` | `src/hand.rs` |
| `HandAnimInstance` | `AnimInstance` | `src/hand.rs` |
| `InputModifierXAxisPositiveOnly` | `InputModifier` | `src/input_modifier.rs` |
| `Menu` | `Actor` | `src/menu.rs` |
| `MenuWidget` | `UserWidget` | `src/menu_widget.rs` |
| `Passthrough` | `Actor` | `src/passthrough.rs` |
| `XRPawn` | `Pawn` | `src/pawn.rs` |
| `Pistol` | `Actor` | `src/pistol.rs` |
| `Projectile` | `Actor` | `src/projectile.rs` |
| `VRSpectator` | `Actor` | `src/spectator.rs` |
| `TeleportVisualizer` | `Actor` | `src/teleport_visualizer.rs` |
Recipes taken from it: [Virtual reality](../recipes/vr.md).
