# Vehicles

The `vehicle` template is the engine's Vehicle template with its gameplay in Rust: a sports car and an offroad car on Chaos Vehicles, two cameras, a speed and gear HUD, and a time trial variant with laps.

```bash
rusteal new Racer --template vehicle
rusteal new Racer --template vehicle --variant time-trial
```

The project needs the `chaos-vehicles` feature, which the template turns on (see [Binding more of the engine](../cli/rusteal_toml.md)):

```toml
features = ["core", "engine", "input", "slate", "umg", "enhanced-input", "chaos-vehicles"]

[codegen.modules]
ChaosVehicles = { module = "chaos_vehicles", feature = "chaos-vehicles", plugin = "ChaosVehiclesPlugin" }
```

## Use case: a car the player drives

**The goal:** a pawn on Chaos Vehicles, driven with Enhanced Input.

A car's parent is `WheeledVehiclePawn`, which brings the skeletal mesh and the Chaos movement component. The input functions pass the axis values to the movement component:

```rust
use bindings::chaos_vehicles::{
    ChaosVehicleMovementComponentExt, ChaosWheeledVehicleMovementComponent, WheeledVehiclePawn,
    WheeledVehiclePawnExt
};
use bindings::enhanced_input::{ETriggerEvent, FInputActionValue, InputAction};
use rusteal_runtime::runtime::{Checked, method};

#[uclass(parent = WheeledVehiclePawn)]
pub struct Car {
    #[uproperty(EditAnywhere, category = "Input")]
    steering_action: UObjectRef<InputAction>,

    #[uproperty(EditAnywhere, category = "Input")]
    throttle_action: UObjectRef<InputAction>,

    #[uproperty(EditAnywhere, category = "Input")]
    brake_action: UObjectRef<InputAction>,

    #[uproperty(EditAnywhere, category = "Input")]
    handbrake_action: UObjectRef<InputAction>,
}

#[uclass_impl]
impl Car {
    #[ufunction(Override)]
    fn receive_restarted(&mut self) {
        if let Err(e) = self.bind_input() {
            log::warn!("[Racer] input binding failed: {e}");
        }
    }

    #[ufunction]
    fn steering(&mut self, value: UStructRef<FInputActionValue>) {
        if let Ok(movement) = self.movement() {
            movement.set_steering_input(value.axis1d() as f32);
        }
    }

    #[ufunction]
    fn throttle(&mut self, value: UStructRef<FInputActionValue>) {
        if let Ok(movement) = self.movement() {
            movement.set_throttle_input(value.axis1d() as f32);
            movement.set_brake_input(0.0);
        }
    }

    #[ufunction]
    fn brake(&mut self, value: UStructRef<FInputActionValue>) {
        if let Ok(movement) = self.movement() {
            movement.set_brake_input(value.axis1d() as f32);
            movement.set_throttle_input(0.0);
        }
    }

    #[ufunction]
    fn start_handbrake(&mut self) {
        if let Ok(movement) = self.movement() {
            movement.set_handbrake_input(true);
        }
    }

    #[ufunction]
    fn stop_handbrake(&mut self) {
        if let Ok(movement) = self.movement() {
            movement.set_handbrake_input(false);
        }
    }
}

impl Car {
    pub fn movement(&self) -> RustealResult<Checked<ChaosWheeledVehicleMovementComponent>> {
        self.as_ref()
            .checked()?
            .get_vehicle_movement_component()
            .cast::<ChaosWheeledVehicleMovementComponent>()?
            .checked()
    }

    fn bind_input(&self) -> RustealResult<()> {
        let me = self.as_ref();

        if !me.checked()?.is_locally_controlled() {
            return Ok(());
        }

        let mut start_handbrake = method(self, Self::start_handbrake);
        let mut stop_handbrake = method(self, Self::stop_handbrake);

        bind_action(&me, self.steering_action(), ETriggerEvent::Triggered, method(self, Self::steering))?;
        bind_action(&me, self.steering_action(), ETriggerEvent::Completed, method(self, Self::steering))?;
        bind_action(&me, self.throttle_action(), ETriggerEvent::Triggered, method(self, Self::throttle))?;
        bind_action(&me, self.throttle_action(), ETriggerEvent::Completed, method(self, Self::throttle))?;
        bind_action(&me, self.brake_action(), ETriggerEvent::Triggered, method(self, Self::brake))?;
        bind_action(&me, self.brake_action(), ETriggerEvent::Completed, method(self, Self::brake))?;
        bind_action(&me, self.handbrake_action(), ETriggerEvent::Started, move |_| start_handbrake())?;
        bind_action(&me, self.handbrake_action(), ETriggerEvent::Completed, move |_| stop_handbrake())?;

        Ok(())
    }
}
```

