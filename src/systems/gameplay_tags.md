# Gameplay tags

**The goal:** a game's tags (`Status.Stunned`, `Damage.Fire`) declared next to the code that uses them, and matched the way the engine matches them: `Damage.Fire` is a `Damage`.

```rust
use bindings::gameplay_tags::BlueprintGameplayTagLibrary;

fn declare_tags() {
    for (name, comment) in [
        ("Status.Stunned", "Cannot move or act"),
        ("Damage.Fire", "Burns over time"),
        ("Damage.Frost", "Slows the target"),
    ] {
        if let Err(e) = declare_gameplay_tag(name, comment) {
            log::error!("{e}");
        }
    }
}

rusteal_runtime::on_load!(declare_tags);

fn is_damage(tag: &OwnedStruct<FGameplayTag>) -> RustealResult<bool> {
    let damage = gameplay_tag("Damage")?;
    Ok(BlueprintGameplayTagLibrary::matches_tag(tag, &damage, false))
}
```

The `gameplay-tags` feature binds the engine's gameplay tags (see [Binding more of the engine](../cli/rusteal_toml.md)). Its prelude has two functions (since 0.10):

| Function | What it does |
|---|---|
| `declare_gameplay_tag(name, comment)` | Adds `name` to the project's tags as a native tag, as `UE_DEFINE_GAMEPLAY_TAG` does in C++, and gives it as an `OwnedStruct<FGameplayTag>`. Its parents come with it: declaring `Damage.Fire` adds `Damage`. Declaring a tag twice does nothing. |
| `gameplay_tag(name)` | The tag `name`, declared in Rust, in C++ or in the project settings; an error when the project has no such tag. |

Declare the tags from `rusteal_runtime::on_load!`, so they exist before a class or a data asset asks for them. A name with an empty part (`Damage..Fire`, a leading or a trailing dot) or a character the engine refuses in tags is refused, and the error says why.

A tag declared in Rust is a native tag, as one declared in C++ is: the project settings do not keep it, and it exists while the library is loaded. How the editor's tag pickers show tags declared after the editor started has not been tested yet.

## Matching

`BlueprintGameplayTagLibrary` has the engine's tag functions, as the Blueprint nodes are:

| Function | True when |
|---|---|
| `matches_tag(tag, other, exact)` | `tag` is `other`, or one of its children unless `exact` is set |
| `has_tag(container, tag, exact)` | the container has `tag`, or a child of it unless `exact` is set |
| `has_any_tags(container, others, exact)`, `has_all_tags(...)` | the same, for each tag of another container |

`make_gameplay_tag_container_from_tag` and `add_gameplay_tag` build a container. An actor whose tags the engine's queries see implements `IGameplayTagAssetInterface`: see [Engine virtuals](../classes/virtuals.md#c-only-interfaces).
