# Timers

**The goal:** run something later, once or repeatedly: a respawn in three seconds, a weapon's refire, a stamina tick.

```rust
use std::time::Duration;

use rusteal_runtime::runtime::method;
use rusteal_runtime::runtime::timer::{self, TimerHandle};

#[uclass(parent = Character)]
pub struct Enemy {
    removal_timer: Option<TimerHandle>,
}

#[uclass_impl]
impl Enemy {
    #[ufunction(Override)]
    fn receive_end_play(&mut self, _end_play_reason: u8) {
        if let Some(timer) = self.removal_timer() {
            timer.clear();
        }
    }
}

impl Enemy {
    fn die(&mut self) -> RustealResult<()> {
        let timer = timer::set_timer(
            &self.as_ref(),
            Duration::from_secs(5),
            false,
            method(self, Self::remove_from_level),
        )?;

        self.set_removal_timer(Some(timer));
        Ok(())
    }

    fn remove_from_level(&mut self) {
        if let Ok(me) = self.as_ref().checked() {
            me.k2_destroy_actor();
        }
    }
}
```

`timer::set_timer` (since 0.10) is Blueprint's **Set Timer by Event**: it calls a closure from the timer manager of the object's world after a `Duration`, once or, when looping, every time it elapses. `method(self, Self::remove_from_level)` makes that closure a method of the class, checked by the compiler; the method needs no `#[ufunction]` (see [Delegates](delegates.md#binding-a-method)).

| Argument | What it is |
|---|---|
| object | the object whose world runs the timer (`&self.as_ref()`) |
| delay | the time until the call, and between calls when looping |
| looping | `true` to call it again after every `delay` |
| callback | the closure or `method(...)` to call |

`timer::set_timer_with_first_delay` takes the first call's delay apart from the others, and `timer::set_timer_for_next_tick` calls a closure once, on the next frame.

It returns a `TimerHandle`:

| Method | What it does |
|---|---|
| `clear()` | Stops the timer. |
| `pause()`, `unpause()` | Freezes and resumes it. |
| `remaining()` | The time left, `None` once it has stopped. |
| `is_active()` | Whether it is still running. |

Dropping the handle does not stop the timer, as in Blueprint: keep it in a field of the class (a field without `#[uproperty]`, as `removal_timer` above) to stop it later. The closure is freed when the timer is cleared, when a one-shot timer has fired, or when its world goes away.

Clear the timers of an object in its `receive_end_play`, so none fires on an object being destroyed. Setting a timer again does not stop the one already running: clear the stored handle first when the new timer replaces it.

## Use case: a looping timer at a fixed rate

**The goal:** update stamina thirty times a second, whatever the frame rate.

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    let tick = timer::set_timer(
        &self.as_ref(),
        Duration::from_secs_f32(1.0 / 30.0),
        true,
        method(self, Self::sprint_fixed_tick),
    );

    self.set_sprint_timer(tick.ok());
}
```

## Use case: refire with a cooldown

**The goal:** an automatic weapon fires every `refire_rate` seconds while the trigger is held; a semi-automatic one fires once and waits.

```rust
fn fire(&mut self) {
    if !self.is_firing() {
        return;
    }

    let next = if self.b_full_auto() {
        method(self, Self::fire)
    } else {
        method(self, Self::fire_cooldown_expired)
    };

    if let Some(previous) = self.refire_handle() {
        previous.clear();
    }

    let refire = timer::set_timer(
        &self.as_ref(),
        Duration::from_secs_f32(self.refire_rate()),
        false,
        next,
    );

    self.set_refire_handle(refire.ok());
}
```

## Waiting in async code

In a task started with `task::spawn_local`, `timer::sleep(&me, Duration::from_secs(2)).await` waits without a callback. See [Async code](async.md).

## Timers and the game's time

Timers follow the game: they pause when the game is paused, and slow down with time dilation; so does `timer::sleep`. `timer::sleep_real` waits on the engine's clock instead, which goes on during a pause. To know the current game time:

```rust
let now = GameplayStatics::get_time_seconds(self.as_ref().upcast_to());
```

Comparing it with a stored time is the other way to handle cooldowns, without a timer:

```rust
if now - self.last_attack_time() > f64::from(self.cooldown()) {
    self.set_last_attack_time(now);
}
```

## Timers by function name

`KismetSystemLibrary::k2_set_timer` is **Set Timer by Function Name**: it calls a function of an object by its Unreal name, and `k2_clear_timer` stops it by the same name. A name is checked when the game runs, not when it compiles: keep it for a function a Blueprint defines.

`KismetSystemLibrary::k2_set_timer_delegate` takes a closure, as `timer::set_timer` does, and gives the engine's `FTimerHandle`; since 0.10 its closure is freed with the timer.

## Every frame without an actor

`receive_tick` runs every frame on an actor. Code that is not an actor, or that must also run in the editor outside Play, can register a frame callback. See [Work off the game thread](../systems/tasks.md), `task::on_tick`.
