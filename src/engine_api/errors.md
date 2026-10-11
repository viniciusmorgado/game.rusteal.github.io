# Errors

The engine fails in ways C++ code tends to crash on: an object destroyed while something still points to it, a cast to the wrong class, a function that is not there. In Rusteal those failures are values: `RustealResult<T>`, an alias of `Result<T, RustealError>`.

## What an error says

A `RustealError` says what went wrong, on which Unreal object, why Unreal refused when it says, and where in your Rust code:

```text
invalid cast: expected Pawn (object StaticMeshActor_0 of class StaticMeshActor)
    at my-game/src/turret.rs:42:23
while aiming at the target
    at my-game/src/turret.rs:30:14
```

- **What**: its kind, and the name involved (the property, the function, the class a cast expected).
- **On what**: the object's name and its class, when the error came from an object.
- **Why, says Unreal**: when the engine side knows more than the kind, its reason, after `Unreal:`: `type mismatch: Unreal: Ammo (IntProperty of Turret) is not a BoolProperty`, `index out of range: Unreal: index 5 past the 3 elements of Targets (ArrayProperty of Turret)`.
- **Where**: the file and line of the call that failed in your code, not inside Rusteal: the functions that return a `RustealResult` pass their caller's location along.
- **While doing what**: the contexts your code added on the way up (below).
- **Caused by**: the error underneath, when it came from another library (a file that did not open, a number that did not parse).

It prints three ways:

| Format | Gives | For |
|---|---|---|
| `{e}` | One line: the contexts, then the error, then the object. | Logs, UI, `to_string()`. |
| `{e:#}` | The same line, then the causes and the location. | A log line with everything. |
| `{e:?}` | The report above, one item per line. | Debugging; it is also what `unwrap()` prints when it panics. |

