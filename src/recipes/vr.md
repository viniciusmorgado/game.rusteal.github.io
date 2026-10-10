# Virtual reality

The `vr` template is the engine's Virtual Reality template with its gameplay in Rust: an OpenXR pawn that teleports along the navigation mesh and snap turns, hands that grab objects and take poses from the controllers, a pistol, a 3D menu, passthrough and a spectator camera.

```bash
rusteal new MyVR --template vr
```

Open the project and press **Play > VR Preview** with an OpenXR runtime and a headset connected. Without a headset, **Play** in the viewport runs the same level with a non-tracked pawn.

The project binds the XR modules, the navigation system the teleports land on and Niagara, which the template turns on (see [Binding more of the engine](../cli/rusteal_toml.md)):

```toml
features = ["core", "engine", "input", "slate", "umg", "enhanced-input", "xr", "openxr", "ai", "niagara"]

[codegen.modules]
HeadMountedDisplay = { module = "head_mounted_display", feature = "xr" }
XRBase = { module = "xr_base", feature = "xr", plugin = "XRBase" }
OpenXRHMD = { module = "openxr", feature = "openxr", plugin = "OpenXR" }
```

`head_mounted_display` has the motion controller component; `xr_base` the headset's function library (`HeadMountedDisplayFunctionLibrary`), the loading screen and the device visualizations; `openxr` the passthrough blend modes.

## Use case: a pawn with a headset and two controllers

**The goal:** the camera follows the headset and four motion controllers follow the hands, two at the grip and two at the aim pose.

```rust
use bindings::engine::{CameraComponent, Pawn, SceneComponent};
use bindings::head_mounted_display::{MotionControllerComponent, MotionControllerComponentExt};

#[uclass(parent = Pawn)]
pub struct XRPawn {
    #[component(root, name = "VROrigin")]
    vr_origin: SceneComponent,

    #[component(attach = "vr_origin")]
    camera: CameraComponent,

    #[component(attach = "vr_origin")]
    motion_controller_left_grip: MotionControllerComponent,

    #[component(attach = "vr_origin")]
    motion_controller_right_aim: MotionControllerComponent,
}

#[uclass_impl]
impl XRPawn {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        self.motion_controller_left_grip()?
            .checked()?
            .set_tracking_motion_source(FName::new("LeftGrip").handle());

        self.motion_controller_right_aim()?
            .checked()?
            .set_tracking_motion_source(FName::new("RightAim").handle());

        Ok(())
    }

    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        if HeadMountedDisplayFunctionLibrary::is_head_mounted_display_enabled() {
            HeadMountedDisplayFunctionLibrary::set_tracking_origin(EHMDTrackingOrigin::LocalFloor);
        }
    }
}
```

The camera of a pawn whose root sits on the floor (`LocalFloor`) is moved by the headset; the motion controllers by the controllers. `get_motion_source()` reads which pose a controller follows (`LeftGrip`, `RightAim`...), which tells the code which hand it is.

## Use case: teleporting along an arc

**The goal:** the right thumbstick aims an arc; where it lands on the navigation mesh, the player can teleport, with their head over the spot.

The arc is a projectile path traced against the world, and its end is projected onto the navigation mesh:

```rust
let (_, hit, path, _) = GameplayStatics::blueprint_predict_projectile_path_by_object_type(
    me.upcast_to(),
    &FVector::from_dvec3(start),
    &FVector::from_dvec3(forward * 650.0),
    true,
    3.6,
    &[EObjectTypeQuery::ObjectTypeQuery1],
    false,
    &[],
    EDrawDebugTrace::None,
    0.0,
    Some(15.0),
    Some(2.0),
    Some(0.0),
);

let (valid, nav_point) = NavigationSystemV1::k2_project_point_to_navigation(
    me.upcast_to(),
    &FVector::from_dvec3(hit.as_ref().get_location().to_dvec3()),
    UObjectRef::null(),
    SubclassOf::null(),
    &self.teleport_project_point_to_navigation_query_extent(),
);
```

The path's points go to the arc's Niagara system as an array parameter:

```rust
NiagaraDataInterfaceArrayFunctionLibrary::set_niagara_array_vector(
    self.teleport_trace_niagara_system()?,
    FName::new("User.PointArray").handle(),
    &positions,
);
```

The teleport keeps the player's head over the target: the pawn moves by the target minus the camera's offset from the pawn, rotated by the pawn's yaw. The model function, tested without the engine:

```rust
pub fn teleport_destination(projected: DVec3, camera_relative: DVec3, actor_yaw: f64) -> DVec3 {
    projected - rotate_yaw(DVec3::new(camera_relative.x, camera_relative.y, 0.0), actor_yaw)
}
```

