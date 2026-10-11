# interactables

A trigger and what it drives: `BP_DoorTrigger` is a switch that broadcasts its `Trigger` event, which toggles `BP_Light`s and unlocks a `BP_DoorFrame_Unlockable`. Around them: `BP_ConfettiTrigger`, a Niagara burst; `BP_SequenceTrigger`, which plays a level sequence; `BP_Metasound`, a MetaSound with a seed parameter; `BP_DangleBlocks`, cubes hanging from physics constraints.

```bash
rusteal new MyGame --template third-person --feature interactables
rusteal plugin new Interactables --template interactables   # in an existing project
```

It becomes the `Interactables` plugin, in `Plugins/Interactables/`. It brings the engine's `LevelPrototyping` pack (door meshes, cubes) when the project lacks it.

## Trying it

Place a BP_DoorTrigger and a BP_DoorFrame_Unlockable (Plugins > Interactables Content), set the door's Trigger to the trigger, press Play and step on it.

> **Screenshot:** the interactables feature in play. *(images/features/interactables.png)*

## Its classes

The Rust classes are in `Plugins/Interactables/Rust/interactables/src/`:

| Class | Parent | File |
|---|---|---|
| `ConfettiTrigger` | `Actor` | `src/confetti_trigger.rs` |
| `DangleBlocks` | `Actor` | `src/dangle_blocks.rs` |
| `UnlockableDoor` | `Actor` | `src/door.rs` |
| `TriggerLight` | `Actor` | `src/light.rs` |
| `SeededSound` | `Actor` | `src/seeded_sound.rs` |
| `SequenceTrigger` | `Actor` | `src/sequence_trigger.rs` |
| `TriggerSwitch` | `Actor` | `src/trigger.rs` |

Its Blueprints, `BP_ConfettiTrigger`, `BP_DangleBlocks`, `BP_DoorFrame_Unlockable`, `BP_DoorTrigger`, `BP_Light`, `BP_Metasound` and `BP_SequenceTrigger`, children of those classes holding the meshes, materials and widget layouts, are in the plugin's content: in the **Content Browser**, click **Settings** and check **Show Plugin Content**, then open **Plugins > Interactables Content**.

Recipes that use it: [Game flow](../recipes/game_flow.md), [Physics](../recipes/physics.md), [Effects and sound](../recipes/effects.md).
