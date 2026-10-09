# Console commands and variables

The console is a game's quickest debugging tool: commands to give items, skip waves, spawn enemies; variables to turn on god mode or tune a value while playing. Rust adds both. (For Rusteal's own commands, `Rusteal.Reload` and `Rusteal.AutoReload`, see [Console commands](../iteration/console.md).)

## Use case: a cheat command

**The goal:** `mygame.give_gold 500` in the console gives the player gold.

```rust
use rusteal_runtime::runtime::{LOG_DISPLAY, LOG_ERROR, UObjectHandle, UObjectRef, console, ulog};

fn startup() {
    let result = console::register_command("mygame.give_gold", "mygame.give_gold <amount>", |args, world| {
        let amount: i32 = args.first().and_then(|a| a.parse().ok()).unwrap_or(100);
        ulog!(LOG_DISPLAY, "[MyGame] giving {amount} gold");
        give_gold(world, amount);
    });

    if let Err(e) = result {
        ulog!(LOG_ERROR, "[MyGame] no console command: {e}");
    }
}

rusteal_runtime::on_load!(startup);

fn give_gold(world: UObjectHandle, amount: i32) {
}
```

- `console::register_command(name, help, closure)` adds the command. The closure receives the words after the command (`["500"]`) and the world the command runs in.
- `on_load!(function)` runs a function every time the library loads, hot reloads included. A library's commands and variables are removed when it unloads and registered again when it loads, so they always run the current code.

The world arrives as a raw handle; turn it into a reference to use it:

```rust
use bindings::engine::{GameplayStatics, World};

let world = unsafe { UObjectRef::<World>::from_raw(world) }.upcast_to::<Object>();
let pawn = GameplayStatics::get_player_pawn(world, 0);
```

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
