# Augmented reality

The `ar` template is the engine's Handheld AR template with its gameplay in Rust: a pawn that starts an AR session, shows the planes the device detects, places an object on the plane the player taps, and moves, rotates and scales it with one and two finger gestures, while the scene's lights follow the room's light. Its menus switch between three styles kept in a data table.

```bash
rusteal new MyAR --template ar
```

The template targets ARCore (Android) and ARKit (iOS) devices. Rusteal does not build for Android or iOS yet, so the project runs in the editor, where there is no AR session: the menus, the gestures and the placing can be tried there by calling the pawn's functions (see [Testing without a device](#testing-without-a-device)).

The project binds the engine's AugmentedReality module, which the template turns on (see [Binding more of the engine](../cli/rusteal_toml.md)):

```toml
features = ["core", "engine", "input", "slate", "umg", "enhanced-input", "ar"]

[codegen.modules]
AugmentedReality = { module = "augmented_reality", feature = "ar" }
```

`augmented_reality` has the session functions (`ARBlueprintLibrary`), the tracked geometries (`ARPlaneGeometry`), the trace results and the light estimates.

## Use case: starting the session

**The goal:** a button starts the AR session with the project's session configuration, and the menu moves on once the session runs and has found planes.

```rust
#[ufunction]
fn start_session(&mut self) {
    ARBlueprintLibrary::start_ar_session(self.session_config());
}
```

`session_config` is a `UObjectRef<ARSessionConfig>` property that the menu's Blueprint sets to `D_ARSessionConfig`. The menu's tick then waits for the session and for two seconds of scanning with at least one geometry:

```rust
if ARBlueprintLibrary::get_ar_session_status().as_ref().get_status() == EARSessionStatus::Running {
    start.to_scanning_state();
    self.set_session_has_started(true);
}

if f64::from(self.scan_timer()) > model::SCAN_SECONDS && !ARBlueprintLibrary::get_all_geometries().is_empty() {
    start.to_select_plane_state();
}
```

## Use case: reading the fingers

**The goal:** know, every frame, where the first two fingers are and when each one touches or leaves the screen.

The template polls the touches instead of binding input actions:

```rust
fn touch(&self, index: ETouchIndex) -> Touch {
    let controller = GameplayStatics::get_player_controller(self.as_ref().upcast_to(), 0);

    match controller.checked() {
        Ok(controller) => {
            let (x, y, pressed) = controller.get_input_touch_state(index);
            Touch { pressed, location: DVec2::new(f64::from(x), f64::from(y)) }
        }
        Err(_) => Touch::default(),
    }
}
```

The pawn keeps last frame's `Touch` for each finger: a finger is pressed when it is down now and was not before, released the other way round. The pinch the device recognises is an analog key, read with `get_input_analog_key_state(&FKey::named("Gesture_Pinch"))`.

## Use case: telling a drag from a twist and a pinch

**The goal:** one finger held for a moment drags the object; two fingers twist it or pinch it, whichever the player started.

The recognition is plain Rust in `model.rs`, tested with `cargo test`. A gesture is recognised after 0.2 seconds; with two fingers, a mostly horizontal movement is a twist and a vertical one a pinch:

```rust
self.recognised = true;
self.current = if fingers_direction.y.abs() < 0.5 {
    EGestureType::Rotation
} else {
    EGestureType::Scale
};
```

Once recognised, the gesture drives the placed object until no finger has touched the screen for 0.1 seconds. Snapping, turned on in the options menu, rounds the location to 5 cm, the rotation to 20 degrees and the scale to 0.1.

## Use case: placing an object on a plane

**The goal:** the first tap on a detected plane selects it; the second spawns the object where the finger touched it.

The tap becomes a ray from the camera through the finger:

```rust
let (_, origin, direction) = GameplayStatics::deproject_screen_to_world(
    controller,
    &FVector2D::from_dvec2(touch_location),
);
let start = origin.to_dvec3();
let end = start + direction.to_dvec3() * 1000.0;
```

