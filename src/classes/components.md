# Components

**The goal:** an actor made of components, a mesh, a collision shape, a camera on a spring arm, created with every instance, as a C++ constructor's `CreateDefaultSubobject` creates them.

```rust
use bindings::engine::{
    BoxComponent, CameraComponent, ProjectileMovementComponent, SceneComponent,
    SpringArmComponent, StaticMeshComponent,
};
use rusteal_runtime::{uclass, uclass_impl};

#[uclass(parent = Actor)]
pub struct Door {
    #[component(root)]
    root: SceneComponent,

    #[component(attach = "root")]
    frame: StaticMeshComponent,

    #[component(attach = "frame", socket = "Hinge")]
    panel: StaticMeshComponent,

    #[component(attach = "root", name = "Trigger Box")]
    trigger: BoxComponent,
}

#[uclass_impl]
impl Door {}
```

| Argument | What it does |
|---|---|
| `root` | The component is the actor's root component. |
| `attach = "<field>"` | The component it is attached to: one declared before it in the class, or an inherited one by its field-style name. |
| `socket = "<name>"` | A socket on the parent component (a bone, a spring arm's end). |
| `name = "..."` | The component's name in the editor, when it is not the field's. |

A component that is not a scene component (a movement component, a perception component, a StateTree component) has no transform: it takes neither `root` nor `attach`.

```rust
#[component(name = "Projectile Movement")]
projectile_movement: ProjectileMovementComponent,
```

## Attaching to inherited components

A `Character` already has a capsule (its root), a skeletal mesh and a movement component. Attach to them by their field-style names:

```rust
#[uclass(parent = Character)]
pub struct Hero {
    #[component(attach = "root_component")]
    camera_boom: SpringArmComponent,

    #[component(attach = "camera_boom", socket = "SpringEndpoint")]
    follow_camera: CameraComponent,
}
```

| Inherited component | Field-style name |
|---|---|
| The root component of any actor | `root_component` |
| A character's skeletal mesh | `mesh` |
| A component of a Rust parent class | its field's name (`first_person_camera_component`) |

## Reaching a component

Each component gets a getter that returns a reference to it:

```rust
let boom = self.camera_boom()?.checked()?;
boom.set_target_arm_length(400.0);
```

The getter returns a `RustealResult`, because the component may not exist on the object at that moment (an object being destroyed). Inherited engine components come from the engine getters: `me.get_mesh()`, `me.get_capsule_component()`, `me.get_character_movement()`.

## Names and Blueprints

A component's name is its field's name, or `name`. A Blueprint child's changes to an inherited component, a mesh assigned to it, a collision preset, are saved under that name. Keep it stable once Blueprints use the class: renaming the field without `name` loses those changes. When you port a C++ class, give each component the name the C++ `CreateDefaultSubobject` used, and the Blueprint children keep their data.

## Configuring components

The values a component starts with go in [Class defaults](class_defaults.md):

```rust
use bindings::engine::{BoxComponentExt, PrimitiveComponentExt};

#[class_defaults]
fn class_defaults(&mut self) -> RustealResult<()> {
    let trigger = self.trigger()?.checked()?;
    trigger.set_box_extent(&FVector::from_dvec3(DVec3::new(100.0, 100.0, 120.0)), Some(false));
    trigger.set_collision_profile_name(FName::new("OverlapAllDynamic").handle(), Some(true));
    Ok(())
}
```

## Adding a component at runtime

A component created during the game, for example on an actor the class does not own, is added through the engine:

```rust
let class = SubclassOf::<InventoryComponent>::base().upcast_to::<ActorComponent>();
let identity = FTransform::from_transform(Transform::IDENTITY);
let component = actor.checked()?.add_component_by_class(class, false, &identity, false);
```

See the [inventory recipe](../recipes/inventory.md).

## In the editor

The Blueprint child's **Components** panel lists the Rust components as inherited components: select one to change its mesh, material or collision in the **Details** panel.

> **Screenshot:** the Components panel of a Blueprint child listing the components of a Rust class. *(images/classes/components_panel.png)*
