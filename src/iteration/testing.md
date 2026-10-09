# Testing without the engine

**The goal:** test gameplay rules with `cargo test`, without opening Unreal.

Code that calls the engine needs the engine to run. Code that only computes does not: damage formulas, combo windows, stamina, inventories, wave schedules, movement math. Keep that code in plain Rust functions and types, apart from the classes, and test it like any Rust code.

The templates do this with a `model.rs` file next to each class: the tunable numbers and the rules live there, and the class calls them.

## Example

`src/character/model.rs`:

```rust
use glam::DVec3;

pub const MAX_WALK_SPEED: f32 = 500.0;
pub const CAMERA_BOOM_LENGTH: f32 = 400.0;

pub fn ground_axes(yaw_degrees: f64) -> (DVec3, DVec3) {
    let (sin, cos) = yaw_degrees.to_radians().sin_cos();
    (DVec3::new(cos, sin, 0.0), DVec3::new(-sin, cos, 0.0))
}

#[cfg(test)]
mod tests {
    use super::*;

    fn close(a: DVec3, b: DVec3) -> bool {
        (a - b).length() < 1e-9
    }

    #[test]
    fn ground_axes_follow_control_yaw() {
        let (forward, right) = ground_axes(90.0);
        assert!(close(forward, DVec3::Y));
        assert!(close(right, -DVec3::X));
    }
}
```

The class uses it:

```rust
let yaw = controller.checked()?.get_control_rotation().to_rotator().yaw;
let (forward, right) = model::ground_axes(yaw);
```

## Running the tests

From `Rust/`:

```bash
cargo test -p my-game
```

The tests compile your crate as a normal Rust program, without the engine. A test that calls an engine function cannot run this way: keep the engine calls in the classes and the rules in the model.

## What else works

Everything Rust offers for testing works here: property tests (`proptest`), benchmarks (`criterion`), snapshot tests. A rules crate of its own, with no Rusteal dependency at all, can be shared with a dedicated server, a balancing tool or a web page.
