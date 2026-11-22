# How to read this book

Most pages are written as use cases. Each one starts from something you want to do in a game and shows how to do it with Rusteal:

- **The goal**: what the use case achieves in the game.
- **The code**: the Rust you write, complete enough to compile in a project made by `rusteal new`.
- **In the editor**: what you do in Unreal Editor, step by step, with the names the editor shows.
- **Notes**: limits, alternatives and what to watch for.

## Code conventions

The examples import from two places:

- `rusteal_runtime`, the runtime crate: the attributes (`uclass`, `uclass_impl`, `ustruct`), the logging macro, `method`, which turns a Rust method into the callback that input, delegates and timers call, and the types every project uses (`UObjectRef`, `RustealResult`, `OwnedStruct`...).
- `bindings`, the crate Rusteal generates from your engine: one Rust type per engine class, struct and enum, grouped by engine module (`bindings::engine`, `bindings::enhanced_input`, `bindings::ai`...). `bindings::prelude` brings the common ones and the helper functions (`bind_action`, `create_widget_of_class`, `FVector::from_dvec3`...).

Engine names become Rust names in a fixed way. Classes lose their C++ prefix (`AActor` is `Actor`, `UCharacterMovementComponent` is `CharacterMovementComponent`); structs and enums keep it (`FVector`, `ETriggerEvent`); functions and properties are in snake_case (`K2_GetActorLocation` is `k2_get_actor_location`). A class's functions come from a trait named after it with `Ext` at the end (`ActorExt`, `CharacterMovementComponentExt`), which you import with the class.

Optional parameters of engine functions, the ones with a default value in C++, are `Option`s in Rust: `Some(value)` passes a value, `None` keeps the default.

## Screenshots

Some steps are easier to follow with a picture of the editor. Where a picture belongs, the page has a placeholder like this one:

> **Screenshot:** what the picture shows. *(images/example.png)*

The file name in parentheses is where the image goes, under `src/images/`.
