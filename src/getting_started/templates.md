# Starting from a template

**The goal:** a playable project in a genre, with its gameplay already written in Rust, to build your game on.

```bash
rusteal new MyGame --template third-person
rusteal new MyGame --template third-person --variant combat
```

A template is one of the engine's templates with its gameplay written in Rust. The C++ classes are gone, the Rust crate in `Rust/` takes their place, and their Blueprint children, with the same names, have Rust parents. The engine's Blueprint-only templates (`vr`, `ar`, `intro`) keep their Blueprints, reparented to Rust classes and holding only data: meshes, materials, effects, input assets, widget layouts and Anim Blueprint graphs. Open the project and press **Play**: it plays like the engine's template.

The engine's templates come with variants, genre starting points that the C++ version puts all in one project. In Rusteal each variant is its own project: `--variant` picks one, and without it the project is the template alone. A variant project opens on the variant's level.

| Template | Variant | What it is |
|---|---|---|
| [`blank`](../templates/blank.md) (default) | | The engine's Blank template and a `HelloActor` in Rust. |
| [`first-person`](../templates/first-person.md) | | The First Person template: its character, camera manager, game mode and player controller in Rust (keyboard, mouse, gamepad, touch). |
| [`first-person`](../templates/first-person.md#horror) | `horror` | A dark level explored with a flashlight, and a sprint that runs on stamina, its meter on screen. |
| [`first-person`](../templates/first-person.md#shooter) | `shooter` | An arena shooter: weapon pickups from a data table, projectiles and grenades, team scores, and NPCs run by a StateTree that sense, snipe and shoot. |
| [`top-down`](../templates/top-down.md) | | The Top Down template: its character, game mode and point and click player controller in Rust (mouse, touch). |
| [`top-down`](../templates/top-down.md#strategy) | `strategy` | A real-time strategy start: units selected with clicks, boxes and touches, sent along the navigation mesh by EnvQueries, and interacting with the units they reach. |
| [`top-down`](../templates/top-down.md#twin-stick) | `twin-stick` | A twin stick shooter: projectiles, dashes and area attacks against waves of enemies run by a StateTree, pickups and a combo score. |
| [`intro`](../templates/intro.md) | | The Intro To Unreal template: its guided tour level, whose stations are the four features (see [Adding features](features.md)), and its first person character in Rust. Follow the tour from the first station. |
| [`third-person`](../templates/third-person.md) | | The Third Person template: its character, game mode and player controller in Rust (keyboard, mouse, gamepad, touch). |
| [`third-person`](../templates/third-person.md#combat) | `combat` | Melee combat: combo and charged attacks, enemies run by a StateTree, spawners, checkpoints, damageable props. |
| [`third-person`](../templates/third-person.md#platforming) | `platforming` | Double jump, wall jump, coyote time, a dash, local multiplayer and respawn. |
| [`third-person`](../templates/third-person.md#side-scrolling) | `side-scrolling` | A side view platformer: soft platforms, jump pads, moving platforms, pickups with a counter and an NPC run by a StateTree. |
| [`vehicle`](../templates/vehicle.md) | | The Vehicle template: a sports car and an offroad car on Chaos Vehicles, their game mode, player controller and speed and gear HUD in Rust (keyboard, gamepad, steering wheel, touch). |
| [`vehicle`](../templates/vehicle.md#offroad) | `offroad` | The offroad car on an offroad level, its game mode ready for local split screen players. |
| [`vehicle`](../templates/vehicle.md#time-trial) | `time-trial` | A race track with a start countdown, gates to pass in order, laps and the best lap time. |
| [`ar`](../templates/ar.md) | | The Handheld AR template on ARCore and ARKit: its pawn (touch gestures to pick a detected plane, place the object on it, move, rotate and scale it; light estimation), plane and placeable actors and its styled menus in Rust. It runs in the editor; Rusteal does not build for Android or iOS yet. |
| [`vr`](../templates/vr.md) | | The Virtual Reality template on OpenXR: its pawn (teleport, snap turn, hand poses), grabbable objects, pistol, 3D menu, passthrough and spectator camera in Rust. Play it with **VR Preview**. |

Names are matched loosely (`side-scrolling` or `SideScrolling`), and `rusteal new` with an unknown template or variant lists them all, with the features any template can take (see [Adding features](features.md)).

Each template has a page of its own, linked from the table: how to play it, its Rust classes and its variants.

![The third-person template's combat variant in play](../images/templates/third-person-combat.png)

## Reading a template

The templates are the largest body of Rusteal code there is, and the [Gameplay recipes](../recipes/index.md) in this book are taken from them. In a template project:

- the Rust classes are in `Rust/<crate>/src/`, one per file, with a variant's classes in a module named after it (`src/combat/`, `src/shooter/`);
- the tunable numbers and the rules that do not call the engine are in the `model.rs` files, with their tests;
- the Blueprint children that assign meshes, animations, input actions and widgets are in `Content/`, under the same names as in the engine's template (`BP_ThirdPersonCharacter`, `BP_ThirdPersonPlayerController`...).

To see where a Rust class is used, open its Blueprint child in the editor: **Class Settings** shows the Rust class as the **Parent Class**, and the **Details** panel shows the properties the Rust class declares.

> **Screenshot:** BP_ThirdPersonCharacter open in the Blueprint editor, with Class Settings showing ThirdPersonCharacter as the Parent Class. *(images/getting_started/bp_parent_class.png)*
