# Errors

The engine fails in ways C++ code tends to crash on: an object destroyed while something still points to it, a cast to the wrong class, a function that is not there. In Rusteal those failures are values: `RustealResult<T>`, an alias of `Result<T, RustealError>`.

| Error | When |
|---|---|
| `ObjectDestroyed` | `checked()` on an object that is gone or was never set. |
| `InvalidCast` | `cast` or `from_obj` to a class the object is not. |
| `NullArgument` | A null class or object where one is required (an unset `SubclassOf` given to `spawn_actor_of_class`). |
| `FunctionNotFound(name)` | A function bound or called by a name the class does not have. |
| `PropertyNotFound(name)` | A property reached by a name the class does not have. |
| `TypeMismatch` | A value of the wrong type for a property or parameter. |
| `IndexOutOfRange` | An array index past the end. |
| `InvalidOperation(message)` | An engine call that refused: a spawn that failed, an asset that could not be loaded. |
| `Internal(message)` | Something Rusteal itself did not expect. |

`RustealError` implements `Display` and `std::error::Error`: log it with `{e}`, or convert it into the error type of your own code.

## The pattern: Result inside, log at the edge

Events and functions the engine calls cannot return an error to anyone. Put the work in a method that returns `RustealResult`, use `?` in it, and log the error in the event:

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    if let Err(e) = self.setup() {
        ulog!(LOG_WARNING, "[MyGame] {} setup failed: {e}", self.as_ref().get_name().unwrap_or_default());
    }
}

impl Turret {
    fn setup(&mut self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        let mesh = self.mesh()?.checked()?;
        let world = self.as_ref().get_world()?;
        Ok(())
    }
}
```

Where a failure is a normal case (no target, the player is not a character), `let else` or `if let` say it without an error:

```rust
let Ok(character) = other_actor.cast::<Character>() else {
    return;
};
```

## Panics

A panic in your code does not crash the editor: it is caught where the engine called into Rust, logged as an error in the Output Log, and the call returns. The object may be left half updated, so a panic is a bug to fix, not a way to handle errors. `unwrap()` on a `RustealResult` is a panic waiting for an object to be destroyed at the wrong moment: prefer `?`, `let else` or `unwrap_or_default()`.
