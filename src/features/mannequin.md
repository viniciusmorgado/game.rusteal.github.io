# mannequin

Mannequins that move like a character: `CopycatAnimInstance` is a parent for Animation Blueprints that copies a character's movement (ground speed, direction, moving, falling), the player's unless told which, and `BP_CopycatMannequin` is the Intro To Unreal template's mannequin that walks, runs and jumps with the player. `ReachingMannequin` reaches for a `ReachTarget` placed in the level with its right hand, a two-bone IK solved in Rust.

```bash
rusteal new MyGame --template third-person --feature mannequin
rusteal plugin new Mannequin --template mannequin   # in an existing project
```

It becomes the `Mannequin` plugin, in `Plugins/Mannequin/`. It brings the engine's `Characters` pack (`Content/Characters/Mannequins`) when the project lacks it; the third person and first person templates already have it.

## Trying it

Place a BP_CopycatMannequin, and a ReachingMannequin with a ReachTarget as its Target (Plugins > Mannequin Content, Place Actors), then press Play.

> **Screenshot:** the mannequin feature in play. *(images/features/mannequin.png)*

## Its classes

The Rust classes are in `Plugins/Mannequin/Rust/mannequin/src/`:

| Class | Parent | File |
|---|---|---|
| `CopycatAnimInstance` | `AnimInstance` | `src/copycat.rs` |
| `CopycatMannequin` | `Actor` | `src/reach.rs` |
| `ReachTarget` | `Actor` | `src/reach.rs` |
| `ReachingMannequin` | `Actor` | `src/reach.rs` |

Its Blueprints, `ABP_Copycat` and `BP_CopycatMannequin`, children of those classes holding the meshes, materials and widget layouts, are in the plugin's content: in the **Content Browser**, click **Settings** and check **Show Plugin Content**, then open **Plugins > Mannequin Content**.

## A mannequin that copies the player

1. Create a project with `--feature mannequin`, or add the plugin to an existing one.
2. In the **Content Browser**, right-click and choose **Animation > Animation Blueprint**; pick the **SK_Mannequin** skeleton and, as the **Parent Class**, **CopycatAnimInstance**.
3. Open it. In the **My Blueprint** panel, click the gear icon and check **Show Inherited Variables**: **Ground Speed**, **Direction**, **Should Move**, **Is Falling**, **Velocity** and **Character** appear under **Copycat**.
4. In its **AnimGraph**, blend the idle, walk and run animations by **Ground Speed** and **Direction**, and switch to a jump loop when **Is Falling** is set.
5. Drag **SKM_Manny_Simple** from `Content/Characters/Mannequins/Meshes` into the level.
6. In its **Details** panel, set **Animation Mode** to **Use Animation Blueprint**.
7. Set **Anim Class** to the new Animation Blueprint.
8. Press **Play** and walk: the mannequin walks, runs and jumps with you.

> **Screenshot:** a copycat mannequin running beside the player in the third person level. *(images/getting_started/feature_copycat.png)*

The copying is plain Rust in `Plugins/Mannequin/Rust/mannequin/src/copycat.rs`: the speed ignores the vertical axis, the mannequin moves only when the speed is above 0.01 and the character accelerates, and the direction is kept within 45 degrees when the character turns toward its movement.

Recipes that use it: [Animation](../recipes/animation.md).
