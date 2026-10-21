# Delegates

Delegates are how the engine says "this happened" to whoever is listening: an actor overlapped, a component was hit, a montage ended, a pawn was destroyed. Rust code listens with a method of its class or a closure, both checked by the compiler, or awaits the next time a delegate fires; Rust classes declare their own delegates for Blueprints and other Rust code to listen to.

## Use case: reacting to an overlap with a method

**The goal:** a jump pad launches any character that steps on it.

```rust
use bindings::engine::{
    BoxComponent, Character, CharacterExt, ECollisionChannel, ECollisionEnabled,
    ECollisionResponse, PrimitiveComponentExt, SceneComponent
};
use rusteal_runtime::runtime::method;

#[uclass(parent = Actor)]
pub struct JumpPad {
    #[component(root)]
    root: SceneComponent,

    #[component(attach = "root")]
    r#box: BoxComponent,

    #[uproperty(EditAnywhere, default = 1000.0)]
    z_strength: f32,
}

#[uclass_impl]
impl JumpPad {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let r#box = self.r#box()?.checked()?;
        r#box.set_collision_enabled(ECollisionEnabled::QueryOnly);
        r#box.set_collision_response_to_all_channels(ECollisionResponse::ECR_Ignore);
        r#box.set_collision_response_to_channel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        let bound = self
            .as_ref()
            .checked()
            .and_then(|actor| actor.on_actor_begin_overlap().add(method(self, Self::begin_overlap)));

        match bound {
            Ok(binding) => binding.detach(),
            Err(e) => log::warn!("[MyGame] jump pad overlap: {e}"),
        }
    }
}

impl JumpPad {
    fn begin_overlap(&mut self, _overlapped_actor: UObjectRef<Actor>, other_actor: UObjectRef<Actor>) {
        let Ok(character) = other_actor.cast::<Character>().and_then(|c| c.checked()) else {
            return;
        };

        let launch = DVec3::Z * f64::from(self.z_strength());
        character.launch_character(&FVector::from_dvec3(launch), false, true);
    }
}
```

### Binding a method

`method(self, Self::begin_overlap)` (since 0.10, `rusteal_runtime::runtime::method`) turns a method of the class into a closure: the compiler checks that it exists and that its parameters are the delegate's, in order. It takes `&self` or `&mut self` and up to nine parameters, and needs no `#[ufunction]`. The object is found again on each call; a call after it is gone does nothing.

`add` returns a binding: dropping it unbinds, `detach()` keeps the method bound as long as the object raising the delegate lives. Binding twice calls the method twice: bind in `receive_begin_play`, once.

Structs reach a method as `OwnedStruct<T>`, a copy. A `#[ufunction]` handler that takes a `UStructRef<T>` (because a Blueprint calls it too) is bound with a closure that passes `.as_ref()`:

```rust
let mut on_hit = method(self, Self::on_floor_hit);

floor.on_component_hit().add(move |component, other, other_comp, impulse, hit| {
    on_hit(component, other, other_comp, impulse, hit.as_ref())
})?.detach();
```

For an actor's own overlaps, overriding `receive_actor_begin_overlap` is simpler still (see [Engine events](../classes/events.md)); delegates are for a component's overlaps and for another object's events.

## Use case: a component's overlap

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    let bound = self.trigger().and_then(|trigger| {
        trigger
            .checked()?
            .on_component_begin_overlap()
            .add(method(self, Self::on_trigger_overlap))
    });

    match bound {
        Ok(binding) => binding.detach(),
        Err(e) => log::warn!("[MyGame] trigger overlap: {e}"),
    }
}