`e.log(LOG_WARNING)` reports it (see [In the editor](#in-the-editor)).

## In the editor

A reported error goes to two places:

- the **Output Log**, as the report above, followed by the Blueprint call stack when a Blueprint was running;
- the **Rusteal** tab of the **Message Log** (*Window > Message Log*, then **Rusteal** in its list), one line per error. Its object is a link that selects it; an object of Play In Editor, once Play has stopped, selects the same actor in the level. Its Rust file and line are a link that opens the file at that line in the IDE set in *Editor Preferences > General > Source Code*. An error also shows a notification, at most once every few seconds.

> **Screenshot:** the Message Log's Rusteal tab with an invalid cast error, its object link and its errors.rs:42 link. *(images/engine_api/message_log.png)*

Panics are reported the same way, with the file and line of the `panic!`, the `unwrap()` or the index out of bounds.

## Use case: a function that can fail

**The goal:** a Blueprint calls a Rust function that can fail, and the failure is a Blueprint runtime error on the node that called it, not a silent default.

A `#[ufunction]` (or an `Override`) returns a `RustealResult<T>` when it can fail. Unreal sees a function returning `T` (nothing for `RustealResult<()>`):

```rust
#[ufunction]
fn reload(&mut self) -> RustealResult<i32> {
    let magazine = self.magazine().checked()?;
    Ok(magazine.rounds())
}

#[ufunction(Override)]
fn receive_begin_play(&mut self) -> RustealResult<()> {
    self.setup()
}
```

An `Err` is reported with the function's name (`Turret::Reload failed: ...`) and the function returns the default value (`0`, `false`, a null reference). Called from a Blueprint, it is also a Blueprint runtime error: in Play In Editor the **Message Log** opens on it with a link to the node, as for an *Accessed None*, and with *Blueprint Exceptions* breakpoints on, the debugger stops there.

> **Screenshot:** a Blueprint runtime error raised by a Rust function, with the link to the calling node in the Message Log. *(images/engine_api/blueprint_error.png)*

The same panic, unwrap or `?` inside a function that does not return a `RustealResult` is still caught: a panic is reported as above, and the function returns the default value.

## Use case: Result inside, log at the edge

**The goal:** a turret's setup can fail in several ways, and the log says which, where and on what.

Events and functions the engine calls cannot return an error to anyone. Put the work in a method that returns `RustealResult`, use `?` in it, and log the error in the event:

```rust
use rusteal_runtime::runtime::{ErrorContext, LOG_WARNING, RustealResult};

#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    if let Err(e) = self.setup() {
        e.log(LOG_WARNING);
    }
}

impl Turret {
    fn setup(&mut self) -> RustealResult<()> {
        let mesh = self.mesh()?.checked()?;
        let target = self.target().cast::<Pawn>().context("aiming at the target")?;
        let range: f32 = self.range_text().parse().context("reading the range")?;
        Ok(())
    }
}
```

`.context("...")` adds what the code was doing to the error, with its own line; contexts read outermost first in `{e}`. It works on any `Result` whose error is a Rust error (a `RustealError`, an `std::io::Error`, a `serde_json::Error`...), and on `Option`, where `None` becomes an error with that message:

```rust
let spawn = self.find_spawn_point().context("no spawn point in the level")?;
```

## Errors from other libraries

`?` turns the standard library's errors (`std::io::Error`, `ParseIntError`, `ParseFloatError`, `Utf8Error`...) into a `RustealError` by itself, keeping them as the cause and recording the line of the `?`. Another crate's error goes through `.context("...")`, or `.map_err(RustealError::other)` with no context.

`RustealError` implements `std::error::Error`, so it goes the other way too: into `anyhow::Error`, a `Box<dyn Error>`, or a `thiserror` enum of your own.

## Telling errors apart

`kind()` gives an `ErrorKind` to match on:

```rust
match other_actor.cast::<Character>() {
    Ok(character) => self.hit(character),
    Err(e) if e.is(ErrorKind::InvalidCast) => {}
    Err(e) => return Err(e),
}
```

| `ErrorKind` | When |
|---|---|
| `ObjectDestroyed` | `checked()` on an object that is gone or was never set. |
| `InvalidCast` | `cast` or `from_obj` to a class the object is not. |
| `NullArgument` | A null class or object where one is required (an unset `SubclassOf` given to `spawn_actor_of_class`). |
| `FunctionNotFound` | A function bound or called by a name the class does not have. |
| `PropertyNotFound` | A property or a parameter reached by a name the class or the function does not have. |
| `TypeMismatch` | A value of the wrong type for a property or parameter. |
| `IndexOutOfRange` | An array index past the end. |
| `InvalidOperation` | An engine call that refused: a spawn that failed, an asset that could not be loaded. |
| `Internal` | Something Rusteal itself did not expect. |
| `Other` | An error of your own, or from another library. |

Where a failure is a normal case (no target, the player is not a character), `let else` or `if let` say it without an error:

```rust
let Ok(character) = other_actor.cast::<Character>() else {
    return;
};
```

## Your own errors

```rust
use rusteal_runtime::runtime::{ErrorKind, RustealResult, bail, ensure};

fn spend(&mut self, cost: i32) -> RustealResult<()> {
    ensure!(cost >= 0, "a cost of {cost}");

    if cost > self.gold() {
        bail!(ErrorKind::InvalidOperation, "{cost} gold, {} in the purse", self.gold());
    }

    self.set_gold(self.gold() - cost);
    Ok(())
}
```

`bail!` returns an error with a message, of the kind given first or `Other` without one; `ensure!` does when its condition is false. `RustealError::msg("...")`, `RustealError::with_message(kind, "...")` and the constructors named after each kind (`RustealError::null_argument()`, `RustealError::invalid_operation("...")`) make one to return yourself, and `.on(&actor)` names the object it concerns.

## Panics

A panic in your code does not crash the editor: it is caught where the engine called into Rust, reported as an error with its file and line (see [In the editor](#in-the-editor)), and the call returns. The object may be left half updated, so a panic is a bug to fix, not a way to handle errors. `unwrap()` on a `RustealResult` is a panic waiting for an object to be destroyed at the wrong moment: prefer `?`, `let else` or `unwrap_or_default()`.
