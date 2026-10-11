# tour

The pieces of the Intro To Unreal tour: `BP_TextSwitcher`, text that changes in game; `BP_Titles`, the station numbers; animated `BP_KeyboardKey`s and `BP_MouseKey`s; `BP_GizmoDemo`, the move, rotate and scale gizmos at work; `BP_TemplateCube`; `BP_SplinePath`, decal arrows along a spline; and `BP_UI_Update`, editor panels shown in game.

```bash
rusteal new MyGame --template third-person --feature tour
rusteal plugin new Tour --template tour   # in an existing project
```

It becomes the `Tour` plugin, in `Plugins/Tour/`. It brings the engine's `LevelPrototyping` pack when the project lacks it.

## Trying it

Place a BP_KeyboardKey (Plugins > Tour Content), set its Key, and a BP_GizmoDemo with a Selection, then press Play.

> **Screenshot:** the tour feature in play. *(images/features/tour.png)*

## Its classes

The Rust classes are in `Plugins/Tour/Rust/tour/src/`:

| Class | Parent | File |
|---|---|---|
| `EditorHintWidget` | `UserWidget` | `src/editor_hint.rs` |
| `TourSubsystem` | `RustealWorldSubsystem` | `src/editor_hint.rs` |
| `EditorHintTrigger` | `Actor` | `src/editor_hint.rs` |
| `GizmoDemo` | `Actor` | `src/gizmo_demo.rs` |
| `KeyboardKeyWidget` | `UserWidget` | `src/keys.rs` |
| `MouseWidget` | `UserWidget` | `src/keys.rs` |
| `KeyboardKey` | `Actor` | `src/keys.rs` |
| `MouseKey` | `Actor` | `src/keys.rs` |
| `SplinePath` | `Actor` | `src/spline_path.rs` |
| `TemplateCube` | `Actor` | `src/template_cube.rs` |
| `TextSwitcher` | `Actor` | `src/texts.rs` |
| `StationTitle` | `Actor` | `src/texts.rs` |

Its Blueprints, `BP_GizmoDemo`, `BP_KeyboardKey`, `BP_MouseKey`, `BP_SplinePath`, `BP_TemplateCube`, `BP_TextSwitcher`, `BP_Titles` and `BP_UI_Update`, children of those classes holding the meshes, materials and widget layouts, are in the plugin's content: in the **Content Browser**, click **Settings** and check **Show Plugin Content**, then open **Plugins > Tour Content**.

Recipes that use it: [UI](../recipes/ui.md).
