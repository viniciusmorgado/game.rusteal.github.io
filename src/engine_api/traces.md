# Traces and overlaps

**The goal:** ask the world what is in a line, a sphere or an area: what the player aims at, what a sword swing hits, what is caught in an explosion.

All of them are functions of `KismetSystemLibrary`, the Blueprint nodes **Line Trace By Channel**, **Sphere Trace For Objects**, **Sphere Overlap Actors** and the others.

## Line trace: what the player aims at

```rust
use bindings::engine::{EDrawDebugTrace, ETraceTypeQuery, FHitResultExt, KismetSystemLibrary};

fn aim_point(&self, start: DVec3, direction: DVec3) -> DVec3 {
    let end = start + direction * 10_000.0;
    let me = self.as_ref().upcast_to::<Actor>();

    let (hit, out_hit) = KismetSystemLibrary::line_trace_single(
        me.upcast_to(),
        &FVector::from_dvec3(start),
        &FVector::from_dvec3(end),
        ETraceTypeQuery::TraceTypeQuery1,
        false,
        &[me],
        EDrawDebugTrace::None,
        true,
        &OwnedStruct::new(),
        &OwnedStruct::new(),
        None,
    );

    if hit {
        out_hit.as_ref().get_impact_point().to_dvec3()
    } else {
        end
    }
}
```

| Argument | What it is |
|---|---|
| world context | any object in the world |
| start, end | the line |
| trace channel | `TraceTypeQuery1` is **Visibility**, `TraceTypeQuery2` is **Camera** |
| trace complex | `true` traces against the mesh's triangles instead of its collision |
| actors to ignore | `&[]` for none |
| draw debug type | `EDrawDebugTrace::ForDuration` draws the trace in the viewport while you work on it |
| ignore self | `true` ignores the actor the context belongs to |
| trace color, hit color | `&OwnedStruct::new()` for the defaults |
| draw time | `None` for the default |

It returns whether something was hit and the `FHitResult`. Read the hit with `as_ref()`: `get_impact_point()`, `get_impact_normal()`, `get_location()`, `get_component()`, `get_blocking_hit()`.

```rust
use bindings::engine::ActorComponentExt;

let hit_actor = out_hit.as_ref().get_component().checked()?.get_owner();
```

## Sphere trace for objects: a sword swing

**The goal:** everything a swing touches between two points, pawns and movable props, not the floor.

```rust
use bindings::engine::{ECollisionChannel, EDrawDebugTrace, KismetSystemLibrary};

fn sweep(&self, start: DVec3, end: DVec3, radius: f32) -> Vec<OwnedStruct<FHitResult>> {
    let object_types: Vec<_> = [ECollisionChannel::ECC_Pawn, ECollisionChannel::ECC_WorldDynamic]
        .into_iter()
        .filter_map(object_type_query)
        .collect();

    let (_, hits) = KismetSystemLibrary::sphere_trace_multi_for_objects(
        self.as_ref().upcast_to(),
        &FVector::from_dvec3(start),
        &FVector::from_dvec3(end),
        radius,
        &object_types,
        false,
        &[],
        EDrawDebugTrace::None,
        true,
        &OwnedStruct::new(),
        &OwnedStruct::new(),
        None,
    );

    hits
}
```

"For objects" traces filter by object type, the **Object Type** of a collision preset, instead of a trace channel. `object_type_query` (from the prelude) turns a collision channel into the object type the function takes.

The single, multi, line, sphere, box and capsule variants (`line_trace_multi`, `sphere_trace_single`, `box_trace_single_for_objects`, `capsule_trace_single`...) follow the same pattern.

## Overlap: an explosion

**The goal:** every actor within a radius.

```rust
let object_types: Vec<_> = [ECollisionChannel::ECC_Pawn, ECollisionChannel::ECC_PhysicsBody]
    .into_iter()
    .filter_map(object_type_query)
    .collect();

let ignored = [self.as_ref().upcast_to::<Actor>()];

let (_, actors) = KismetSystemLibrary::sphere_overlap_actors(
    self.as_ref().upcast_to(),
    &FVector::from_dvec3(center),
    500.0,
    &object_types,
    SubclassOf::<Actor>::base().upcast_to(),
    &ignored,
);

for actor in actors {
}
```

`sphere_overlap_components` returns the components instead, useful to push the physics bodies hit.

## Overlaps of a component

A component with overlap events lists what overlaps it now:

```rust
let sphere = self.collision_sphere()?.checked()?;

for actor in sphere.get_overlapping_actors(Some(SubclassOf::<Enemy>::base().upcast_to())) {
}
```

To react when something starts overlapping, bind the component's delegate. See [Delegates](delegates.md).

## What is under the mouse

```rust
let controller = self.as_ref().checked()?;
let (hit, out_hit) = controller.get_hit_result_under_cursor_by_channel(ETraceTypeQuery::TraceTypeQuery1, true);

if hit {
    let point = out_hit.as_ref().get_location().to_dvec3();
}
```

`get_hit_result_under_finger_by_channel(ETouchIndex::Touch1, ...)` is the same for touch.