fn on_trigger_overlap(
    &mut self,
    _overlapped_component: UObjectRef<PrimitiveComponent>,
    other_actor: UObjectRef<Actor>,
    _other_comp: UObjectRef<PrimitiveComponent>,
    _other_body_index: i32,
    _from_sweep: bool,
    _sweep_result: OwnedStruct<FHitResult>,
) {
}
```

The common component delegates and their parameters:

| Delegate | Parameters |
|---|---|
| `on_component_begin_overlap` | overlapped component, other actor, other component, other body index, from sweep, sweep result |
| `on_component_end_overlap` | overlapped component, other actor, other component, other body index |
| `on_component_hit` | hit component, other actor, other component, normal impulse, hit |
| `on_actor_begin_overlap`, `on_actor_end_overlap` | overlapped actor, other actor |
| `on_destroyed` | destroyed actor |

## Use case: listening with a closure

**The goal:** the player controller respawns the player when its pawn is destroyed.

```rust
#[ufunction(Override)]
fn receive_possess(&mut self, possessed_pawn: UObjectRef<Pawn>) {
    let controller: UObjectRef<PlayerController> = self.as_ref();

    let bound = possessed_pawn.checked().and_then(|pawn| {
        pawn.on_destroyed().add(move |_destroyed_actor| {
            if let Ok(mut me) = MyPlayerController::from_obj(controller) {
                me.respawn();
            }
        })
    });

    match bound {
        Ok(binding) => binding.detach(),
        Err(e) => log::warn!("[MyGame] cannot watch the pawn: {e}"),
    }
}
```

`add(closure)` binds a closure, which receives the delegate's parameters, as a method does. It returns a binding:

- dropping the binding unbinds the closure;
- `detach()` keeps the closure bound for as long as the object that raises the delegate lives.

The closure captures `UObjectRef`s, which are `Copy`, and turns them back into Rust classes with `from_obj` when it runs: by then the object may be gone, and `from_obj` says so.

Methods and closures work on any delegate the engine exposes to Blueprints: animation (`on_montage_ended`), AI perception (`on_target_perception_updated`), movement (`on_projectile_stop`), AI moves (`receive_move_completed`), EQS (`on_query_finished_event`). See the [Gameplay recipes](../recipes/index.md).

## Use case: a delegate of your own

**The goal:** a health component announces every change, and a UMG widget, a Blueprint or other Rust code updates itself.

```rust
#[uclass(parent = ActorComponent)]
pub struct Health {
    #[uproperty(EditAnywhere, default = 100.0)]
    max_health: f32,

    #[uproperty(VisibleAnywhere)]
    current: f32,
}

#[uclass_impl]
impl Health {
    #[udelegate]
    pub fn on_health_changed(&self, current: f32, max: f32) {}

    #[ufunction]
    pub fn take_damage(&mut self, amount: f32) {
        self.set_current((self.current() - amount).max(0.0));
        self.on_health_changed(self.current(), self.max_health());
    }
}
```

`#[udelegate]` declares a multicast delegate, C++'s `DECLARE_DYNAMIC_MULTICAST_DELEGATE` with a `BlueprintAssignable` property. The method's body stays empty; calling it broadcasts to every listener. The macro adds two methods for Rust listeners:

```rust
health.on_health_changed_add(method(self, Self::refresh_health))?.detach();

health.on_health_changed_add(|current, max| {
})?.detach();
```

