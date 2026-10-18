# Console commands

Rusteal adds two commands to the editor's console.

## Opening the console

Either of these works:

- press the `` ` `` key (the one under `Esc`) with the mouse over the level viewport;
- type in the **Cmd** box at the bottom of the **Output Log** tab.

> **Screenshot:** the Cmd box at the bottom of the Output Log tab with `Rusteal.Reload` typed in. *(images/iteration/output_log_cmd.png)*

## The commands

| Command | What it does |
|---|---|
| `Rusteal.Reload` | Reloads every Rust library of the project: the game's and each Rust plugin's. |
| `Rusteal.Reload <Name>` | Reloads one library: the game's is named after the project, a plugin's after the plugin, a plugin's editor library after its module (`<Plugin>Editor`). |
| `Rusteal.AutoReload 0` | Stops reloading a library when `rusteal build` deploys it again. `Rusteal.Reload` still works. |
| `Rusteal.AutoReload 1` | Reloads a library when it is deployed again. This is the default. |

## Use case: reloading when you choose

**The goal:** build several times without the editor reloading after each build, for example while a Play session must not be interrupted.

1. In the console, run `Rusteal.AutoReload 0`.
2. Build as many times as you want with `rusteal build`.
3. When you are ready, run `Rusteal.Reload`.
4. Run `Rusteal.AutoReload 1` to go back to reloading on every build.

## Use case: reloading one plugin

**The goal:** reload a plugin's library and leave the game's alone.

```text
Rusteal.Reload Inventory
```

## Your own commands

A game can add its own console commands and variables from Rust. See [Console commands and variables](../systems/console.md).
