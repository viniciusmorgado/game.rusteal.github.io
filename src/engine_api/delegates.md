# Delegates

Delegates are how the engine says "this happened" to whoever is listening: an actor overlapped, a component was hit, a montage ended, a pawn was destroyed. Rust code listens with a closure or with a function of its class, and Rust classes declare their own delegates for Blueprints and other Rust code to listen to.

## Use case: reacting to an overlap with a function

**The goal:** a jump pad launches any character that steps on it.

```rust
use bindings::engine::{
    BoxComponent, Character, CharacterExt, ECollisionChannel, ECollisionEnabled,
    ECollisionResponse, PrimitiveComponentExt, SceneComponent,
};

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
        let me = self.as_ref();

        if let Err(e) = me
            .checked()
            .and_then(|actor| actor.on_actor_begin_overlap().add_ufunction(&me, "BeginOverlap"))
        {
            ulog!(LOG_WARNING, "[MyGame] jump pad overlap: {e}");
        }
    }

    #[ufunction]
    fn begin_overlap(&mut self, _overlapped_actor: UObjectRef<Actor>, other_actor: UObjectRef<Actor>) {
        let Ok(character) = other_actor.cast::<Character>().and_then(|c| c.checked()) else {
            return;
        };

        let launch = DVec3::Z * f64::from(self.z_strength());
        character.launch_character(&FVector::from_dvec3(launch), false, true);
    }
}
```

`add_ufunction(&target, "Function")` binds a `#[ufunction]` of `target` by its Unreal name, as C++'s `AddDynamic` does. The function takes the delegate's parameters, in order. Binding the same function again does nothing, so it is safe in `receive_begin_play`.

For an actor's own overlaps, overriding `receive_actor_begin_overlap` is simpler still (see [Engine events](../classes/events.md)); delegates are for a component's overlaps and for another object's events.

## Use case: a component's overlap

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    let me = self.as_ref();

    let bound = self.trigger().and_then(|trigger| {
        trigger.checked()?.on_component_begin_overlap().add_ufunction(&me, "OnTriggerOverlap")
    });

    if let Err(e) = bound {
        ulog!(LOG_WARNING, "[MyGame] trigger overlap: {e}");
    }
}

#[ufunction]
#[allow(clippy::too_many_arguments)]
fn on_trigger_overlap(
    &mut self,
    _overlapped_component: UObjectRef<PrimitiveComponent>,
    other_actor: UObjectRef<Actor>,
    _other_comp: UObjectRef<PrimitiveComponent>,
    _other_body_index: i32,
    _from_sweep: bool,
    _sweep_result: UStructRef<FHitResult>,
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
        Err(e) => ulog!(LOG_WARNING, "[MyGame] cannot watch the pawn: {e}"),
    }
}
```

`add(closure)` binds a Rust closure, which receives the delegate's parameters. It returns a binding:

- dropping the binding unbinds the closure;
- `detach()` keeps the closure bound for as long as the object that raises the delegate lives.

The closure captures `UObjectRef`s, which are `Copy`, and turns them back into Rust classes with `from_obj` when it runs: by then the object may be gone, and `from_obj` says so.

Closures and `add_ufunction` both work on any delegate the engine exposes to Blueprints: animation (`on_montage_ended`), AI perception (`on_target_perception_updated`), movement (`on_projectile_stop`), AI moves (`receive_move_completed`), EQS (`on_query_finished_event`). See the [Gameplay recipes](../recipes/index.md).

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
health.on_health_changed_add(|current, max| {
})?.detach();

health.on_health_changed_add_ufunction(&hud.as_ref(), "RefreshHealth")?;
```

Its parameters are scalars, objects, classes and structs (as `&OwnedStruct<T>`; a closure receives an `OwnedStruct<T>`). `#[udelegate(name = "...")]` gives the Unreal name when PascalCase is not it.

### In the editor

In a Blueprint that holds the component, select the component in the **Components** panel; in the **Details** panel, the **Events** section lists **On Health Changed** with a **+** button that adds the bound event to the graph. Or, from any reference to the component, drag a wire and search for **Bind Event to On Health Changed**.

> **Screenshot:** the Events section of the Details panel listing On Health Changed for a Rust component. *(images/engine_api/udelegate_events.png)*

## Delegates that cannot be bound

Only delegates exposed to Blueprints can be bound. Some engine delegates exist only in C++ (`UPathFollowingComponent::OnRequestFinished`, for one); for those, look for a Blueprint-exposed delegate raised for the same thing (the AI controller's `ReceiveMoveCompleted`).