Its parameters are the types a [`#[ufunction]`](../classes/functions.md#parameters-and-return-values) takes (since 0.10): scalars, strings, `FName`, `Text`, objects, classes, soft and weak references, interfaces, `Vec<T>`, `HashMap<K, V>`, `HashSet<T>`, and structs as `OwnedStruct<T>` or `&OwnedStruct<T>` (a closure receives an `OwnedStruct<T>`). `#[udelegate(name = "...")]` gives the Unreal name when PascalCase is not it.

### Single-cast and sparse delegates

| Declaration | C++ | What it is |
|---|---|---|
| `#[udelegate]` | `DECLARE_DYNAMIC_MULTICAST_DELEGATE` | Any number of listeners, Blueprint or Rust. |
| `#[udelegate(sparse)]` | `DECLARE_DYNAMIC_MULTICAST_SPARSE_DELEGATE` | The same for its listeners, but it takes one byte in the object until something binds to it, where a multicast delegate takes sixteen. For a delegate on a class with many instances that is rarely bound, such as an event on every projectile or every tree. |
| `#[udelegate(single)]` | `DECLARE_DYNAMIC_DELEGATE` | One binding at most: binding again replaces it. Calling the method calls the binding, and does nothing when there is none. Blueprints cannot bind to it. |

A single-cast delegate's methods are named `_bind` instead of `_add`:

```rust
#[udelegate(single)]
pub fn on_answer(&self, choice: i32, text: String) {}
```

```rust
let binding = dialog.on_answer_bind(|choice, text| {
})?;
```

Dropping the binding unbinds it, unless a later binding has replaced it; `on_answer_bind_ufunction(&target, "Function")` binds a UFunction instead.

### In the editor

In a Blueprint that holds the component, select the component in the **Components** panel; in the **Details** panel, the **Events** section lists **On Health Changed** with a **+** button that adds the bound event to the graph. Or, from any reference to the component, drag a wire and search for **Bind Event to On Health Changed**.

> **Screenshot:** the Events section of the Details panel listing On Health Changed for a Rust component. *(images/engine_api/udelegate_events.png)*

## Use case: waiting for a delegate

**The goal:** in async code, go on when something happens, without a handler.

```rust
let destroyed = boss.on_destroyed().next();
let opened = gate.on_opened_next();

let _killer = destroyed.await?;
opened.await?;
```

`next()` (since 0.10) binds at once and unbinds when the delegate has fired, and gives its parameters: one value, or a tuple when there are several. A `#[udelegate]` has `<name>_next()`. When the object goes away first, the future gives an error instead of waiting forever. Take `next()` before doing what makes the delegate fire, as the binding starts then. See [Async code](async.md).

## Use case: an engine event that is not a delegate

Some engine events exist only in C++, as native delegates: the application going to the background, a world being initialized, a level streamed in, a player logging in. Rusteal binds a few of them (since 0.10), with `add(closure)` and `next()` as for the others:

| Events | What they are |
|---|---|
| `core_events::pre_exit`, `will_enter_background`, `has_entered_foreground`, `will_deactivate`, `has_reactivated` | the application's lifecycle (`FCoreDelegates`) |
| `world_events::initialized`, `cleanup`, `level_added`, `level_removed` | worlds and their levels (`FWorldDelegates`) |
| `game_mode_events::post_login`, `logout` | players joining and leaving (`FGameModeEvents`) |
| `component.on_request_finished()` | a path following component's move ending, with its request id and result (`ai` feature) |

```rust
use bindings::prelude::*;

core_events::will_enter_background()
    .add(move |()| save_progress())?
    .detach();
```

Their bindings unbind when dropped, as the others do; `detach()` keeps them until the library unloads.

## Use case: an engine function that takes a delegate

Some engine functions take a delegate to call later: a timeline's track, a timer, a widget's event. In Rust they take a closure, whose arguments are the delegate's (since 0.10):

```rust
timeline.add_interp_float(curve, move |alpha| {
    // called on every update of the timeline, with the curve's value
}, None, None);
```

The closure lives as long as the engine keeps the delegate, and is freed with it: when the object whose function took it is destroyed. Timers set with `KismetSystemLibrary::k2_set_timer_delegate` free theirs with the timer (see [Timers](timers.md)). Delegates that return a value, or write to an argument, are not bound.

## Binding a function by name

`add_ufunction(&target, "Function")` binds the UFunction of `target` with that Unreal name, as C++'s `AddDynamic` does; binding the same function again does nothing. A name is checked when the game runs, not when it compiles: it is for a function a Blueprint defines. A `#[udelegate]` has `<name>_add_ufunction(&target, "Function")`.

## Delegates that cannot be bound

Only delegates exposed to Blueprints can be bound, and the native ones in the table above. For another engine delegate that exists only in C++, look for a Blueprint-exposed delegate raised for the same thing.