- `get_vehicle_movement_component()` returns the base Chaos movement component; casting it to `ChaosWheeledVehicleMovementComponent` reaches the wheeled vehicle's settings.
- `method(self, Self::steering)` binds the method itself, checked by the compiler; the handbrake methods take no value, so a closure calls them and ignores it.
- Binding `Completed` to the same method as `Triggered` sends the final value of the axis (zero) when the key is released, so the car stops steering and accelerating.
- The car's mesh simulates physics and uses the **Vehicle** collision profile, set in class defaults: `mesh.set_simulate_physics(true)`, `mesh.set_collision_profile_name(FName::new("Vehicle").handle(), Some(true))`.

The Blueprint child (`BP_SportsCar_Pawn` in the template) assigns the skeletal mesh, its physics asset, the Animation Blueprint that turns the wheels, and the input actions.

## Use case: engine, gearbox and steering

**The goal:** tune how the car accelerates, shifts and turns, in code, as the C++ template's constructor does.

The movement component's setups are structs: `EngineSetup`, `TransmissionSetup`, `SteeringSetup`, `DifferentialSetup`. Read one, change its fields, and write it back, in class defaults:

```rust
use bindings::chaos_vehicles::{
    ChaosVehicleMovementComponentExt, ChaosWheeledVehicleMovementComponentExt, ESteeringType,
    FVehicleEngineConfigExt, FVehicleSteeringConfigExt, FVehicleTransmissionConfigExt
};

#[uclass(parent = Car)]
pub struct SportsCar {}

#[uclass_impl]
impl SportsCar {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let movement = Car::from_obj(self.as_ref())?.movement()?;

        movement.set_chassis_height(144.0);
        movement.set_drag_coefficient(0.31);

        let engine = movement.get_engine_setup();
        engine.as_ref().set_max_torque(750.0);
        engine.as_ref().set_max_rpm(7000.0);
        engine.as_ref().set_engine_idle_rpm(900.0);
        movement.set_engine_setup(&engine);

        let transmission = movement.get_transmission_setup();
        let gears = transmission.as_ref();
        gears.set_use_automatic_gears(true);
        gears.set_final_ratio(2.81);
        gears.set_change_up_rpm(6000.0);
        gears.set_change_down_rpm(2000.0);
        gears.forward_gear_ratios().set_all(&[4.25, 2.52, 1.66, 1.22, 1.0])?;
        gears.reverse_gear_ratios().set_all(&[4.04])?;
        movement.set_transmission_setup(&transmission);

        let steering = movement.get_steering_setup();
        steering.as_ref().set_steering_type(ESteeringType::Ackermann);
        steering.as_ref().set_angle_ratio(0.7);
        movement.set_steering_setup(&steering);

        Ok(())
    }
}
```

- The getter returns a copy of the struct; the setter writes the whole copy back. A field the code does not set keeps the engine's default.
- An array inside a struct (`forward_gear_ratios`) is a `UeArray` view into the copy: `set_all` replaces it, `push` adds to it.
- The template keeps these steps in a small `setup.rs` module, with one function per setup, so each car's class defaults read as a list of values.

## Use case: the wheels

**The goal:** front wheels that steer and rear wheels that drive, each car with its own tires.

A wheel is a class whose parent is `ChaosVehicleWheel`, configured in class defaults. The movement component creates one wheel object per entry of its `WheelSetups`, from the class and the bone the entry names:

