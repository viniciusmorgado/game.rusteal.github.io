# Blueprint children

**The goal:** a Blueprint whose parent is a Rust class, to assign the assets the class needs (meshes, sounds, input actions, widget classes) and to tune its values, without touching Rust.

This is how every Rusteal template works: the Rust class declares the properties, and a Blueprint child named `BP_<Class>` fills them in. The game then uses the Blueprint, not the Rust class directly.

## Creating the Blueprint

1. In the **Content Browser**, open the folder the Blueprint goes in.
2. Right-click an empty area and choose **Blueprint Class**.
3. In the **Pick Parent Class** window, expand **All Classes**.
4. Type the Rust class's name in the search box (`Turret`).
5. Select the class in the list and click **Select**.
6. Name the new asset `BP_Turret` and press **Enter**.

> **Screenshot:** the Pick Parent Class window with All Classes expanded and the Rust class found by the search. *(images/classes/pick_parent_class.png)*

## Filling in the properties

1. Double-click **BP_Turret** to open it.
2. Click **Class Defaults** in the toolbar.
3. In the **Details** panel, find the Rust class's properties under their categories, and set them: pick the sound in **Fire Sound**, the projectile Blueprint in **Projectile Class**.
4. Click **Compile**, then **Save**.

> **Screenshot:** the Class Defaults of BP_Turret with Fire Sound and Projectile Class assigned. *(images/classes/class_defaults_assigned.png)*

The components the Rust class declares are in the **Components** panel: select one to give it a mesh or a material.

## Using the Blueprint

Place **BP_Turret** in the level by dragging it from the **Content Browser** into the viewport, or spawn it from Rust through a `SubclassOf<T>` property that holds it (see [Spawning actors](../engine_api/spawning.md)).

## Game mode, pawn and controller

A game mode names the pawn and the controller the game uses. With all three in Rust:

1. Create a Blueprint child of each: `BP_MyGameMode`, `BP_MyCharacter`, `BP_MyPlayerController`.
2. Open **BP_MyGameMode**, click **Class Defaults**, and in the **Details** panel, under **Classes**, set **Default Pawn Class** to **BP_MyCharacter** and **Player Controller Class** to **BP_MyPlayerController**. **Compile** and **Save**.
3. Open the level, and in the **World Settings** tab (**Window > World Settings**), set **GameMode Override** to **BP_MyGameMode**.

Or, for every level of the project: **Edit > Project Settings**, **Maps & Modes**, **Default GameMode**.

> **Screenshot:** the World Settings tab with GameMode Override set to the game mode Blueprint. *(images/classes/world_settings_game_mode.png)*

## Rules for Blueprint children

- **Data, not logic, for what Rust does.** A Blueprint child that implements an event the Rust class implements replaces the Rust code for it. Keep the Blueprint's graph for the `BlueprintImplementableEvent`s the Rust class declares for it.
- **Values changed in the Blueprint win.** A value set in the Blueprint's **Class Defaults** overrides the Rust default. The arrow next to a changed value resets it.
- **Hot reload keeps them.** When the Rust class changes shape, its Blueprint children move to the new class with their values. See [Hot reload](../iteration/hot_reload.md).
