# Engine events

**The goal:** run Rust code when the engine says something happened: the game started, a frame passed, the actor was hit, a controller possessed the pawn.

```rust
#[uclass_impl]
impl Hero {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
    }

    #[ufunction(Override)]
    fn receive_tick(&mut self, delta_seconds: f32) {
    }

    #[ufunction(Override)]
    fn receive_end_play(&mut self, end_play_reason: u8) {
    }

    #[ufunction(Override)]
    fn on_landed(&mut self, hit: UStructRef<FHitResult>) {
    }
}
```

`#[ufunction(Override)]` implements an event of the parent class: one of the functions a Blueprint implements as an **Event** node (**Event BeginPlay**, **Event Tick**, **Event Hit**). The method is named after the event in snake_case and takes the event's parameters, in order.

## C++ virtuals are events

A C++ class overrides virtual functions (`BeginPlay`, `Tick`, `NotifyHit`). A Rust class, like a Blueprint, overrides the events the engine raises for them:

| C++ virtual | Event | Rust method |
|---|---|---|
| `BeginPlay` | `ReceiveBeginPlay` | `receive_begin_play(&mut self)` |
| `EndPlay` | `ReceiveEndPlay` | `receive_end_play(&mut self, end_play_reason: u8)` |
| `Tick` | `ReceiveTick` | `receive_tick(&mut self, delta_seconds: f32)` |
| `Destroyed` | `ReceiveDestroyed` | `receive_destroyed(&mut self)` |
| `TakeDamage` | `ReceiveAnyDamage` | `receive_any_damage(&mut self, damage: f32, damage_type: UObjectRef<DamageType>, instigated_by: UObjectRef<Controller>, damage_causer: UObjectRef<Actor>)` |
| `NotifyHit` | `ReceiveHit` | `receive_hit(&mut self, my_comp, other, other_comp, self_moved, hit_location, hit_normal, normal_impulse, hit)` |
| `NotifyActorBeginOverlap` | `ReceiveActorBeginOverlap` | `receive_actor_begin_overlap(&mut self, other_actor: UObjectRef<Actor>)` |
| `NotifyActorEndOverlap` | `ReceiveActorEndOverlap` | `receive_actor_end_overlap(&mut self, other_actor: UObjectRef<Actor>)` |
| `Landed` | `OnLanded` | `on_landed(&mut self, hit: UStructRef<FHitResult>)` |
| `OnMovementModeChanged` | `K2_OnMovementModeChanged` | see below |
| `NotifyControllerChanged` | `ReceiveControllerChanged` | `receive_controller_changed(&mut self, old: UObjectRef<Controller>, new: UObjectRef<Controller>)` |
| `SetupPlayerInputComponent` | `ReceiveRestarted` | `receive_restarted(&mut self)`: the input component exists once a local player possesses the pawn |
| `APlayerController::SetupInputComponent` | `ReceiveBeginPlay` | `receive_begin_play(&mut self)` |
| `OnPossess` | `ReceivePossess` | `receive_possess(&mut self, possessed_pawn: UObjectRef<Pawn>)` |
| `ChoosePlayerStart` | `ChoosePlayerStart` | `choose_player_start(&mut self, player: UObjectRef<Controller>) -> UObjectRef<Actor>` |
| `DrawHUD` | `ReceiveDrawHUD` | `receive_draw_hud(&mut self, size_x: i32, size_y: i32)` |
| `UpdateCamera` | `BlueprintUpdateCamera` | see [Cameras](../recipes/cameras.md) |
| `UAnimNotify::Notify` | `Received_Notify` | see [Animation](../recipes/animation.md) |

Overriding `receive_tick` turns the actor's tick on: nothing else is needed.

A virtual the engine raises no event for, such as `PostInitializeComponents`, a component's `TickComponent` or a widget's `NativeConstruct`, has one on Rusteal's bases: see [Engine virtuals](virtuals.md).

## Parameter types

| The event takes | The Rust method takes |
|---|---|
| A number or a bool | the Rust scalar (`f32`, `i32`, `bool`...) |
| An object | `UObjectRef<T>` |
| A class | `SubclassOf<T>` |
| A struct (`FHitResult`, `FVector`) | `UStructRef<T>`, a reference to the struct inside the call; `to_owned()` keeps a copy |
| An enum | the enum type or its `u8` value |
| A string, a name or a text | `String`, `FName`, `Text` |
| An array, a map or a set | `Vec<T>`, `HashMap<K, V>`, `HashSet<T>`, copies |
| A soft or weak reference, an interface | `SoftObjectRef<T>`, `SoftClassRef<T>`, `TWeakObjectPtr<T>`, `Option<ScriptInterface<dyn I>>` |
| An output struct | `UStructRef<T>`, written in place |
| An output scalar or object | `OutRef<T>`, written with `set` |

An event that returns a value returns it from the method:

```rust
#[ufunction(Override)]
fn choose_player_start(&mut self, _player: UObjectRef<Controller>) -> UObjectRef<Actor> {
    let starts = GameplayStatics::get_all_actors_of_class(
        self.as_ref().upcast_to(),
        SubclassOf::<PlayerStart>::base().upcast_to(),
    );

    starts.first().copied().unwrap_or_default()
}
```

## Events whose name is not the method's

When the event's name is not the method's name in PascalCase, `name` gives it:

```rust
const MOVE_FALLING: u8 = 3;

#[ufunction(Override, name = "K2_OnMovementModeChanged")]
fn on_movement_mode_changed(
    &mut self,
    _prev_movement_mode: u8,
    new_movement_mode: u8,
    _prev_custom_mode: u8,
    _new_custom_mode: u8,
) {
    if new_movement_mode == MOVE_FALLING {
    }
}
```

## Finding an event's name and parameters

An event is any function the parent class lets Blueprints implement. To find one:

- in a Blueprint child of the parent class, right-click the **Event Graph** and look under **Add Event**: the node's title is the event, and hovering it shows its name;
- in the engine's C++ headers, events are the `UFUNCTION(BlueprintImplementableEvent)` and `UFUNCTION(BlueprintNativeEvent)` declarations of the class, many named `Receive...` or `K2_...`;
- in the generated bindings (`Rust/bindings/src/`), the class's module lists them with their parameters.

## A Blueprint child must not implement the same event

A Blueprint child that implements the same event in its graph replaces the Rust one: the engine runs the Blueprint's graph instead. Keep the Blueprint children of Rust classes for data (meshes, sounds, input actions), and if a Blueprint needs to react too, give it an event of its own (see [Functions](functions.md), `BlueprintImplementableEvent`).
