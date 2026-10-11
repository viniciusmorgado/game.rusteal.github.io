# AI

AI in Rusteal uses the engine's tools: AI controllers, StateTree, EQS, AI perception and the navigation mesh. Rust writes the controller, the StateTree tasks and conditions, and the EQS contexts; the StateTree and the EQS queries are assets made in the editor.

The project needs the `ai` feature, and `state-tree` for StateTree, in `rusteal.toml` (see [Binding more of the engine](../cli/rusteal_toml.md)):

```toml
features = ["core", "engine", "input", "slate", "umg", "enhanced-input", "ai", "state-tree"]
```

## Use case: an AI controller that runs a StateTree

**The goal:** an enemy's controller that starts its StateTree when it possesses the enemy.

```rust
use bindings::ai::{AIController, AIControllerExt, BrainComponentExt};
use bindings::engine::{ControllerExt, Pawn};
use bindings::gameplay_state_tree::{StateTreeAIComponent, StateTreeComponentExt};

#[uclass(parent = AIController)]
pub struct EnemyAIController {
    #[component(name = "StateTreeAI")]
    state_tree_ai: StateTreeAIComponent,
}

#[uclass_impl]
impl EnemyAIController {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        me.set_start_ai_logic_on_possess(false);
        me.set_attach_to_pawn(true);
        self.state_tree_ai()?.checked()?.set_start_logic_automatically(false);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_possess(&mut self, _possessed_pawn: UObjectRef<Pawn>) {
        if let Ok(state_tree) = self.state_tree_ai().and_then(|s| s.checked()) {
            state_tree.start_logic();
        }
    }
}
```

Starting the StateTree on possession, instead of automatically, makes sure the tree finds its pawn: started earlier, it logs that it could not find its context actor.

The enemy uses it, in its class defaults:

```rust
me.set_ai_controller_class(SubclassOf::<EnemyAIController>::base().upcast_to());
me.set_auto_possess_ai(EAutoPossessAI::PlacedInWorldOrSpawned);
```

The controller's Blueprint child sets the **State Tree** of the **StateTreeAI** component.

## Use case: a StateTree task

**The goal:** a task that makes the AI face an actor while its state is active.

```rust
use bindings::ai::{AIController, AIControllerExt};
use bindings::state_tree::{FStateTreeTransitionResult, StateTreeTaskBlueprintBase, StateTreeTaskBlueprintBaseExt};

#[uclass(parent = StateTreeTaskBlueprintBase)]
pub struct StateTreeFaceActorTask {
    #[uproperty(EditAnywhere, category = "Context")]
    controller: UObjectRef<AIController>,

    #[uproperty(EditAnywhere, category = "Input")]
    actor_to_face_towards: UObjectRef<Actor>,
}

#[uclass_impl]
impl StateTreeFaceActorTask {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        self.as_ref().checked()?.set_should_state_change_on_reselect(false);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_latent_enter_state(&mut self, _transition: UStructRef<FStateTreeTransitionResult>) {
        if let Ok(controller) = self.controller().checked() {
            controller.k2_set_focus(self.actor_to_face_towards());
        }
    }

    #[ufunction(Override)]
    fn receive_exit_state(&mut self, _transition: UStructRef<FStateTreeTransitionResult>) {
        if let Ok(controller) = self.controller().checked() {
            controller.k2_clear_focus();
        }
    }
}
```

- A task's parent is `StateTreeTaskBlueprintBase`; it overrides `receive_latent_enter_state`, `receive_exit_state` and `receive_tick`.
- The property categories are the StateTree's binding categories: **Context** binds to the tree's context (the actor, the controller), **Input** and **Parameter** to values, **Output** to values other tasks read.
- A task that finishes on its own calls `finish_task(Some(true))` (succeeded) or `finish_task(Some(false))` on itself; one that runs until its state is left does not.

```rust
#[ufunction(Override)]
fn receive_latent_enter_state(&mut self, _transition: UStructRef<FStateTreeTransitionResult>) {
    let found = self.select_target();

    if let Ok(me) = self.as_ref().checked() {
        me.finish_task(Some(found));
    }
}
```

## Use case: a StateTree condition

```rust
use bindings::state_tree::StateTreeConditionBlueprintBase;

#[uclass(parent = StateTreeConditionBlueprintBase)]
pub struct StateTreeIsGroundedCondition {
    #[uproperty(EditAnywhere, category = "Context")]
    character: UObjectRef<Character>,

    #[uproperty(EditAnywhere, category = "Condition", default = false)]
    b_must_be_in_air: bool,
}

#[uclass_impl]
impl StateTreeIsGroundedCondition {
    #[ufunction(Override)]
    fn receive_test_condition(&self) -> bool {
        let Ok(movement) = self.character().checked().and_then(|c| c.get_character_movement().checked()) else {
            return false;
        };

        movement.is_moving_on_ground() != self.b_must_be_in_air()
    }
}
```

### In the editor

1. Open the StateTree asset (`ST_Enemy`).
2. Select a state. In the **Details** panel, add a task under **Tasks** and pick **State Tree Face Actor Task** from the class list.
3. Bind the task's **Controller** to the tree's AI controller context, and its **Actor To Face Towards** to an output of another task.
4. Add the condition the same way, under the state's **Enter Conditions** or a transition's **Conditions**, and pick **State Tree Is Grounded Condition**.
5. **Compile** and **Save**.