```rust
use bindings::chaos_vehicles::{
    ChaosVehicleWheel, ChaosVehicleWheelExt, ChaosWheeledVehicleMovementComponentExt, EAxleType,
    FChaosWheelSetup, FChaosWheelSetupExt
};

#[uclass(parent = ChaosVehicleWheel)]
pub struct FrontWheel {}

#[uclass_impl]
impl FrontWheel {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let wheel = self.as_ref().checked()?;
        wheel.set_axle_type(EAxleType::Front);
        wheel.set_affected_by_steering(true);
        wheel.set_max_steer_angle(40.0);
        wheel.set_wheel_radius(39.0);
        wheel.set_wheel_width(35.0);
        Ok(())
    }
}

fn set_wheels(movement: &Checked<ChaosWheeledVehicleMovementComponent>) -> RustealResult<()> {
    let front = SubclassOf::<FrontWheel>::base().upcast_to::<ChaosVehicleWheel>();
    let rear = SubclassOf::<RearWheel>::base().upcast_to::<ChaosVehicleWheel>();

    let setups = movement.wheel_setups();
    setups.clear()?;

    for (class, bone) in [
        (front, "Phys_Wheel_FL"),
        (front, "Phys_Wheel_FR"),
        (rear, "Phys_Wheel_BL"),
        (rear, "Phys_Wheel_BR"),
    ] {
        let setup = OwnedStruct::<FChaosWheelSetup>::new();
        setup.as_ref().set_wheel_class(class);
        setup.as_ref().set_bone_name(FName::new(bone).handle());
        setups.push(&setup)?;
    }

    Ok(())
}
```

The bone names are the wheel bones of the car's skeletal mesh. `set_wheels` runs in the car's class defaults, next to the engine setup.

## Use case: two cameras

**The goal:** a chase camera behind the car that the player can turn with the mouse and that recenters on its own, and a hood camera, toggled with a key.

```rust
#[uclass(parent = WheeledVehiclePawn)]
pub struct Car {
    #[component(attach = "mesh", name = "Front Spring Arm")]
    front_spring_arm: SpringArmComponent,

    #[component(attach = "front_spring_arm", name = "Front Camera")]
    front_camera: CameraComponent,

    #[component(attach = "mesh", name = "Back Spring Arm")]
    back_spring_arm: SpringArmComponent,

    #[component(attach = "back_spring_arm", name = "Back Camera")]
    back_camera: CameraComponent,

    front_camera_active: bool,
}
```

The front camera starts inactive (`set_auto_activate(false)` in class defaults). Toggling activates one and deactivates the other:

```rust
#[ufunction(BlueprintCallable)]
pub(crate) fn do_toggle_camera(&mut self) {
    let front_active = !self.front_camera_active();
    self.set_front_camera_active(front_active);

    if let Ok(front) = self.front_camera().and_then(|camera| camera.checked()) {
        front.set_active(front_active, None);
    }

    if let Ok(back) = self.back_camera().and_then(|camera| camera.checked()) {
        back.set_active(!front_active, None);
    }
}
```

Looking around turns the back spring arm (`k2_add_local_rotation`), and every frame it turns back toward the front:

```rust
#[ufunction(Override)]
fn receive_tick(&mut self, delta_seconds: f32) {
    let Ok(arm) = self.back_spring_arm().and_then(|arm| arm.checked()) else {
        return;
    };

    let yaw = arm.get_relative_rotation().to_rotator().yaw;
    let yaw = model::f_interp_to(yaw, 0.0, f64::from(delta_seconds), 1.0);

    arm.k2_set_relative_rotation(&FRotator::from_rotator(Rotator::new(0.0, yaw, 0.0)), false, false);
}
```

`f_interp_to` is the engine's `FMath::FInterpTo` written in plain Rust in the template's `model.rs`, where it is tested.

## Use case: putting a flipped car back on its wheels

**The goal:** a car upside down for two checks in a row resets itself; a key resets it at any time.

A looping timer checks the car every few seconds. Its handle is kept in a Rust-private field, so the timer is cleared when the car leaves play:

