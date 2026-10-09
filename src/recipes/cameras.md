# Cameras

## Use case: a third person camera on a spring arm

**The goal:** a camera behind the character that orbits with the mouse and pulls in when a wall is in the way.

```rust
use bindings::engine::{CameraComponent, CameraComponentExt, SpringArmComponent, SpringArmComponentExt};

#[uclass(parent = Character)]
pub struct Hero {
    #[component(attach = "root_component")]
    camera_boom: SpringArmComponent,

    #[component(attach = "camera_boom", socket = "SpringEndpoint")]
    follow_camera: CameraComponent,
}

#[uclass_impl]
impl Hero {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        me.set_use_controller_rotation_pitch(false);
        me.set_use_controller_rotation_yaw(false);
        me.set_use_controller_rotation_roll(false);

        let boom = self.camera_boom()?.checked()?;
        boom.set_target_arm_length(400.0);
        boom.set_use_pawn_control_rotation(true);

        self.follow_camera()?.checked()?.set_use_pawn_control_rotation(false);

        Ok(())
    }
}
```

The spring arm follows the controller's rotation, which the look input changes; the character does not turn with it, and the movement component turns it toward where it walks (`set_orient_rotation_to_movement(true)`).

A smoother camera adds lag: `boom.set_enable_camera_lag(true)` and `boom.set_camera_lag_speed(10.0)`.

## Use case: a first person camera

**The goal:** a camera at the head, with arms only the player sees and a body only others see.

```rust
use bindings::engine::{
    CameraComponent, CameraComponentExt, EFirstPersonPrimitiveType, PrimitiveComponentExt,
    SceneComponentExt, SkeletalMeshComponent,
};

#[uclass(parent = Character)]
pub struct FirstPersonHero {
    #[component(attach = "mesh", name = "First Person Mesh")]
    first_person_mesh: SkeletalMeshComponent,

    #[component(attach = "first_person_mesh", socket = "head", name = "First Person Camera")]
    first_person_camera: CameraComponent,
}

#[uclass_impl]
impl FirstPersonHero {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let arms = self.first_person_mesh()?.checked()?;
        arms.set_only_owner_see(true);
        arms.set_first_person_primitive_type(EFirstPersonPrimitiveType::FirstPerson);
        arms.set_collision_profile_name(FName::new("NoCollision").handle(), Some(true));

        let camera = self.first_person_camera()?.checked()?;
        camera.set_use_pawn_control_rotation(true);
        camera.set_enable_first_person_field_of_view(true);
        camera.set_first_person_field_of_view(70.0);

        let body = self.as_ref().checked()?.get_mesh().checked()?;
        body.set_owner_no_see(true);
        body.set_first_person_primitive_type(EFirstPersonPrimitiveType::WorldSpaceRepresentation);

        Ok(())
    }
}
```

The Blueprint child assigns the arms' and the body's skeletal meshes and Animation Blueprints.

## Use case: limiting how far the player looks up and down

**The goal:** a camera manager that clamps the pitch.

```rust
use bindings::engine::{PlayerCameraManager, PlayerCameraManagerExt};

#[uclass(parent = PlayerCameraManager)]
pub struct MyCameraManager {}

#[uclass_impl]
impl MyCameraManager {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let me = self.as_ref().checked()?;
        me.set_view_pitch_min(-70.0);
        me.set_view_pitch_max(80.0);
        Ok(())
    }
}
```

The player controller uses it, set in its own class defaults:

```rust
self.as_ref().checked()?.set_player_camera_manager_class(
    SubclassOf::<MyCameraManager>::base().upcast_to(),
);
```

## Use case: a side scroller camera computed every frame

**The goal:** a camera that follows the character along X, stays within the level's bounds, and moves up and down smoothly.

A camera manager decides the camera's location, rotation and field of view every frame in `BlueprintUpdateCamera`:

```rust
#[uclass(parent = PlayerCameraManager)]
pub struct SideCameraManager {
    #[uproperty(EditAnywhere, default = 400.0)]
    current_zoom: f32,

    #[uproperty(EditAnywhere, default = -1000.0)]
    camera_x_min_bounds: f32,

    #[uproperty(EditAnywhere, default = 1000.0)]
    camera_x_max_bounds: f32,
}

#[uclass_impl]
impl SideCameraManager {
    #[ufunction(Override)]
    fn blueprint_update_camera(
        &mut self,
        camera_target: UObjectRef<Actor>,
        new_camera_location: UStructRef<FVector>,
        new_camera_rotation: UStructRef<FRotator>,
        new_camera_fov: OutRef<f32>,
    ) -> bool {
        let Ok(target) = camera_target.checked() else {
            return false;
        };

        let location = target.k2_get_actor_location().to_dvec3();
        let x = location.x.clamp(f64::from(self.camera_x_min_bounds()), f64::from(self.camera_x_max_bounds()));

        new_camera_location.set_x(x);
        new_camera_location.set_y(location.y + f64::from(self.current_zoom()));
        new_camera_location.set_z(location.z);

        new_camera_rotation.set_pitch(0.0);
        new_camera_rotation.set_yaw(-90.0);
        new_camera_rotation.set_roll(0.0);

        new_camera_fov.set(65.0);

        true
    }
}
```

The event writes the camera into the structs it receives (`UStructRef`, written in place) and the field of view into an `OutRef`, and returns `true` to use them. The side-scrolling template adds smoothing with the engine's interpolation formulas, written as plain Rust in its `model.rs` and tested there.
