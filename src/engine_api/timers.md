# Timers

**The goal:** run something later, once or repeatedly: a respawn in three seconds, a weapon's refire, a stamina tick.

```rust
use bindings::engine::KismetSystemLibrary;

#[uclass_impl]
impl Enemy {
    #[ufunction]
    fn remove_from_level(&mut self) {
        if let Ok(me) = self.as_ref().checked() {
            me.k2_destroy_actor();
        }
    }

    #[ufunction(Override)]
    fn receive_end_play(&mut self, _end_play_reason: u8) {
        KismetSystemLibrary::k2_clear_timer(self.as_ref().upcast_to(), "RemoveFromLevel");
    }
}

impl Enemy {
    fn die(&mut self) {
        KismetSystemLibrary::k2_set_timer(
            self.as_ref().upcast_to(),
            "RemoveFromLevel",
            5.0,
            false,
            None,
            None,
            None,
        );
    }
}
```

`k2_set_timer` is the Blueprint node **Set Timer by Function Name**: it calls a function of the object, by its Unreal name, after the given seconds. The function is any `#[ufunction]` of the class: `remove_from_level` is `RemoveFromLevel`.

| Argument | What it is |
|---|---|
| object | the object whose function is called, as `UObjectRef<Object>` (`self.as_ref().upcast_to()`) |
| function name | the Unreal name of the function |
| time | seconds until the call |
| looping | `true` to call it every `time` seconds |
| the rest | initial start delay and variance, `None` for the defaults |

`k2_clear_timer` stops a timer by the same object and name. Clear the timers of an object in its `receive_end_play`, so none fires on an object being destroyed. Setting a timer that is already running restarts it.

## Use case: a looping timer at a fixed rate

**The goal:** update stamina thirty times a second, whatever the frame rate.

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    KismetSystemLibrary::k2_set_timer(
        self.as_ref().upcast_to(),
        "SprintFixedTick",
        1.0 / 30.0,
        true,
        None,
        None,
        None,
    );
}

#[ufunction]
fn sprint_fixed_tick(&mut self) {
}
```

## Use case: refire with a cooldown

**The goal:** an automatic weapon fires every `refire_rate` seconds while the trigger is held; a semi-automatic one fires once and waits.

```rust
#[ufunction]
fn fire(&mut self) {
    if !self.is_firing() {
        return;
    }

    let next = if self.b_full_auto() { "Fire" } else { "FireCooldownExpired" };

    KismetSystemLibrary::k2_set_timer(
        self.as_ref().upcast_to(),
        next,
        self.refire_rate(),
        false,
        None,
        None,
        None,
    );
}
```

## Timers and the game's time

Timers follow the game: they pause when the game is paused, and slow down with time dilation. To know the current game time:

```rust
let now = GameplayStatics::get_time_seconds(self.as_ref().upcast_to());
```

Comparing it with a stored time is the other way to handle cooldowns, without a timer:

```rust
if now - self.last_attack_time() > f64::from(self.cooldown()) {
    self.set_last_attack_time(now);
}
```

## Every frame without an actor

`receive_tick` runs every frame on an actor. Code that is not an actor, or that must also run in the editor outside Play, can register a frame callback. See [Work off the game thread](../systems/tasks.md), `task::on_tick`.
