# Physics

## Use case: a prop that reacts to hits

**The goal:** a box that simulates physics, is pushed where it is hit, and breaks after enough damage.

```rust
use bindings::engine::{ECollisionChannel, KismetSystemLibrary, PrimitiveComponentExt, StaticMeshComponent};

#[uclass(parent = Actor)]
pub struct Crate {
    #[component(root)]
    mesh: StaticMeshComponent,

    #[uproperty(EditAnywhere, name = "CurrentHP", default = 30.0)]
    current_hp: f32,
}

#[uclass_impl]
impl Crate {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let mesh = self.mesh()?.checked()?;
        mesh.set_collision_profile_name(FName::new("BlockAllDynamic").handle(), Some(true));
        mesh.set_simulate_physics(true);
        mesh.set_can_ever_affect_navigation(false);
        Ok(())
    }
}

impl Crate {
    pub fn hit(&mut self, damage: f32, location: DVec3, impulse: DVec3) {
        self.set_current_hp(self.current_hp() - damage);

        if let Ok(mesh) = self.mesh().and_then(|m| m.checked()) {
            let scaled = impulse * f64::from(mesh.get_mass());
            mesh.add_impulse_at_location(&FVector::from_dvec3(scaled), &FVector::from_dvec3(location), None);

            if self.current_hp() <= 0.0 {
                mesh.set_collision_object_type(ECollisionChannel::ECC_Visibility);
            }
        }
    }
}
```

`add_impulse_at_location` pushes and spins the body from the point that was hit. Multiplying by the mass makes the push the same for a light and a heavy prop; `add_impulse(impulse, bone, vel_change = Some(true))` does that for you.

## Use case: knockback on a character

A character's capsule does not simulate physics; its movement component takes the impulse:

```rust
if let Ok(movement) = me.get_character_movement().checked() {
    movement.add_impulse(&FVector::from_dvec3(knockback), Some(true));
}
```

Or replace its velocity at once with `launch_character`.

## Use case: a hit reaction with partial ragdoll

```rust
let mesh = me.get_mesh().checked()?;
mesh.set_physics_blend_weight(0.5);
mesh.set_all_bodies_below_simulate_physics(FName::new("pelvis").handle(), true, Some(false));
```

The combat template blends the ragdoll in when hit and back out on landing (`set_physics_blend_weight(0.0)` in `on_landed`).

## Use case: a physics constraint

**The goal:** a training dummy that wobbles on its base.

```rust
use bindings::engine::{PhysicsConstraintComponent, PhysicsConstraintComponentExt};

#[uclass(parent = Actor)]
pub struct Dummy {
    #[component(root)]
    root: SceneComponent,

    #[component(attach = "root", name = "Base Plate")]
    base_plate: StaticMeshComponent,

    #[component(attach = "root")]
    dummy: StaticMeshComponent,

    #[component(attach = "root", name = "Physics Constraint")]
    constraint: PhysicsConstraintComponent,
}

#[uclass_impl]
impl Dummy {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        self.dummy()?.checked()?.set_simulate_physics(true);
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        if let (Ok(constraint), Ok(base_plate), Ok(dummy)) =
            (self.constraint().and_then(|c| c.checked()), self.base_plate(), self.dummy())
        {
            constraint.set_constrained_components(
                base_plate.upcast_to(),
                FName::NONE.handle(),
                dummy.upcast_to(),
                FName::NONE.handle(),
            );
        }
    }
}
```

The constraint's limits (swing, twist) are set on the component in the Blueprint child's **Details** panel.

A chain is the same thing repeated: the `interactables` feature's `BP_DangleBlocks` has four cubes and four constraints, the first holding the top cube to the world (its **Component Name 1** names no component) and each other one a cube to the one above, with where each piece sits and how far each joint swings. Its Rust parent, `DangleBlocks`, has no code: the Blueprint is all data.

## Collision presets and channels

| Call | What it does |
|---|---|
| `set_collision_profile_name(FName::new("BlockAllDynamic").handle(), Some(true))` | Applies a preset from **Project Settings > Collision**. |
| `set_collision_enabled(ECollisionEnabled::QueryOnly)` | Overlaps and traces, no physics. |
| `set_collision_object_type(ECollisionChannel::ECC_WorldDynamic)` | What the component is, for others' responses. |
| `set_collision_response_to_all_channels(ECollisionResponse::ECR_Ignore)` | Ignores everything... |
| `set_collision_response_to_channel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap)` | ...then overlaps pawns. |

A component meant to fire overlap events also needs **Generate Overlap Events** on, which most presets have.
