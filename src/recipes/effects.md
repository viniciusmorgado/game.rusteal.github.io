# Effects and sound

Two ways to play effects, and the templates use both:

- **From the Blueprint child**, through a `BlueprintImplementableEvent` the Rust class calls (`BP_OnHit`, `BP_OnDeath`): the artist picks and tunes the effect in the graph, and Rust says when. This is the default in the templates. See [Functions](../classes/functions.md).
- **From Rust**, with the engine's functions, when the effect is part of the rule: a cursor marker where the player clicked, a sound at a computed location.

## Use case: a Niagara effect at a point

**The goal:** a marker effect where the player clicked.

The project needs the `niagara` feature (see [Binding more of the engine](../cli/rusteal_toml.md)).

```rust
use bindings::niagara::{ENCPoolMethod, NiagaraFunctionLibrary, NiagaraSystem};

#[uproperty(EditAnywhere, name = "FXCursor", category = "Input")]
fx_cursor: UObjectRef<NiagaraSystem>,
```

```rust
NiagaraFunctionLibrary::spawn_system_at_location(
    self.as_ref().upcast_to(),
    self.fx_cursor(),
    &FVector::from_dvec3(location),
    &FRotator::from_rotator(Rotator::ZERO),
    &FVector::from_dvec3(DVec3::ONE),
    Some(true),
    Some(true),
    Some(ENCPoolMethod::None),
    Some(true),
);
```

The arguments after the scale are auto destroy, auto activate, the pooling method and whether to pre-cull. `spawn_system_attached` attaches the effect to a component instead, to follow it.

## Use case: a sound at a location

```rust
use bindings::engine::{GameplayStatics, SoundBase};

#[uproperty(EditAnywhere)]
impact_sound: UObjectRef<SoundBase>,
```

```rust
GameplayStatics::play_sound_at_location(
    self.as_ref().upcast_to(),
    self.impact_sound(),
    &FVector::from_dvec3(location),
    &FRotator::from_rotator(Rotator::ZERO),
    None,
    None,
    None,
    None,
    None,
    None,
    None,
);
```

The `None`s keep the defaults for volume, pitch, start time, attenuation, concurrency, owner and initial parameters. `play_sound_2d` plays a sound without a location (UI, music), and `spawn_sound_at_location` returns the audio component, to stop or fade it later.

## Use case: a light on the character

**The goal:** a flashlight attached to the first-person camera.

```rust
use bindings::engine::{ELightUnits, LightComponentExt, LocalLightComponentExt, SpotLightComponent, SpotLightComponentExt};

#[uclass(parent = FirstPersonCharacter)]
pub struct HorrorCharacter {
    #[component(attach = "first_person_camera_component", name = "SpotLight")]
    spot_light: SpotLightComponent,
}

#[uclass_impl]
impl HorrorCharacter {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let light = self.spot_light()?.checked()?;
        light.set_intensity(1500.0);
        light.set_intensity_units(ELightUnits::Lumens);
        light.set_attenuation_radius(2000.0);
        Ok(())
    }
}
```

Turning it on and off is `light.set_visibility(on, None)`, bound to an input action.