Rust tasks and conditions are listed under their class names, like Blueprint ones.

> **Screenshot:** a StateTree state with the Rust task added and its Controller bound to the context. *(images/recipes/state_tree_task.png)*

## Use case: AI perception

**The goal:** an enemy that reacts to what it sees and hears.

```rust
use bindings::ai::{AIPerceptionComponent, AIPerceptionComponentExt, FAIStimulus, FAIStimulusExt};

#[uclass(parent = AIController)]
pub struct GuardController {
    #[component(name = "AIPerception")]
    ai_perception: AIPerceptionComponent,
}

#[uclass_impl]
impl GuardController {
    #[ufunction(Override)]
    fn receive_possess(&mut self, _possessed_pawn: UObjectRef<Pawn>) {
        if let Err(e) = self.bind_perception() {
            log::warn!("[MyGame] AI perception: {e}");
        }
    }
}

impl GuardController {
    fn bind_perception(&self) -> RustealResult<()> {
        let me: UObjectRef<AIController> = self.as_ref();

        self.ai_perception()?
            .checked()?
            .on_target_perception_updated()
            .add(move |actor, stimulus| {
                if let Ok(guard) = GuardController::from_obj(me) {
                    guard.on_perceived(actor, stimulus);
                }
            })?
            .detach();

        Ok(())
    }

    fn on_perceived(&self, actor: UObjectRef<Actor>, stimulus: OwnedStruct<FAIStimulus>) {
        let location = stimulus.as_ref().get_stimulus_location().to_dvec3();
        let heard_a_shot = FName(stimulus.as_ref().get_tag()) == FName::new("Shot");
    }
}
```

The senses (sight, hearing, their ranges and angles) are set on the **AIPerception** component in the controller's Blueprint child: **Senses Config**, **+**, **AI Sight config**. Noises come from `make_noise` (see [Projectiles and weapons](projectiles.md)), which also needs a **Pawn Noise Emitter** component on the player.

## Use case: moving to a point

```rust
let result = ai_controller.move_to_location(
    &FVector::from_dvec3(goal),
    Some(100.0),
    Some(true),
    Some(true),
    Some(true),
    Some(false),
    None,
    Some(true),
);

if result == EPathFollowingRequestResult::AlreadyAtGoal {
}
```

To know when the move ends, bind the controller's `receive_move_completed` delegate:

```rust
controller
    .receive_move_completed()
    .add(move |_request_id, _result| {
        if let Ok(mut unit) = Unit::from_obj(me) {
            unit.on_arrived();
        }
    })?
    .detach();
```

A level needs a **Nav Mesh Bounds Volume** for the AI to find paths.

## Use case: running an EQS query from Rust

**The goal:** a unit asks EQS for the best point around a goal and moves there.

```rust
use bindings::ai::{EEnvQueryRunMode, EnvQuery, EnvQueryInstanceBlueprintWrapper, EnvQueryInstanceBlueprintWrapperExt, EnvQueryManager};

let me: UObjectRef<Unit> = self.as_ref().cast().unwrap_or_default();

let query_instance = EnvQueryManager::run_eqs_query(
    me.upcast_to(),
    self.move_query(),
    me.upcast_to(),
    EEnvQueryRunMode::RandomBest25Pct,
    SubclassOf::<EnvQueryInstanceBlueprintWrapper>::base(),
);

self.set_env_query_instance(query_instance);

query_instance
    .checked()?
    .on_query_finished_event()
    .add(move |finished_query, _status| {
        if let Ok(mut unit) = Unit::from_obj(me) {
            unit.on_eqs_finished(finished_query);
        }
    })?
    .detach();
```

```rust
fn on_eqs_finished(&mut self, query: UObjectRef<EnvQueryInstanceBlueprintWrapper>) {
    let Ok(query) = query.checked() else {
        return;
    };

    let (found, locations) = query.get_query_results_as_locations();

    if let Some(best) = locations.first().filter(|_| found) {
        self.move_to(best.to_dvec3());
    }
}
```

Keep the query instance in a `#[uproperty]` until it finishes, or the garbage collector may collect it.

## Use case: an EQS context

**The goal:** EQS queries that score points around a location Rust knows, such as where danger was last seen.

```rust
use bindings::ai::EnvQueryContext_BlueprintBase;
use bindings::core_ue::Object;

#[uclass(parent = EnvQueryContext_BlueprintBase)]
pub struct EnvQueryContextDanger {}

#[uclass_impl]
impl EnvQueryContextDanger {
    #[ufunction(Override)]
    fn provide_single_location(
        &self,
        _querier_object: UObjectRef<Object>,
        querier_actor: UObjectRef<Actor>,
        resulting_location: UStructRef<FVector>,
    ) {
        if let Ok(enemy) = Enemy::from_obj(querier_actor) {
            let location = enemy.last_danger_location();
            resulting_location.set_x(location.x);
            resulting_location.set_y(location.y);
            resulting_location.set_z(location.z);
        }
    }
}
```

`provide_single_actor(&self, querier_object, querier_actor, resulting_actor: OutRef<UObjectRef<Actor>>)` provides an actor instead. In the EQS editor, the context shows in every **Context** dropdown under its class name.
