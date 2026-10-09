# Introduction

Rusteal lets you write Unreal Engine 5.8 games in Rust. Your gameplay code is a Rust crate, compiled into a library that a small Unreal plugin loads. The classes you declare in it are real Unreal classes: they show up in the editor, they can be placed in a level, Blueprints can derive from them, and they are saved, cooked and packaged like classes written in C++.

This book is for the game developer. It shows how to use Rusteal to make a game: how to create a project, how to write actors, characters and systems in Rust, how to work with the editor and Blueprints, how to iterate without restarting the editor, and how to ship. It does not explain how Rusteal itself is built.

## What stays in the editor, what goes in Rust

Rusteal does not replace the editor. A Rusteal game is an Unreal project like any other, and the work is split the way a C++ project splits it:

| In Rust | In the editor |
|---|---|
| Gameplay classes: actors, characters, controllers, game modes, components, subsystems | Levels, lighting, materials, meshes, animations |
| Their properties, components and default values | Blueprint children that assign assets to those properties |
| Rules, state, AI logic, systems | UMG widget layouts, input actions and mapping contexts |
| Console commands, settings, save data | Data tables and data assets filled with content |

A Rust class declares a property, and a Blueprint child of that class picks the mesh, the sound or the input action that goes into it. That is how Epic's own C++ templates work, and Rusteal's templates are those same templates with the C++ replaced by Rust.

## How the book is organized

- **Getting started** installs the tools and creates a first project.
- **The rusteal CLI** covers every command you run while making the game.
- **The iteration loop** covers hot reload, `rusteal watch`, the console commands and testing.
- **Classes** covers what you write: classes, properties, components, functions, events, interfaces and structs.
- **The engine API** covers how Rust code talks to the engine: object references, spawning, assets, timers, traces and delegates.
- **Gameplay recipes** are complete use cases taken from the templates: movement, cameras, combat, AI, UI and more.
- **Systems** covers subsystems, settings, data assets, console commands and background work.
- **Plugins** covers gameplay and editor plugins written in Rust.
- **Shipping** covers packaging the game.
- **Reference** collects the attributes, the types, the event names, the known limits and the common problems.

## Status

Rusteal is in early development. Its API changes between versions, and many engine features are not covered yet. Each chapter says what works today, and [Known limits](../reference/limits.md) lists what does not.