```rust
use std::time::Duration;

use rusteal_runtime::runtime::timer::{self, TimerHandle};

#[uclass(parent = WheeledVehiclePawn)]
pub struct Car {
    #[uproperty(EditAnywhere, category = "Flip Check", default = 3.0)]
    flip_check_time: f32,

    #[uproperty(EditAnywhere, category = "Flip Check", default = -0.2)]
    flip_check_min_dot: f32,

    previous_flip_check: bool,
    flip_check_timer: Option<TimerHandle>,
}
```

```rust
#[ufunction(Override)]
fn receive_begin_play(&mut self) {
    match timer::set_timer(
        &self.as_ref(),
        Duration::from_secs_f32(self.flip_check_time()),
        true,
        method(self, Self::flipped_check),
    ) {
        Ok(timer) => self.set_flip_check_timer(Some(timer)),
        Err(e) => log::warn!("[Vehicle] failed to set the flip check timer: {e}"),
    }
}

#[ufunction(Override)]
fn receive_end_play(&mut self, _end_play_reason: u8) {
    if let Some(timer) = self.flip_check_timer() {
        timer.clear();
    }
}

#[ufunction]
fn flipped_check(&mut self) {
    let Ok(mesh) = self.as_ref().checked().and_then(|me| me.get_mesh().checked()) else {
        return;
    };

    let up_dot = DVec3::Z.dot(mesh.get_up_vector().to_dvec3()) as f32;
    let check = model::flip_check(self.previous_flip_check(), up_dot, self.flip_check_min_dot());

    if check.reset {
        self.do_reset_vehicle();
    }

    self.set_previous_flip_check(check.flipped);
}
```

The reset lifts the car, keeps its yaw, teleports it and clears its velocity:

```rust
fn reset(&self) -> RustealResult<()> {
    let me = self.as_ref().checked()?;
    let location = me.k2_get_actor_location().to_dvec3() + DVec3::Z * 50.0;
    let yaw = me.k2_get_actor_rotation().to_rotator().yaw;

    let transform = KismetMathLibrary::make_transform(
        &FVector::from_dvec3(location),
        &FRotator::from_rotator(Rotator::new(0.0, yaw, 0.0)),
        &FVector::from_dvec3(DVec3::ONE),
    );

    me.k2_set_actor_transform(&transform, false, true);

    let mesh = me.get_mesh().checked()?;
    let zero = FVector::from_dvec3(DVec3::ZERO);
    mesh.set_physics_angular_velocity_in_degrees(&zero, None, None);
    mesh.set_physics_linear_velocity(&zero, None, None);

    Ok(())
}
```

The last argument of `k2_set_actor_transform` is the teleport flag: the physics body moves with the actor instead of being swept there.

## Use case: speed and gear on screen

**The goal:** a HUD showing the speed, in km/h or mph, and the current gear.

The widget class declares what it can be told; its Widget Blueprint draws it:

```rust
#[uclass(parent = UserWidget)]
pub struct VehicleAdvUI {
    #[uproperty(EditAnywhere, BlueprintReadOnly, name = "bIsMPH", category = "Vehicle", default = false)]
    b_is_mph: bool,
}

#[uclass_impl]
impl VehicleAdvUI {
    #[ufunction(BlueprintImplementableEvent)]
    fn on_speed_update(&self, new_speed: f32) {}

    #[ufunction(BlueprintImplementableEvent)]
    fn on_gear_update(&self, new_gear: i32) {}
}

impl VehicleAdvUI {
    pub fn update_speed(&self, forward_speed: f32) {
        self.on_speed_update(model::display_speed(forward_speed, self.b_is_mph()));
    }

    pub fn update_gear(&self, gear: i32) {
        self.on_gear_update(gear);
    }
}
```

The player controller creates it and feeds it every frame:

```rust
#[ufunction(Override)]
fn receive_tick(&mut self, _delta_seconds: f32) {
    let (Ok(car), Ok(ui)) = (Car::from_obj(self.vehicle_pawn()), VehicleAdvUI::from_obj(self.vehicle_ui())) else {
        return;
    };

    if let Ok(movement) = car.movement() {
        ui.update_speed(movement.get_forward_speed());
        ui.update_gear(movement.get_current_gear());
    }
}
```

`get_forward_speed` is in centimeters per second; `display_speed` multiplies it by 0.036 for km/h or 0.022 for mph, as the C++ template does.

