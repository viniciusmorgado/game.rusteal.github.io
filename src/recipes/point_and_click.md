# Point and click

**The goal:** click where the character should go, and it walks there along the navigation mesh; hold the button and it follows the cursor.

The `top-down` template's player controller does both: a short click pathfinds to the point, a held click steers toward the cursor.

```rust
use bindings::ai::{AIBlueprintHelperLibrary, PathFollowingComponent};
use bindings::core_ue::EMouseCursor;
use bindings::engine::{
    ControllerExt, ETraceTypeQuery, FHitResultExt, GameplayStatics, PawnExt, PlayerController,
    PlayerControllerExt
};
use bindings::enhanced_input::{
    ETriggerEvent, EnhancedInputLocalPlayerSubsystemExt, InputAction, InputMappingContext
};
use rusteal_runtime::runtime::method;

#[uclass(parent = PlayerController)]
pub struct ClickController {
    #[component(name = "Path Following Component")]
    path_following_component: PathFollowingComponent,

    #[uproperty(EditAnywhere, category = "Input", default = 0.3)]
    short_press_threshold: f32,

    #[uproperty(EditAnywhere, category = "Input")]
    default_mapping_context: UObjectRef<InputMappingContext>,

    #[uproperty(EditAnywhere, category = "Input")]
    set_destination_click_action: UObjectRef<InputAction>,

    cached_destination: DVec3,
    follow_time: f32,
}

#[uclass_impl]
impl ClickController {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        me.set_show_mouse_cursor(true);
        me.set_default_mouse_cursor(EMouseCursor::Default);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        if let Err(e) = self.setup_input() {
            log::warn!("[MyGame] input setup failed: {e}");
        }
    }

    #[ufunction]
    fn on_input_started(&mut self) {
        if let Ok(me) = self.as_ref().checked() {
            me.stop_movement();
        }
    }

    #[ufunction]
    fn on_set_destination_triggered(&mut self) {
        let Ok(me) = self.as_ref().checked() else {
            return;
        };

        let delta = GameplayStatics::get_world_delta_seconds(me.as_ref().upcast_to()) as f32;
        self.set_follow_time(self.follow_time() + delta);

        let (hit, out_hit) = me.get_hit_result_under_cursor_by_channel(ETraceTypeQuery::TraceTypeQuery1, true);

        if hit {
            self.set_cached_destination(out_hit.as_ref().get_location().to_dvec3());
        }

        if let Ok(pawn) = me.k2_get_pawn().checked() {
            let direction = (self.cached_destination() - pawn.k2_get_actor_location().to_dvec3()).normalize_or_zero();
            pawn.add_movement_input(&FVector::from_dvec3(direction), Some(1.0), Some(false));
        }
    }

    #[ufunction]
    fn on_set_destination_released(&mut self) {
        if self.follow_time() <= self.short_press_threshold() {
            let destination = FVector::from_dvec3(self.cached_destination());
            AIBlueprintHelperLibrary::simple_move_to_location(self.as_ref().upcast_to(), &destination);
        }

        self.set_follow_time(0.0);
    }
}

impl ClickController {
    fn setup_input(&self) -> RustealResult<()> {
        let me: UObjectRef<PlayerController> = self.as_ref();

        if !me.checked()?.is_local_player_controller() {
            return Ok(());
        }

        enhanced_input_subsystem(me)?
            .checked()?
            .add_mapping_context(self.default_mapping_context(), 0, &OwnedStruct::new());

        let click = self.set_destination_click_action();
        self.bind_input(click, ETriggerEvent::Started, Self::on_input_started)?;
        self.bind_input(click, ETriggerEvent::Triggered, Self::on_set_destination_triggered)?;
        self.bind_input(click, ETriggerEvent::Completed, Self::on_set_destination_released)?;
        self.bind_input(click, ETriggerEvent::Canceled, Self::on_set_destination_released)?;

        Ok(())
    }

    fn bind_input(
        &self,
        action: UObjectRef<InputAction>,
        trigger_event: ETriggerEvent,
        handler: fn(&mut Self),
    ) -> RustealResult<()> {
        let mut handler = method(self, handler);
        bind_action(&self.as_ref(), action, trigger_event, move |_| handler())?;

        Ok(())
    }
}
```

- The handlers take no input value, so `bind_input` wraps each one, made with `method`, in a closure that ignores the value. A misspelled handler is a compile error.

- `simple_move_to_location` is the Blueprint node **Simple Move to Location**: it pathfinds for the controller's pawn, which needs the `PathFollowingComponent` the controller declares.
- The project needs the `ai` feature for `bindings::ai` (see [Binding more of the engine](../cli/rusteal_toml.md)), and the level a **Nav Mesh Bounds Volume**.
- Touch works the same with `get_hit_result_under_finger_by_channel(ETouchIndex::Touch1, ...)` and a second input action.

The template also spawns a Niagara effect at the clicked point: see [Effects and sound](effects.md).
