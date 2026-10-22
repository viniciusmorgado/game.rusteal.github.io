# Automation tests

**The goal:** tests that need the engine (a spawned actor, a loaded level, the game's rules) written in Rust and run by Unreal's automation system, next to C++ ones: from the **Session Frontend**, from the editor's command line, or on a build machine.

```rust
use rusteal_runtime::runtime::automation::{self, Contexts};

fn register_tests() {
    let registered = automation::register_test("Game.Combat.DamageIsClamped", Contexts::ALL, || {
        let health = Health::new(100.0);
        let left = health.after_damage(250.0);

        if left == 0.0 {
            Ok(())
        } else {
            Err(format!("health went to {left}"))
        }
    });

    if let Err(e) = registered {
        log::error!("{e}");
    }
}

rusteal_runtime::on_load!(register_tests);
```

`automation::register_test(name, contexts, closure)` (since 0.10) registers a test whose closure runs on the game thread: `Err` fails it with its message, and `automation::add_error`, `add_warning` and `add_info` add lines to its report while it runs. The dots of the name make the tree of the **Automation** tab; registering a name again replaces the test, and a library's tests go away when it unloads. `Contexts` says where it may run: `EDITOR`, `CLIENT`, `SERVER`, `COMMANDLET`, or `ALL`.

Run them from a command line, with a report the build machine can read:

```bash
UnrealEditor MyGame.uproject -nullrhi -unattended -ExecCmds="Automation RunTests Game" -testexit="Automation Test Queue Empty" -ReportExportPath=Saved/Automation
```

`Saved/Automation/index.json` has each test's state and messages. Tests that need no engine at all are faster as Rust unit tests (see [Testing without the engine](testing.md)).
