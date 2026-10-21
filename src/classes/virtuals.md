# Engine virtuals

**The goal:** run Rust code where a C++ class would override a virtual function the engine raises no event for: after an actor's components are initialized, on a component's own tick, in a custom movement mode, in a widget's native construct, in the animation update that runs on a worker thread.

```rust
use bindings::rusteal::RustealActor;

#[uclass(parent = RustealActor)]
pub struct Turret {
    #[uproperty(EditAnywhere)]
    range: f32,
}

#[uclass_impl]
impl Turret {
    #[ufunction(Override)]
    fn receive_post_initialize_components(&mut self) {
    }
}
```

A Rust class is an Unreal class created when the library loads: it overrides the events of its parent, as a Blueprint does, not its C++ virtual functions (see [Engine events](events.md#c-virtuals-are-events)). For the virtuals with no event, Rusteal's plugin has bases (since 0.10): C++ classes that override them and raise an event, which the Rust class overrides. Use the base as the parent, `RustealActor` instead of `Actor`; everything the parent has, the base has.

## The bases

| Parent | Base | Virtual | Event (Rust method) |
|---|---|---|---|
| `Actor`, `Pawn`, `Character`, `PlayerController`, `GameModeBase` | `RustealActor`, `RustealPawn`, `RustealCharacter`, `RustealPlayerController`, `RustealGameModeBase` | `PreInitializeComponents` | `receive_pre_initialize_components(&mut self)` |
| | | `PostInitializeComponents` | `receive_post_initialize_components(&mut self)` |
| | | `PostActorCreated` | `receive_post_actor_created(&mut self)` |
| | | `PostRegisterAllComponents` | `receive_post_register_all_components(&mut self)` |
| | | `OnConstruction` | `receive_on_construction(&mut self, transform: UStructRef<FTransform>)` |
| `ActorComponent`, `SceneComponent` | `RustealActorComponent`, `RustealSceneComponent` | `OnRegister`, `OnUnregister` | `receive_on_register(&mut self)`, `receive_on_unregister(&mut self)` |
| | | `InitializeComponent`, `UninitializeComponent` | `receive_initialize_component(&mut self)`, `receive_uninitialize_component(&mut self)` |
| | | `TickComponent` | `receive_tick_component(&mut self, delta_time: f32)` |
| `UserWidget` | `RustealUserWidget` | `NativeOnInitialized`, `NativePreConstruct`, `NativeConstruct`, `NativeDestruct` | `receive_on_initialized`, `receive_pre_construct`, `receive_construct`, `receive_destruct` |
| | | `NativeTick` | `receive_tick(&mut self, my_geometry: UStructRef<FGeometry>, in_delta_time: f32)` |
| `AnimInstance` | `RustealAnimInstance` | `NativeInitializeAnimation`, `NativeBeginPlay`, `NativePostEvaluateAnimation`, `NativeUninitializeAnimation` | `receive_initialize_animation`, `receive_begin_play`, `receive_post_evaluate_animation`, `receive_uninitialize_animation` |
| | | `NativeUpdateAnimation` | `receive_update_animation(&mut self, delta_seconds: f32)` |
| | | `NativeThreadSafeUpdateAnimation` | `receive_thread_safe_update_animation`, on a worker thread (below) |
| `CharacterMovementComponent` | `RustealCharacterMovementComponent` | `PhysCustom` | `receive_phys_custom(&mut self, delta_time: f32, iterations: i32)` |
| | | `GetMaxSpeed`, `GetMaxAcceleration` | `receive_get_max_speed(&self) -> f32`, `receive_get_max_acceleration(&self) -> f32` |
| | | `OnMovementModeChanged` | `receive_on_movement_mode_changed(&mut self, previous_movement_mode: EMovementMode, previous_custom_mode: u8)` |
| `Commandlet` | `RustealCommandlet` | `Main` | `receive_main(&mut self, params: String) -> i32` |
| `Actor` with `IGameplayTagAssetInterface` | `RustealTaggedActor` | `GetOwnedGameplayTags` | `receive_get_owned_gameplay_tags(&self, tag_container: UStructRef<FGameplayTagContainer>)` |

The event runs after the parent's version of the virtual, except for the ones that tear down (`OnUnregister`, `UninitializeComponent`, `NativeDestruct`, `NativeUninitializeAnimation`), which run before it. A virtual that returns a value is a Blueprint native event: the parent's version runs when the Rust class does not override it, and the override replaces it.

A component on a `RustealActorComponent` base ticks only when its class overrides `receive_tick_component`.

## Use case: a custom movement mode

**The goal:** a character that climbs, glides or swims in a way of its own, with `MOVE_Custom` and the character movement component's `PhysCustom`.

```rust
use bindings::rusteal::{RustealCharacter, RustealCharacterMovementComponent};

#[uclass(parent = RustealCharacterMovementComponent)]
pub struct ClimbingMovement {
    #[uproperty(EditAnywhere, default = 200.0)]
    climb_speed: f32,
}

#[uclass_impl]
impl ClimbingMovement {
    #[ufunction(Override)]
    fn receive_phys_custom(&mut self, delta_time: f32, _iterations: i32) {
        let up = f64::from(self.climb_speed() * delta_time);

        self.k2_move_updated_component(
            &FVector::from_dvec3(glam::DVec3::new(0.0, 0.0, up)),
            &FRotator::from_rotator(Rotator::ZERO),
            Some(true),
            Some(false),
        );
    }

    #[ufunction(Override)]
    fn receive_get_max_speed(&self) -> f32 {
        self.climb_speed()
    }
}

#[uclass(parent = RustealCharacter, subobject_class(CharMoveComp = ClimbingMovement))]
pub struct Climber {}
```

`subobject_class(CharMoveComp = ClimbingMovement)` (since 0.10) creates the character's inherited movement component as the Rust class, as C++'s `SetDefaultSubobjectClass` does: the name is the one the parent gives the subobject (`ACharacter::CharacterMovementComponentName`, `CharMoveComp`). It works for any inherited subobject whose class the new one derives from. `set_movement_mode(EMovementMode::MOVE_Custom, Some(0))` then enters the mode; the second argument tells several custom modes apart.

## Use case: a commandlet

**The goal:** a tool that runs the editor without a window, as a step of a build or a batch job: validate every level, export a data table.

```rust
use bindings::rusteal::RustealCommandlet;

#[uclass(parent = RustealCommandlet)]
pub struct ValidateLevelsCommandlet {}

#[uclass_impl]
impl ValidateLevelsCommandlet {
    #[ufunction(Override)]
    fn receive_main(&mut self, params: String) -> i32 {
        log::info!("validating with {params}");
        0
    }
}
```

Run it with the editor binary, the project and `-run=` with the class name; the process exits with the number `receive_main` returns:

```bash
UnrealEditor MyGame.uproject -run=ValidateLevelsCommandlet -unattended -nullrhi
```

On Windows the binary is `UnrealEditor-Cmd.exe`, in the engine's `Engine/Binaries/Win64`.

## Use case: an animation update on a worker thread

**The goal:** the animation instance computes its pose variables off the game thread, where the engine runs `NativeThreadSafeUpdateAnimation` and the Blueprint thread-safe update.

```rust
use bindings::rusteal::RustealAnimInstance;

#[uclass(parent = RustealAnimInstance)]
pub struct HeroAnim {
    #[uproperty(BlueprintReadOnly)]
    speed: f32,

    #[uproperty]
    target_speed: f32,

    smoothing: f32,
}

#[uclass_impl]
impl HeroAnim {
    #[ufunction(Override)]
    fn receive_update_animation(&mut self, _delta_seconds: f32) {
        if let Ok(pawn) = self.try_get_pawn_owner().checked() {
            self.set_target_speed(pawn.get_velocity().to_dvec3().length() as f32);
        }
    }

    #[ufunction(Override, ThreadSafe)]
    fn receive_thread_safe_update_animation(this: HeroAnimThreadSafe, delta_seconds: f32) {
        let blend = (delta_seconds * 8.0).min(1.0);
        this.set_speed(this.speed() + (this.target_speed() - this.speed()) * blend);
    }
}
```

The engine runs the thread-safe update on a worker thread, while the game thread goes on with other objects. A function that runs there is a `#[ufunction(ThreadSafe)]` (since 0.10) and takes the class's thread-safe view, `HeroAnimThreadSafe`, instead of `&self` or `&mut self`. The view has the `#[uproperty]` fields' getters and setters only: the Rust-only fields (`smoothing` above) are not there, since the game thread may be using them at the same time. Read what the game thread computes in the game-thread update (`receive_update_animation`), and keep it in properties the view reaches.

Call no engine function from a thread-safe function but those Unreal marks thread safe, the math libraries' among them: the others belong to the game thread, as in C++.

A Rust function without `ThreadSafe` that the engine calls on another thread does not run, and the Output Log says which and why, once: an override of `blueprint_thread_safe_update_animation`, for one, takes `ThreadSafe` as well.

## A tick function of its own

**The goal:** code that runs every frame in another tick group than the actor's, or after another object has ticked: a camera that follows a character after its movement, a solver after physics.

```rust
use bindings::engine::ETickingGroup;
use rusteal_runtime::runtime::tick::TickFunction;

#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    let me = self.as_ref();

    let after_physics = add_tick_function(&me, ETickingGroup::TG_PostPhysics, move |delta| {
        if let Ok(mut rig) = CameraRig::from_obj(me) {
            rig.follow(delta);
        }
    });

    if let Ok(tick) = after_physics {
        let _ = tick.add_prerequisite(&self.target());
    }
}
```

`add_tick_function(&object, group, closure)` (since 0.10) registers a tick function in the world of `object`, called with the frame's delta time. It returns a `TickFunction`:

| Method | What it does |
|---|---|
| `add_prerequisite(&actor_or_component)` | Ticks after that actor's or component's tick. |
| `add_prerequisite_function(&other)` | Ticks after another tick function. |
| `set_interval(duration)` | Ticks every `duration` instead of every frame. |
| `set_enabled(bool)` | Stops and starts it. |
| `remove()` | Removes it. |

Dropping the `TickFunction` leaves it ticking. It is removed by itself when its object is destroyed, when its world goes away, and when the library unloads. An actor's own tick group and prerequisites are set with the engine's functions, as in Blueprints: `set_tick_group`, `add_tick_prerequisite_actor`, `add_tick_prerequisite_component`.

## Bases of your own

**The goal:** another virtual of another engine class, one Rusteal does not list.

```toml
[[codegen.shims]]
class = "RustealSpectatorPawn"
parent = "ASpectatorPawn"
include = ["GameFramework/SpectatorPawn.h"]
virtuals = [
    "void PossessedBy(AController *NewController)",
    "void UnPossessed()",
]
```

Each `[[codegen.shims]]` in the project's `rusteal.toml` (since 0.10) is a base that `rusteal build --all` generates in the project's copy of the plugin, before it builds the engine side: `RustealSpectatorPawn` in `bindings::rusteal`, with `receive_possessed_by` and `receive_un_possessed`.

| Key | What it is |
|---|---|
| `class` | The base's name, without its `A` or `U` prefix. |
| `parent` | The C++ class, with its prefix. |
| `include` | The headers that declare the parent and the parameter types. |
| `modules` | The engine modules they come from, when the bindings do not use them already. |
| `virtuals` | The virtuals, as their C++ declarations. |
| `category` | The Blueprint category of the events, `Rusteal|<class>` otherwise. |
| `constructor` | C++ lines for the base's constructor. |
| `interfaces` | C++-only interfaces the base implements, each with `name`, `include` and `virtuals`. |

A virtual is a string, its declaration, or a table with `signature` and:

| Key | What it does |
|---|---|
| `event` | The event's name, `Receive` and the virtual's name without `Native` otherwise. |
| `order = "before"` | Raises the event before the parent's version runs. |
| `drop = ["Name"]` | Leaves a parameter out of the event, for a type a Blueprint event cannot take (`ELevelTick`, `FActorComponentTickFunction*`). |
| `meta = ["BlueprintThreadSafe"]` | Function metadata for the event. |

The parameters an event takes are numbers, `bool`, objects and classes, Unreal enums, `FString`, `FName`, `FText` and Unreal structs, by value or by reference. A virtual whose parameters cannot all be passed is refused, and the build says which: drop them, or leave the virtual out. After changing `[[codegen.shims]]`, `rusteal build --all` builds the new base; `rusteal build` says so and builds only Rust.

## C++-only interfaces

**The goal:** a Rust class implements an engine interface Blueprints cannot implement, one the engine calls through C++: `IGameplayTagAssetInterface`, which makes `HasMatchingGameplayTag` and the gameplay tag queries see an actor's tags.

```rust
use bindings::gameplay_tags::{BlueprintGameplayTagLibrary, FGameplayTagContainer};
use bindings::rusteal::RustealTaggedActor;

#[uclass(parent = RustealTaggedActor)]
pub struct Door {
    #[uproperty(EditAnywhere)]
    tags: OwnedStruct<FGameplayTagContainer>,
}

#[uclass_impl]
impl Door {
    #[ufunction(Override)]
    fn receive_get_owned_gameplay_tags(&self, tag_container: UStructRef<FGameplayTagContainer>) {
        tag_container.assign(&self.tags());
    }
}
```

`assign` (since 0.10) replaces the struct an event's output parameter refers to. For another interface, declare a base that implements it, under `interfaces` in `[[codegen.shims]]`:

```toml
[[codegen.shims]]
class = "RustealTeamPawn"
parent = "APawn"
include = ["GameFramework/Pawn.h"]
modules = ["AIModule"]

[[codegen.shims.interfaces]]
name = "IGenericTeamAgentInterface"
include = ["GenericTeamAgentInterface.h"]
virtuals = ["FGenericTeamId GetGenericTeamId() const"]
```

An interface's virtual is a native event with an empty default: the Rust class overrides it.
