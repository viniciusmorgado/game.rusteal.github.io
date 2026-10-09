# Logging

**The goal:** print messages from Rust code to the Output Log.

```rust
use rusteal_runtime::runtime::{LOG_DISPLAY, LOG_ERROR, LOG_WARNING, ulog};

ulog!(LOG_DISPLAY, "[MyGame] wave {wave} started");
ulog!(LOG_WARNING, "[MyGame] no spawn points in the level");
ulog!(LOG_ERROR, "[MyGame] could not load the save: {e}");
```

`ulog!` takes a level and the arguments of `format!`. Its messages go to the Output Log under the `LogRusteal` category, with the level's color: plain for `LOG_DISPLAY`, yellow for `LOG_WARNING`, red for `LOG_ERROR`.

```text
LogRusteal: [MyGame] wave 3 started
LogRusteal: Warning: [MyGame] no spawn points in the level
```

## Filtering

In the **Output Log** tab, **Filters > Categories** can show `LogRusteal` alone. A prefix of your own at the start of each message (`[MyGame]`, `[Inventory]`) makes them easy to find with the search box as well.

> **Screenshot:** the Output Log filtered to the LogRusteal category. *(images/iteration/output_log_filter.png)*

## Errors that reach the log by themselves

Some failures are logged without any code of yours:

- A panic in your code is caught where the engine called into Rust, and logged as an error. The editor keeps running; the function that panicked stops there.
- A function bound by name (an input action, a delegate, a timer) that does not exist is logged with its name.
- A library built against another Rusteal version is refused, with both versions in the log.

## Logging a `RustealResult`

Engine calls that can fail return a `RustealResult`. A common pattern is to put the work in a method that returns one, and log the error in the event that calls it:

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    if let Err(e) = self.setup() {
        ulog!(LOG_WARNING, "[MyGame] setup failed: {e}");
    }
}
```

See [Errors](../engine_api/errors.md).
