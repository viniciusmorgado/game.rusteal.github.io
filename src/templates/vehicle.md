# vehicle

The engine's Vehicle template on Chaos Vehicles: a sports car and an offroad car, with their wheels, the game mode, the player controller and the speed and gear HUD in Rust. It plays with keyboard, gamepad, steering wheel or touch.

```bash
rusteal new MyGame --template vehicle
```

From `TP_VehicleAdv`, the engine template it is made of.

## Playing it

Open the project and press **Play**: W / right trigger throttle, S / left trigger brake and reverse, A D / left stick steer, Space / left shoulder handbrake, mouse / right stick look around, Tab toggles the camera, Backspace resets the car.

> **Screenshot:** the vehicle template's sports car on the track. *(images/templates/vehicle.png)*

## Its classes

The Rust classes are in `Rust/<crate>/src/`; their Blueprint children, under the engine template's names, are in `Content/`.

| Class | Parent | File |
|---|---|---|
| `VehicleAdvGameMode` | `GameModeBase` | `src/game_mode.rs` |
| `VehicleAdvOffroadCar` | `VehicleAdvPawn` | `src/offroad_car.rs` |
| `VehicleAdvOffroadWheelFront` | `VehicleAdvWheelFront` | `src/offroad_car.rs` |
| `VehicleAdvOffroadWheelRear` | `VehicleAdvWheelRear` | `src/offroad_car.rs` |
| `VehicleAdvPawn` | `WheeledVehiclePawn` | `src/pawn.rs` |
| `VehicleAdvPlayerController` | `PlayerController` | `src/player_controller.rs` |
| `VehicleAdvSportsCar` | `VehicleAdvPawn` | `src/sports_car.rs` |
| `VehicleAdvSportsWheelFront` | `VehicleAdvWheelFront` | `src/sports_car.rs` |
| `VehicleAdvSportsWheelRear` | `VehicleAdvWheelRear` | `src/sports_car.rs` |
| `VehicleAdvUI` | `UserWidget` | `src/ui.rs` |
| `VehicleAdvWheelFront` | `ChaosVehicleWheel` | `src/wheels.rs` |
| `VehicleAdvWheelRear` | `ChaosVehicleWheel` | `src/wheels.rs` |

## Variants

Each variant is a project of its own: the template plus the variant, opening on the variant's level.

### offroad

The offroad car on an offroad level, its game mode ready for local split screen players.

```bash
rusteal new MyGame --template vehicle --variant offroad
```

It opens `/Game/Variant_OffRoad/Maps/Lvl_Offroad`. To play: W / right trigger throttle, S / left trigger brake and reverse, A D / left stick steer, Space / left shoulder handbrake, mouse / right stick look around, Tab toggles the camera, Backspace resets the car.

> **Screenshot:** the vehicle template's offroad variant in play. *(images/templates/vehicle-offroad.png)*

Its own classes, next to the template's, in a module named after it:

| Class | Parent | File |
|---|---|---|
| `OffroadGameMode` | `GameModeBase` | `src/offroad/game_mode.rs` |

### time-trial

A race track with a start countdown, gates to pass in order, laps and the best lap time.

```bash
rusteal new MyGame --template vehicle --variant time-trial
```

It opens `/Game/Variant_TimeTrial/Maps/Lvl_Timetrial`. To play: W / right trigger throttle, S / left trigger brake and reverse, A D / left stick steer, Space / left shoulder handbrake, mouse / right stick look around, Tab toggles the camera, Backspace resets the car.

> **Screenshot:** the vehicle template's time-trial variant in play. *(images/templates/vehicle-time-trial.png)*

Its own classes, next to the template's, in a module named after it:

| Class | Parent | File |
|---|---|---|
| `TimeTrialGameMode` | `GameModeBase` | `src/time_trial/game_mode.rs` |
| `TimeTrialPlayerController` | `PlayerController` | `src/time_trial/player_controller.rs` |
| `TimeTrialTrackGate` | `Actor` | `src/time_trial/track_gate.rs` |
| `TimeTrialUI` | `UserWidget` | `src/time_trial/ui.rs` |
| `TimeTrialStartUI` | `UserWidget` | `src/time_trial/ui.rs` |

Recipes taken from it: [Vehicles](../recipes/vehicles.md).