> **Screenshot:** the vehicle template in play, with the gear and the speed in the corner of the screen. *(images/recipes/vehicle_hud.png)*

## Use case: a time trial

**The goal:** a countdown, gates the car must pass in order, laps, and the best lap time.

The time trial variant uses four classes:

| Class | Parent | What it does |
|---|---|---|
| `TimeTrialTrackGate` | `Actor` | A box on the track; it knows the next gate, and whether it is the finish line. |
| `TimeTrialGameMode` | `GameModeBase` | Finds the finish line by its tag when the race begins. |
| `TimeTrialPlayerController` | `PlayerController` | Holds the gate the player must pass next and counts the laps. |
| `TimeTrialUI`, `TimeTrialStartUI` | `UserWidget` | The countdown, the laps and the best time. |

A gate checks that the car passing it is going to the right gate:

```rust
#[ufunction(Override)]
fn receive_actor_begin_overlap(&mut self, other_actor: UObjectRef<Actor>) {
    let Ok(other) = other_actor.checked() else {
        return;
    };

    let Ok(mut controller) = TimeTrialPlayerController::from_obj(other.get_instigator_controller()) else {
        return;
    };

    let me: UObjectRef<TimeTrialTrackGate> = self.as_ref().cast().unwrap_or_default();

    if controller.target_gate() != me {
        return;
    }

    controller.set_target_gate(self.next_marker());

    if self.b_is_finish_line() {
        controller.increment_lap_count();
    }
}
```

The countdown is a widget animation in the Blueprint; when it ends, the Blueprint calls **Finish Countdown**, and two delegates carry the start to the controller:

```rust
#[uclass(parent = UserWidget)]
pub struct TimeTrialStartUI {}

#[uclass_impl]
impl TimeTrialStartUI {
    #[udelegate]
    pub fn on_countdown_finished(&self) {}

    #[ufunction(BlueprintImplementableEvent, name = "BP_StartCountdown")]
    fn bp_start_countdown(&self) {}

    #[ufunction(BlueprintCallable)]
    fn finish_countdown(&mut self) {
        self.on_countdown_finished();
    }
}
```

The lap UI creates the countdown when it is constructed, listens to it, and passes the start on with its own delegate, which the player controller listens to:

```rust
#[ufunction(Override)]
fn construct(&mut self) {
    if let Err(e) = self.start_countdown() {
        log::error!("[TimeTrial] could not start the countdown: {e}");
    }
}

fn start_countdown(&mut self) -> RustealResult<()> {
    let owning_player = self.as_ref().checked()?.get_owning_player();
    let start_ui = create_widget_of_class(&owning_player, self.start_ui_class())?;
    start_ui.upcast_to::<UserWidget>().checked()?.add_to_viewport(Some(0));

    let countdown = TimeTrialStartUI::from_obj(start_ui)?;
    countdown.on_countdown_finished_add(method(self, Self::start_race))?.detach();
    countdown.start_countdown();

    Ok(())
}
```

```rust
TimeTrialUI::from_obj(widget)?
    .on_race_start_add(method(self, Self::start_race))?
    .detach();
```

Until the race starts, the controller keeps the car's input off (`pawn.disable_input(controller)` when it possesses it, `enable_input` in `start_race`). The lap times are plain Rust in `time_trial/model.rs`, with tests:

```rust
impl LapTimes {
    pub fn start_lap(&mut self, lap: i32, lap_start_time: f32) {
        let lap_time = lap_start_time - self.last_lap_time;

        self.best_lap_time = if lap <= 1 {
            -1.0
        } else if self.best_lap_time < 0.0 || lap_time < self.best_lap_time {
            lap_time
        } else {
            self.best_lap_time
        };

        self.current_lap = lap;
        self.lap_start_time = lap_start_time;
        self.last_lap_time = lap_start_time;
    }
}
```

### In the editor

The gates are placed in the level, each one's **Next Marker** set to the next gate in the **Details** panel, and the finish line has **Is Finish Line** checked and the tag the game mode's **Finish Tag** names.

> **Screenshot:** a track gate selected in the level, with Next Marker and Is Finish Line in the Details panel. *(images/recipes/track_gate.png)*
