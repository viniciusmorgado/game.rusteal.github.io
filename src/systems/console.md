# Console commands and variables

The console is a game's quickest debugging tool: commands to give items, skip waves, spawn enemies; variables to turn on god mode or tune a value while playing. Rust adds both. (For Rusteal's own commands, `Rusteal.Reload` and `Rusteal.AutoReload`, see [Console commands](../iteration/console.md).)

## Use case: a cheat command

**The goal:** `mygame.give_gold 500` in the console gives the player gold.

```rust
use bindings::core_ue::Object;
use bindings::engine::{GameplayStatics, World};
use bindings::prelude::*;
use rusteal_runtime::runtime::{CommandContext, RustealResult, UObjectRef, console};

fn startup() {
    let result = console::register_command(
        "mygame.give_gold",
        "mygame.give_gold <amount>",
        |args, context| {
            let amount: i32 = args.first().and_then(|a| a.parse().ok()).unwrap_or(100);

            if let Err(e) = give_gold(context, amount) {
                log::error!("[MyGame] give_gold: {e}");
            }
        },
    );

    if let Err(e) = result {
        log::error!("[MyGame] no console command: {e}");
    }
}

rusteal_runtime::on_load!(startup);

fn give_gold(context: &CommandContext, amount: i32) -> RustealResult<()> {
    let world: UObjectRef<World> = context.world()?;
    let pawn = GameplayStatics::get_player_pawn(world.upcast_to::<Object>(), 0);
    log::info!("[MyGame] giving {amount} gold to {}", pawn.get_name()?);
    Ok(())
}
```

- `console::register_command(name, help, closure)` adds the command. The closure receives the words after the command (`["500"]`) and a `CommandContext`, whose `world()` gives the world the command runs in, typed and checked: an error when the command runs with no world.
- `on_load!(function)` runs a function every time the library loads, hot reloads included. A library's commands and variables are removed when it unloads and registered again when it loads, so they always run the current code.

## Use case: a console variable

**The goal:** `mygame.god 1` makes the player invulnerable.

```rust
use rusteal_runtime::runtime::ConsoleVariable;

pub const GOD_MODE: ConsoleVariable<bool> = ConsoleVariable::new("mygame.god");

fn startup() {
    let _ = GOD_MODE.register("The player takes no damage", false);
}

rusteal_runtime::on_load!(startup);
```

```rust
if GOD_MODE.get().unwrap_or(false) {
    return;
}
```

A `ConsoleVariable<T>` holds a `bool`, `i32`, `f32` or `String`. It also reads and writes the engine's own variables by name:

```rust
let _ = ConsoleVariable::<i32>::new("t.MaxFPS").set(60);
```

## Use case: a command on the player controller

**The goal:** `GiveGold 100` as a command of the player controller, as a C++ `UFUNCTION(Exec)` is.

```rust
#[uclass_impl]
impl MyPlayerController {
    #[ufunction(Exec)]
    fn give_gold(&mut self, amount: i32) {
    }
}
```

A `#[ufunction(Exec)]` is a console command on the classes the console reaches: the player controller, its pawn, its HUD, the cheat manager, the game mode and the game state. The command is the function's Unreal name, and its parameters come from the words after it.

## In the editor

Press **Play**, open the console (`` ` `` in the viewport, or the **Cmd** box of the **Output Log**), and type the command. Registered commands and variables show in the console's autocomplete with their help text.