`ObjectTypeQuery1` is **WorldStatic**: the arc lands on the level, not on objects. The template's level has a **Nav Mesh Bounds Volume** around the floor and a **Nav Modifier Volume** where teleports are not allowed.

## Use case: snap turning around the head

**The goal:** the left thumbstick turns the player by 45 degrees without moving their head.

```rust
let delta = model::snap_turn_delta(f64::from(self.snap_turn_degrees()), right_turn);
let camera_world = camera.k2_get_component_location().to_dvec3();
let camera_relative = camera.get_relative_location().to_dvec3();
let new_yaw = me.k2_get_actor_rotation().to_rotator().yaw + delta;

me.k2_add_actor_world_rotation(&FRotator::from_rotator(Rotator::new(0.0, delta, 0.0)), false, true);
me.k2_set_actor_location(
    &FVector::from_dvec3(model::snap_turn_location(camera_world, camera_relative, new_yaw)),
    false,
    true,
);
```

Rotating the pawn alone would swing the head around the pawn's origin; moving the pawn by the difference keeps the head where it was.

## Use case: grabbing objects

**The goal:** any object with a grab component can be picked up with the grip; some snap into the hand, others keep the hold they were taken with.

The grab component is a `SceneComponent` attached to the object's physics body. Its grab type is a [`#[uenum]`](../classes/enums.md), and it tells the object when it is grabbed and dropped with [`#[udelegate]`s](../engine_api/delegates.md):

```rust
#[uenum]
pub enum EGrabType {
    None = 0,
    Snap = 1,
    Custom = 3,
    Free = 4,
}

#[uclass(parent = SceneComponent)]
pub struct GrabComponent {
    #[uproperty(EditAnywhere, BlueprintReadWrite, category = "Grab", default = EGrabType::Free)]
    grab_type: EGrabType,

    #[uproperty(BlueprintReadOnly, category = "Grab")]
    motion_controller_ref: UObjectRef<MotionControllerComponent>,

    #[uproperty(BlueprintReadOnly, category = "Grab")]
    b_is_held: bool,
}

#[uclass_impl]
impl GrabComponent {
    #[udelegate]
    pub fn on_grabbed(&self) {}

    #[udelegate]
    pub fn on_dropped(&self) {}

    #[ufunction(BlueprintCallable)]
    pub fn try_grab(&mut self, motion_controller: UObjectRef<MotionControllerComponent>) -> bool {
        match self.grab_type() {
            EGrabType::Free | EGrabType::Snap => {
                self.set_primitive_comp_physics(false);
                self.attach_parent_to_motion_controller(motion_controller);
                self.set_b_is_held(true);
            }
            EGrabType::Custom => self.set_b_is_held(true),
            EGrabType::None => {}
        }

        if self.b_is_held() {
            self.set_motion_controller_ref(motion_controller);
            self.on_grabbed();
        }

        self.b_is_held()
    }
}
```

The pawn finds what a hand can take with a sphere trace for physics bodies at the grip, and picks the nearest grab component of the actor it hits:

```rust
let (hit, result) = KismetSystemLibrary::sphere_trace_single_for_objects(
    me.upcast_to(), &point, &point, self.grab_radius_from_grip_position(),
    &[EObjectTypeQuery::ObjectTypeQuery4], false, &[], EDrawDebugTrace::None, true, &white, &green, Some(1.0),
);
```

`ObjectTypeQuery4` is **PhysicsBody**: grabbable objects use the **PhysicsActor** collision preset, which the grab component sets on its parent at begin play.

## Use case: an object that reacts to being held

**The goal:** a pistol that takes the trigger input of the hand holding it, and hides that hand while it is held.

The pistol listens to its grab component's delegates, enables input on itself and adds the hand's weapon mapping context:

```rust
let grabbed = grab.on_grabbed_add(move || {
    if let Ok(mut pistol) = Pistol::from_obj(me) {
        let _ = pistol.grabbed();
    }
});
```

```rust
fn grabbed(&mut self) -> RustealResult<()> {
    let me = self.as_ref();
    let controller = GameplayStatics::get_player_controller(me.upcast_to(), 0);
    me.checked()?.enable_input(controller);

    bind_action(&me, self.shoot_left_action(), ETriggerEvent::Started, "ShootLeft")?;
    bind_action(&me, self.shoot_right_action(), ETriggerEvent::Started, "ShootRight")?;

    let subsystem = enhanced_input_subsystem(controller)?.checked()?;
    subsystem.add_mapping_context(self.weapon_context()?, 1, &OwnedStruct::new());
    Ok(())
}
```

The pawn hides the hand through the template's Blueprint Interface, `BPI_PawnAnim`, which the pawn implements (`implements = ["/Game/XRFramework/Blueprints/BPI_PawnAnim.BPI_PawnAnim_C"]`) and the pistol calls on whoever holds it, without knowing its class:

