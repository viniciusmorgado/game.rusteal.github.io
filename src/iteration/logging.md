# Logging

**The goal:** print messages from Rust code to the Output Log.

```rust
log::info!("[MyGame] wave {wave} started");
log::warn!("[MyGame] no spawn points in the level");
log::error!("[MyGame] could not load the save: {e}");
```

Rusteal logs through the [`log`](https://docs.rs/log) crate, the logging facade of the Rust ecosystem: the projects Rusteal creates depend on it (`log = { workspace = true }` in the game crate), and the library routes it to the Output Log when it loads. Its messages go under the `LogRusteal` category, with the level's color: plain for `info!`, yellow for `warn!`, red for `error!`.

```text
LogRusteal: [MyGame] wave 3 started
LogRusteal: Warning: [MyGame] no spawn points in the level
```

`debug!` and `trace!` are off by default. Turn them on, for the whole library, where it starts:

```rust
fn verbose_logs() {
    log::set_max_level(log::LevelFilter::Debug);
}

rusteal_runtime::on_load!(verbose_logs);
```

They show at the display level, like `info!`. The crates your game depends on that log through `log` (most do) show in the Output Log the same way.

A project made before Rusteal routed `log` has no `log` dependency: add `log = "0.4"` to `[workspace.dependencies]` in `Rust/Cargo.toml` and `log = { workspace = true }` to the game crate's `[dependencies]`. Its `ulog!` calls keep working.

## Filtering

In the **Output Log** tab, **Filters > Categories** can show `LogRusteal` alone. A prefix of your own at the start of each message (`[MyGame]`, `[Inventory]`) makes them easy to find with the search box as well.

> **Screenshot:** the Output Log filtered to the LogRusteal category. *(images/iteration/output_log_filter.png)*

## Errors that reach the log by themselves

Some failures are logged without any code of yours:

- A panic in your code is caught where the engine called into Rust, and logged as an error. The editor keeps running; the function that panicked stops there.
- A function bound by name (`bind_action_by_name`, `add_ufunction`, `k2_set_timer`) that does not exist is logged with its name.
- An `async #[ufunction]` that returns an error logs it when it finishes.
- A library built against another Rusteal version is refused, with both versions in the log.

## Logging a `RustealResult`

Engine calls that can fail return a `RustealResult`. A common pattern is to put the work in a method that returns one, and log the error in the event that calls it:

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    if let Err(e) = self.setup() {
        log::warn!("[MyGame] setup failed: {e}");
    }
}
```

See [Errors](../engine_api/errors.md).
