# Adding features

A feature is a piece of gameplay any template can take: a Rusteal plugin with its Rust classes, and the engine content they need. `--feature` adds it when the project is created, as many times as wanted:

```bash
rusteal new MyGame --template third-person --feature mannequin
```

| Feature | Plugin | What it is |
|---|---|---|
| `mannequin` | `Mannequin` | `CopycatAnimInstance`, a parent for Animation Blueprints that copies a character's movement: its ground speed, direction, whether it moves and whether it falls. It copies the player unless its **Source Character** names another character. |

Each feature becomes a plugin in `Plugins/<Plugin>/`, enabled in the `.uproject` and built with the project: its Rust classes are in `Plugins/<Plugin>/Rust/`, and `rusteal build` builds them with the game's. When the project lacks the engine content the feature uses, it is copied in: `mannequin` brings the `Characters` pack (`Content/Characters/Mannequins`), which the third person and first person templates already have.

An existing project takes a feature as a plugin:

```bash
rusteal plugin new Mannequin --template mannequin
```

A feature whose plugin carries assets keeps its plugin name, since its assets live under `/<Plugin>/`.

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