traced with `KismetSystemLibrary::line_trace_single_for_objects`. When the actor hit is the selected plane, the pawn spawns the placeable there and tells the menus through its delegate:

```rust
let placeable = world.spawn_actor_of_class(self.placeable_class(), &transform)?;
Placeable::from_obj(placeable)?.set_bp_plane(self.selected_plane());
self.set_placeable_object(placeable);
self.ar_object_spawned();
```

`ar_object_spawned` is a `#[udelegate(name = "ARObjectSpawned")]` the main menu binds to show the bottom menu.

## Use case: showing the planes the device finds

**The goal:** while the player looks for a surface, the plane at the center of the screen is drawn, a different color for each plane.

Every second the pawn traces the tracked geometries through the center of the screen and spawns a plane actor with the geometry's transform and size:

```rust
let results = ARBlueprintLibrary::line_trace_tracked_objects(
    &FVector2D::from_dvec2(center),
    Some(false),
    Some(false),
    Some(true),
    Some(true),
);

let geometry = ARTraceResultLibrary::get_tracked_geometry(first);
let plane_geometry = geometry.cast::<ARPlaneGeometry>()?.checked()?;
let plane = world.spawn_actor_of_class(self.plane_class(), &plane_geometry.get_local_to_world_transform())?;
```

The color is the geometry's index among `ARBlueprintLibrary::get_all_geometries()`, modulo the ten colors of the pawn's `ColorArray`.

## Use case: lighting from the room

**The goal:** the virtual object is lit as brightly, and in the same tint, as the room the camera sees.

```rust
let estimate = ARBlueprintLibrary::get_current_light_estimate().cast::<ARBasicLightEstimate>();

let Ok(estimate) = estimate.and_then(|e| e.checked()) else {
    return;
};

let intensity = (estimate.get_ambient_intensity_lumens() / 1000.0) as f32;
component.set_intensity(intensity);
component.set_light_color(&estimate.get_ambient_color(), Some(true));
```

The pawn does it every tick for the level's directional light, and sets the sky light's intensity too. Without a session there is no estimate, and the lights keep their values.

## Use case: menu styles in a data table

**The goal:** the player switches the menus between a light, a dark and a game style; each style's colors, font and icons are a row of a data table.

The row is a Rust struct, used as the data table's row structure:

```rust
#[ustruct]
pub struct StructStyle {
    #[uproperty(EditAnywhere)]
    style: EUIStyle,

    #[uproperty(EditAnywhere)]
    text_color: OwnedStruct<FLinearColor>,

    #[uproperty(EditAnywhere)]
    font_size: i32,

    #[uproperty(EditAnywhere)]
    icons: UeArray<OwnedStruct<StructIcon>>,
}
```

Each menu reads the row of the style it is asked for:

```rust
for row_name in DataTableFunctionLibrary::get_data_table_row_names(table) {
    let Some(row) = find_data_table_row::<StructStyle>(table, FName(row_name)) else {
        continue;
    };

    if row.style() == current_style {
        return row;
    }
}
```

All the menus derive from a Rust `StylizedUI` widget with a `SwitchStyle` delegate; each binds its own function to it, and the main menu broadcasts the new style to them all. A data table made with a Blueprint struct moves to the Rust struct by filling it again from its JSON export with **Fill Data Table from JSON String**, the Rust struct as its row structure.

> **Screenshot:** the ar template's start menu in the editor viewport, with the Begin Scan button. *(images/recipes/ar_menu.png)*

## Testing without a device

In the editor, Play In Editor runs the level with no AR session, and the start menu waits. A plane actor placed in the level stands in for a detected one: `HandleTouchHit(plane, location)` on the pawn selects it, a second call places the object, and then `MoveOnPlane`, `SetPlaceableRotation` and `SetPlaceableScale` on the placeable, `ToggleSnap`, `ToggleHUD`, `SwitchStyle` and `ResetPlaceable` on the main menu do what the gestures and buttons would.
