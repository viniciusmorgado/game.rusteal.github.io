# Hot reload

**The goal:** change Rust code and see it in the running editor, without restarting it.

```bash
rusteal build
```

When the build deploys the library, the editor reloads it on its own, and a notification says so. Press **Play** and the new code runs. Nothing else is needed: no button in the editor, no restart.

> **Screenshot:** the editor notification shown after a reload. *(images/iteration/reload_notification.png)*

## What a reload keeps and what it changes

| You changed | After the reload |
|---|---|
| The body of a function or event | The new code runs. Classes, objects and their properties stay as they are. |
| A `#[uproperty]`: added, removed, renamed, retyped | The class is created again with the new properties, and its objects move to it, keeping the values of the properties that still exist. |
| A `#[component]` | The class is created again; its objects move to it. |
| A `#[ufunction]` or `#[udelegate]`: added, removed, signature changed | The class is created again; Blueprints that use it are recompiled. |
| A class's parent, its `implements` or its `config` | The class is created again. |
| A new `#[uclass]` | The class appears, ready to be placed or derived from. |
| A Rust-only field (no attribute) | Starts over from its `Default` value, on every reload, changed or not. |
| A `#[uinterface]` (after 0.7.0): a function added, removed, signature changed | The interface keeps its class and gets the new functions; the Blueprints implementing or calling it are recompiled, and the Rust classes implementing it are created again. |
| A `#[ustruct]` | Logged as changed; the struct keeps its old fields until the editor restarts. |

When a class is created again, every object of it moves to the new class: the actors placed in the level, the Blueprint children of the class, and the Rust classes that derive from it. Their property values are kept.

## Blueprints that use a changed class

A Blueprint that calls a function you removed, or reads a property you removed, gets a compile error on that node after the reload, as it would after the same change to a C++ class. Open the Blueprint, and either fix the node or put the function back and build again: the error clears.

> **Screenshot:** a Blueprint graph with a node in error after its Rust function was removed. *(images/iteration/blueprint_node_error.png)*

## Rust-only fields start over

A field with no attribute is Rust's alone, kept outside the Unreal object, and a reload drops it. Keep what must survive a reload in `#[uproperty]` fields: they live in the object's memory, and the reload keeps them. A Rust-only field is for state that can be rebuilt (a cache, a path, a timer you set again in `receive_begin_play`).

## When a restart is needed

- A changed `#[ustruct]`: the editor keeps the old struct until it restarts.
- A change to `rusteal.toml`, or a new engine version: run `rusteal build --all`, then restart the editor.
- A new Rusteal version after `rusteal upgrade`: restart the editor.
- Outside the editor (a game started with `-game`), a changed class or interface cannot be replaced: it keeps its old shape, and the log asks for a restart. Function bodies still update.
- On macOS, after a long session of reloads: macOS never unloads a Rust library, so each reload keeps the previous one in memory until the editor closes. A restart gives that memory back.

## Reloading by hand

The editor also reloads on request, from its console. See [Console commands](console.md).

## A reload during Play

The safest moment to reload is between Play sessions. A reload during Play drops the Rust-only fields of the objects in the game, and moves the objects of a class that is created again to the new class in the middle of the game.

## How long it takes

A reload that only changes code takes a few milliseconds. A reload that creates classes again takes longer, around a tenth of a second for a few classes, which is the engine moving the objects to the new classes. The Output Log ends each reload with how long it took.
