# checkpoints

Spawn points that save the player's progress: walking past a `BP_SpawnPoint` writes it to a save (`CheckpointSaveData`), and a world subsystem, `CheckpointSubsystem`, brings the player back to the last one when the level starts, or when `RespawnPlayer` is called.

```bash
rusteal new MyGame --template third-person --feature checkpoints
rusteal plugin new Checkpoints --template checkpoints   # in an existing project
```

It becomes the `Checkpoints` plugin, in `Plugins/Checkpoints/`. It needs no engine content of its own.

## Trying it

Place BP_SpawnPoint actors (Plugins > Checkpoints Content) along the level, press Play, walk past one, stop and play again: you start there.

> **Screenshot:** the checkpoints feature in play. *(images/features/checkpoints.png)*

## Its classes

The Rust classes are in `Plugins/Checkpoints/Rust/checkpoints/src/`:

| Class | Parent | File |
|---|---|---|
| `CheckpointSaveData` | `SaveGame` | `src/save_data.rs` |
| `SpawnPoint` | `Actor` | `src/spawn_point.rs` |
| `CheckpointSubsystem` | `RustealWorldSubsystem` | `src/subsystem.rs` |

Its Blueprint, `BP_SpawnPoint`, a child of those classes holding the meshes and materials, is in the plugin's content: in the **Content Browser**, click **Settings** and check **Show Plugin Content**, then open **Plugins > Checkpoints Content**.

Recipes that use it: [Saving the game](../recipes/save_game.md).
