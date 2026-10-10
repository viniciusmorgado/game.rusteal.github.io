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

## Use case: a MetaSound parameter

**The goal:** a tuning fork whose MetaSound plays a pattern chosen by a seed set on each placed fork.

```rust
#[ufunction(Override)]
fn receive_tick(&mut self, _delta_seconds: f32) {
    if let Ok(audio) = self.audio().and_then(|a| a.checked()) {
        audio.set_int_parameter(FName::new("Seed").handle(), self.random_seed());
    }
}
```

The audio component's **Sound** is the MetaSound source; any of its inputs is set by name with `set_int_parameter`, `set_float_parameter` or `set_bool_parameter`.

## Use case: an effect a moment after a trigger

**The goal:** a confetti burst one second after a trigger first fires, and never again.

```rust
#[ufunction]
fn trigger_confetti(&mut self) {
    if self.fired() {
        return;
    }

    self.set_fired(true);
    KismetSystemLibrary::k2_set_timer(self.as_ref().upcast_to(), "Burst", 1.0, false, None, None, None);
}

#[ufunction]
fn burst(&mut self) {
    if let Ok(niagara) = self.niagara().and_then(|n| n.checked()) {
        niagara.activate(Some(true));
    }
}
```

The Niagara component is not auto-activated; `activate(Some(true))` restarts it from the beginning.

## Use case: playing a level sequence

**The goal:** walking into an area plays a cinematic placed in the level.

```rust
#[ufunction(Override)]
fn receive_actor_begin_overlap(&mut self, _other_actor: UObjectRef<Actor>) {
    if let Ok(sequence) = self.seq_ref().checked()
        && let Ok(player) = sequence.get_sequence_player().checked()
    {
        player.play();
    }
}
```

`seq_ref` is a `UObjectRef<LevelSequenceActor>` set on the placed trigger. The project binds the sequencer modules with the `level-sequence` and `movie` features (see [Binding more of the engine](../cli/rusteal_toml.md)).
