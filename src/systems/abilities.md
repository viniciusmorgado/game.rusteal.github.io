# The Gameplay Ability System

**The goal:** health, damage and skills on the engine's Gameplay Ability System (GAS), as Lyra and most action games build them: attributes in an attribute set, changes to them in gameplay effects, actions in gameplay abilities, all three written in Rust.

The `gameplay-abilities` feature binds the system (see [Binding more of the engine](../cli/rusteal_toml.md)), and `rusteal build` makes the project's Rusteal plugin depend on the GameplayAbilities plugin. With the feature, `rusteal build --all` builds a base for attribute sets (since 0.10): `RustealAttributeSet`, in `bindings::rusteal`.

## Use case: an attribute set

```rust
use bindings::gameplay_abilities::{
    AbilitySystemComponent, FGameplayAttribute, FGameplayAttributeData, FGameplayAttributeDataExt,
    FGameplayAttributeExt, FGameplayEffectSpec, FGameplayModifierEvaluatedData,
    FGameplayModifierEvaluatedDataExt,
};
use bindings::rusteal::RustealAttributeSet;

fn attribute_data(value: f32) -> OwnedStruct<FGameplayAttributeData> {
    let data = OwnedStruct::<FGameplayAttributeData>::new();
    data.as_ref().set_base_value(value);
    data.as_ref().set_current_value(value);
    data
}

#[uclass(parent = RustealAttributeSet)]
pub struct HealthSet {
    #[uproperty(BlueprintReadOnly)]
    health: OwnedStruct<FGameplayAttributeData>,

    #[uproperty(BlueprintReadOnly)]
    max_health: OwnedStruct<FGameplayAttributeData>,
}

impl HealthSet {
    fn clamped(&self, attribute: &UStructRef<FGameplayAttribute>, value: f32) -> f32 {
        if attribute.get_attribute_name() == "Health" {
            value.clamp(0.0, self.max_health().as_ref().get_current_value())
        } else {
            value
        }
    }
}

#[uclass_impl]
impl HealthSet {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        self.set_health(&attribute_data(100.0));
        self.set_max_health(&attribute_data(100.0));
        Ok(())
    }

    #[ufunction(Override)]
    fn receive_pre_attribute_change(
        &mut self,
        attribute: UStructRef<FGameplayAttribute>,
        new_value: OutRef<f32>,
    ) {
        new_value.set(self.clamped(&attribute, new_value.get()));
    }

    #[ufunction(Override)]
    fn receive_pre_attribute_base_change(
        &self,
        attribute: UStructRef<FGameplayAttribute>,
        new_value: OutRef<f32>,
    ) {
        new_value.set(self.clamped(&attribute, new_value.get()));
    }

    #[ufunction(Override)]
    fn receive_post_gameplay_effect_execute(
        &mut self,
        _effect_spec: UStructRef<FGameplayEffectSpec>,
        evaluated_data: UStructRef<FGameplayModifierEvaluatedData>,
        target: UObjectRef<AbilitySystemComponent>,
    ) {
        if evaluated_data.get_magnitude() < 0.0 {
            show_hit(target);
        }
    }
}
```

An attribute is a property of type `OwnedStruct<FGameplayAttributeData>`: the system finds it by reflection, as it finds a C++ set's. Its initial values go in the class defaults.

The `UAttributeSet` hooks are C++ virtuals; `RustealAttributeSet` raises them as events:

| Virtual | Event (Rust method) |
|---|---|
| `PreAttributeChange` | `receive_pre_attribute_change(&mut self, attribute, new_value: OutRef<f32>)`: clamp the new current value |
| `PostAttributeChange` | `receive_post_attribute_change(&mut self, attribute, old_value: f32, new_value: f32)` |
| `PreAttributeBaseChange`, `PostAttributeBaseChange` | the same for the base value; `&self`, since the engine's are const |
| `PreGameplayEffectExecute` | `receive_pre_gameplay_effect_execute(&mut self, effect_spec, evaluated_data: UStructRef<FGameplayModifierEvaluatedData>, target) -> bool`: change the magnitude, or return `false` to refuse the change |
| `PostGameplayEffectExecute` | `receive_post_gameplay_effect_execute(&mut self, effect_spec, evaluated_data, target)` |

The engine passes the two effect hooks an `FGameplayEffectModCallbackData`, which Blueprints and Rust cannot take: their events take its three parts, the effect's spec, the evaluated modifier and the target's ability system component.

`RustealAttributeSet::attribute_of(set_class, name)` gives the `FGameplayAttribute` of a set's property, by its Unreal name (`Health` for `health`): the attribute a modifier changes and the one `get_gameplay_attribute_value` reads.