```rust
let mut call = DynamicCall::new(&owner, "HideUnhideHand")?;
call.set("bHide", true)?;
call.set("bRightHand", self.held_by_right()?)?;
call.call()?;
```

Firing spawns the projectile at the muzzle and plays a haptic effect on the hand holding the pistol:

```rust
controller.play_haptic_effect(self.pistol_fire_haptic_effect(), self.grab()?.get_held_by_hand(), Some(1.0), Some(false));
```

## Use case: a 3D menu

**The goal:** a menu the player opens with the menu button, floating in front of the controller, used with the other hand as a laser pointer or with the thumbstick as a cursor.

The menu is an actor with a **Widget** component showing a Widget Blueprint, and the pawn's two **Widget Interaction** components (pointer index 0 and 1) are what press its buttons:

```rust
fn press(interaction: UObjectRef<WidgetInteractionComponent>, pressed: bool) {
    let Ok(interaction) = interaction.checked() else {
        return;
    };

    let key = FKey::named("LeftMouseButton");

    if pressed {
        interaction.press_pointer_key(&key);
    } else {
        interaction.release_pointer_key(&key);
    }
}
```

The other hand's interaction component reports whether it points at the menu (`is_over_hit_test_visible_widget`) and where (`get_last_hit_result`): the menu draws the laser there; otherwise the menu hand's thumbstick moves a cursor over the widget, its value read with `EnhancedInputLibrary::get_bound_action_value`. The menu's buttons are bound with `BindWidget`, as in [UI](ui.md#use-case-the-buttons-of-a-widget-blueprint).

The menu opens and closes by scaling its widget over 0.2 seconds, a Blueprint timeline that is a few lines of state played in the actor's tick (`model::Timeline`).

## Use case: hand poses from the controllers

**The goal:** the hands grasp, curl the index finger, point and raise the thumb as the player does on the controller.

The pawn binds the hand input actions and sets the targets on its hands' Anim Blueprint, whose logic is in a Rust `AnimInstance`; see [Animation](animation.md#use-case-the-animation-blueprints-logic-in-rust).

## Use case: passthrough

**The goal:** a menu button switches between the virtual room and the real one, on headsets that show the camera feed.

```rust
if OpenXRBlueprintFunctionLibrary::get_environment_blend_mode() != EOpenXREnvironmentBlendMode::AlphaBlend {
    OpenXRBlueprintFunctionLibrary::set_environment_blend_mode(EOpenXREnvironmentBlendMode::AlphaBlend);
    self.hide_passthrough_actors(true);
} else {
    OpenXRBlueprintFunctionLibrary::set_environment_blend_mode(EOpenXREnvironmentBlendMode::Opaque);
    self.hide_passthrough_actors(false);
}
```

Actors tagged **Passthrough** are hidden in passthrough mode (and lose their collision, except those also tagged **Floor**). `get_supported_environment_blend_modes()` lists what the runtime supports; without **AlphaBlend**, passthrough needs the headset vendor's OpenXR plugin.

## Use case: a spectator camera

**The goal:** a second camera, on the monitor, that people around the player control with the keyboard and a gamepad.

The spectator is an actor with a **Scene Capture Component 2D** rendering to a render target the headset's spectator screen shows:

```rust
if HeadMountedDisplayFunctionLibrary::is_head_mounted_display_enabled() {
    HeadMountedDisplayFunctionLibrary::set_spectator_screen_mode(ESpectatorScreenMode::Texture);
    HeadMountedDisplayFunctionLibrary::set_spectator_screen_texture(self.spectator_texture());
} else {
    controller.set_view_target_with_blend(me.upcast_to(), Some(0.0), None, Some(0.0), Some(false));
}
```

Without a headset it becomes the view target, which makes it testable in the viewport. Its fade writes the capture's post process **Color Gain**:

```rust
let settings = capture.get_post_process_settings();
settings.as_ref().set_override_color_gain(true);
settings.as_ref().set_color_gain(&FVector4::from_dvec4(DVec4::new(gain, gain, gain, 1.0)));
capture.set_post_process_settings(&settings);
```

> **Screenshot:** the vr template's level in the viewport, with the spectator camera, the cubes and the pistols. *(images/recipes/vr_level.png)*

## Testing without a headset

On a machine without a headset, Play In Editor runs the level with the pawn untracked. The template's code can be checked there by calling what the input would call: `StartTeleportTrace`, `TeleportTrace`, `TryTeleport`, `SnapTurn`, `ToggleMenu` on the pawn, `Fire` on a pistol, `EnableVRSpectator` on the spectator. The hand and menu input actions only fire with a headset's controllers.