```rust
fn health_attribute() -> OwnedStruct<FGameplayAttribute> {
    RustealAttributeSet::attribute_of(
        SubclassOf::<HealthSet>::base().upcast_to::<AttributeSet>(),
        FName::new("Health").handle(),
    )
}
```

## Use case: a gameplay effect

```rust
use bindings::gameplay_abilities::{
    EGameplayEffectDurationType, EGameplayEffectMagnitudeCalculation, EGameplayModOp,
    FGameplayEffectModifierMagnitudeExt, FGameplayModifierInfo, FGameplayModifierInfoExt,
    FScalableFloatExt, GameplayEffect, GameplayEffectExt,
};

#[uclass(parent = GameplayEffect)]
pub struct FireDamage {}

#[uclass_impl]
impl FireDamage {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        let effect = self.as_ref().checked()?;
        effect.set_duration_policy(EGameplayEffectDurationType::Instant);

        let modifier = OwnedStruct::<FGameplayModifierInfo>::new();
        modifier.as_ref().set_attribute(&health_attribute());
        modifier.as_ref().set_modifier_op(EGameplayModOp::AddBase);

        let magnitude = modifier.as_ref().get_modifier_magnitude();
        magnitude
            .as_ref()
            .set_magnitude_calculation_type(EGameplayEffectMagnitudeCalculation::ScalableFloat);
        let amount = magnitude.as_ref().get_scalable_float_magnitude();
        amount.as_ref().set_value(-25.0);
        magnitude.as_ref().set_scalable_float_magnitude(&amount);
        modifier.as_ref().set_modifier_magnitude(&magnitude);

        effect.modifiers().push(&modifier)?;
        Ok(())
    }
}
```

A gameplay effect is data: a class whose defaults hold the duration and the modifiers, which a Rust class sets in its class defaults as a Blueprint child sets them in its details panel. The bindings reach the modifier's fields the editor shows, protected ones included.

## Use case: a gameplay ability

```rust
use bindings::gameplay_abilities::{GameplayAbility, GameplayAbilityExt};

#[uclass(parent = GameplayAbility)]
pub struct Dash {}

#[uclass_impl]
impl Dash {
    #[ufunction(Override, name = "K2_ActivateAbility")]
    fn activate_ability(&mut self) {
        if let Ok(ability) = self.as_ref().checked() {
            if ability.k2_commit_ability() {
                dash_forward(ability.get_avatar_actor_from_actor_info());
            }

            ability.k2_end_ability();
        }
    }
}
```

An ability overrides the Blueprint events of `GameplayAbility`: their Unreal names start with `K2_`, which `name = "..."` gives. Its functions (`k2_commit_ability`, `k2_end_ability`, the ability tasks) are the Blueprint nodes'.

## The actor

```rust
use bindings::gameplay_abilities::{AbilitySystemComponent, AbilitySystemComponentExt, AttributeSet};

#[uclass(parent = Character)]
pub struct Hero {
    #[component]
    ability_system: AbilitySystemComponent,
}

fn give_hero_its_abilities(hero: UObjectRef<Hero>) -> RustealResult<()> {
    let asc = Hero::from_obj(hero)?.ability_system()?.checked()?;

    asc.k2_init_stats(
        SubclassOf::<HealthSet>::base().upcast_to::<AttributeSet>(),
        UObjectRef::null(),
    );
    asc.k2_give_ability(SubclassOf::<Dash>::base().upcast_to::<GameplayAbility>(), None, None);

    Ok(())
}
```

The component is an `AbilitySystemComponent` of the actor. `k2_init_stats` adds an attribute set to it, with the defaults of the set's class (or the rows of a data table); `bp_apply_gameplay_effect_to_self` applies an effect, and `k2_give_ability` and `try_activate_ability_by_class` give and run an ability.

The engine's functions that look for an actor's ability system component (`AbilitySystemBlueprintLibrary::get_ability_system_component`, the effect and ability targeting) find it by its class when the actor does not implement `IAbilitySystemInterface`, a C++-only interface; a base of your own can implement it, with `GetAbilitySystemComponent` as a virtual, when a component of another actor (the player state's, as in Lyra) must be found instead (see [Engine virtuals](../classes/virtuals.md#c-only-interfaces)); that has not been tested yet.

## What is not there yet

- Tested so far: an attribute set clamping in `receive_pre_attribute_change` and `receive_pre_attribute_base_change`, an instant effect seen by `receive_post_gameplay_effect_execute`, and an ability given, activated and ended. The other events and duration effects have not been tested.

- Abilities over the network (prediction, replicated attributes): with the multiplayer work.
- Gameplay cues and execution calculations (`UGameplayEffectExecutionCalculation`, a C++ virtual) in Rust.
