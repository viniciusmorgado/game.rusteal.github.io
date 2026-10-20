# Functions

**The goal:** functions Blueprints can call, events Blueprints implement, and functions the engine calls by name (input, timers, delegates).

```rust
#[uclass_impl]
impl Turret {
    #[ufunction(BlueprintCallable)]
    fn fire(&mut self, target: UObjectRef<Actor>) {
    }

    #[ufunction(BlueprintPure)]
    fn is_reloading(&self) -> bool {
        self.reload_left() > 0.0
    }

    #[ufunction(BlueprintImplementableEvent)]
    fn on_fired(&self, muzzle_location: &OwnedStruct<FVector>) {}
}
```

A method marked `#[ufunction]` is a real Unreal function of the class. It is `BlueprintCallable` unless it says otherwise:

| Kind | What it is |
|---|---|
| `#[ufunction]` or `#[ufunction(BlueprintCallable)]` | A function with execution pins in Blueprints, and a function the engine can call by name. |
| `#[ufunction(BlueprintPure)]` | A function without execution pins: a getter, a query. |
| `#[ufunction(BlueprintImplementableEvent)]` | An event the Blueprint child implements. The Rust body is empty; calling the method runs the Blueprint's graph. |
| `#[ufunction(BlueprintNativeEvent)]` | An event with a Rust body that children, Rust or Blueprint, can override. See [Events with a Rust default](#events-with-a-rust-default). |
| `#[ufunction(Override)]` | Rust code for an event of the parent class, the engine's or a Rust parent's `BlueprintNativeEvent`. See [Engine events](events.md). |
| `#[ufunction(Exec)]` | A console command too. See [Console commands and variables](../systems/console.md). |

The other arguments:

| Argument | What it does |
|---|---|
| `name = "..."` | The Unreal name, when the method's name in PascalCase is not it. |
| `CallInEditor` | A button in the **Details** panel of the class's instances, in the editor; for a function with no parameters. |
| `category = "..."` | The category the function is listed under in the Blueprint's **Palette** and the **Details** panel. `"Combat|Ranged"` makes a subcategory. |
| `meta(...)` | Function metadata, as C++'s `meta = (...)`: `meta(Keywords = "reset clear")`, `meta(DevelopmentOnly)` for a flag. |

A doc comment on the method is its tooltip in the editor (a `ToolTip` in `meta` wins). Any other argument is a compile error.

```rust
/// Puts every counter back to zero.
#[ufunction(CallInEditor, category = "Debug")]
fn reset_counters(&mut self) {
}
```

## Parameters and return values

A function takes and returns every type a [`#[uproperty]`](properties.md) holds: scalars (`bool`, every integer, `f32`, `f64`), `String`, `FName`, `Text`, enums (engine enums and [`#[uenum]`s](enums.md)), objects (`UObjectRef<T>`), classes (`SubclassOf<T>`), soft and weak references, interfaces (`Option<ScriptInterface<dyn I>>`, `None` for no object), and structs (`OwnedStruct<T>`, a copy). Containers are Rust's: `Vec<T>`, `HashMap<K, V>` and `HashSet<T>`, copied in and out of the engine's arrays, maps and sets.

```rust
#[ufunction(BlueprintPure)]
fn count_words(&self, words: Vec<String>) -> HashMap<String, i32> {
    let mut counts = HashMap::new();

    for word in words {
        *counts.entry(word).or_insert(0) += 1;
    }

    counts
}
```

A function may take a struct the engine lends as `UStructRef<T>`, a reference into the call's parameters (`to_owned()` keeps a copy); it returns structs as `OwnedStruct<T>`. A `BlueprintImplementableEvent` takes structs as `&OwnedStruct<T>` or `OwnedStruct<T>`.

### Out parameters

A `&mut T` parameter is an output: Blueprints show it as an output pin next to the return value, and C++ passes it by reference. The function receives the value the caller's variable holds (a Blueprint passes the type's default) and writes it:

```rust
#[ufunction]
fn split_name(&self, full: String, first: &mut String, rest: &mut Vec<String>) -> bool {
    let mut words = full.split(' ').map(String::from);
    *first = words.next().unwrap_or_default();
    *rest = words.collect();
    !first.is_empty()
}
```

Called from Rust, it takes `&mut` variables, as any Rust function does. Blueprint events have no out parameters, and an `Override` of an engine event takes the engine's as `OutRef<T>` (see [Engine events](events.md)).

## Names

The Unreal name is the method's name in PascalCase: `fire` is `Fire`, `do_jump_start` is `DoJumpStart`. A method named after a Rust keyword uses a raw identifier: `fn r#move` is `Move`, `fn r#use` is `Use`. When PascalCase gets it wrong, `name` fixes it:

```rust
#[ufunction(name = "SpawnNPC")]
fn spawn_npc(&mut self) {
}
```

## Events with a Rust default

**The goal:** an event with Rust code that runs by default, which a child class, Rust or Blueprint, can replace.

```rust
#[uclass_impl]
impl Door {
    #[ufunction(BlueprintNativeEvent)]
    fn can_open(&self, by: UObjectRef<Pawn>) -> bool {
        !self.locked()
    }
}
```

This is C++'s `BlueprintNativeEvent` with its `_Implementation`. The macro renames the body to `can_open_implementation`, and `can_open` calls the event: it runs the child's override when there is one, and the body otherwise. Rust code calls `self.can_open(pawn)`; `can_open_implementation` runs this class's body only, skipping any override.

A Rust child overrides it with `#[ufunction(Override)]` and the same signature:

```rust
#[uclass(parent = Door)]
pub struct VaultDoor {}

#[uclass_impl]
impl VaultDoor {
    #[ufunction(Override)]
    fn can_open(&self, by: UObjectRef<Pawn>) -> bool {
        self.has_code(by)
    }
}
```

In a Blueprint child:

1. In the **My Blueprint** panel, hover over **Functions** and click the **Override** dropdown.
2. Select **Can Open**.
3. To run the Rust code too, right-click the new node and select **Add Call to Parent Function**.
4. Click **Compile**, then **Save**.

## Static functions and function libraries

A `#[ufunction]` with no `self` is static: Blueprints call it without a target, and Rust calls it as an associated function, `Turret::max_range()`. A class whose parent is `BlueprintFunctionLibrary` is a function library, C++'s `UBlueprintFunctionLibrary`: a set of static functions any Blueprint can call.

```rust
#[uclass(parent = BlueprintFunctionLibrary)]
pub struct DamageLibrary {}

#[uclass_impl]
impl DamageLibrary {
    /// The damage left after the armor takes its share.
    #[ufunction(BlueprintPure, category = "Damage")]
    fn after_armor(damage: f32, armor: f32) -> f32 {
        damage * 100.0 / (100.0 + armor.max(0.0))
    }
}
```

In any Blueprint, right-click the graph and search for **After Armor**. Events are never static: they take `&self` or `&mut self`.

## Functions the engine calls by name

Several engine features call a function by its Unreal name: an input action bound with `bind_action`, a timer set with `k2_set_timer`, a delegate bound with `add_ufunction`. Any `#[ufunction]` works for them:

```rust
#[ufunction]
fn respawn(&mut self) {
}

KismetSystemLibrary::k2_set_timer(self.as_ref().upcast_to(), "Respawn", 3.0, false, None, None, None);
```

A name that matches no function is logged with that name. See [Timers](../engine_api/timers.md) and [Delegates](../engine_api/delegates.md).

## Use case: letting the Blueprint play the effects

**The goal:** Rust decides when something happens; the Blueprint child decides how it looks and sounds.

```rust
#[uclass(parent = Actor)]
pub struct Pickup {
    #[component(root)]
    root: SceneComponent,
}

#[uclass_impl]
impl Pickup {
    #[ufunction(BlueprintImplementableEvent, name = "BP_OnPickedUp")]
    fn bp_on_picked_up(&self) {}

    #[ufunction(Override)]
    fn receive_actor_begin_overlap(&mut self, other_actor: UObjectRef<Actor>) {
        if other_actor.cast::<Character>().is_err() {
            return;
        }

        if let Ok(me) = self.as_ref().checked() {
            me.set_actor_enable_collision(false);
        }

        self.bp_on_picked_up();
    }
}
```

In the editor:

1. Open the Blueprint child of `Pickup` (see [Blueprint children](blueprint_child.md)).
2. Right-click the **Event Graph** and search for **BP On Picked Up**.
3. Add the **Event BP On Picked Up** node, and from it play a sound, spawn a Niagara system, hide the mesh.
4. Click **Compile**, then **Save**.

> **Screenshot:** the Event Graph of BP_Pickup with the Event BP On Picked Up node playing a sound. *(images/classes/implementable_event.png)*

The templates name these events `BP_...` so they are easy to tell apart from the engine's, and use them for every effect: hit sparks, death animations, UI updates.

## Use case: a getter for an Animation Blueprint

**The goal:** an Animation Blueprint reads gameplay state from the Rust character.

```rust
#[ufunction(BlueprintPure)]
fn has_double_jumped(&self) -> bool {
    self.state().has_double_jumped
}
```

In the Animation Blueprint's **Event Graph**, cast **Try Get Pawn Owner** to the character's class, and call **Has Double Jumped** on it. The pure function has no execution pins, so it can feed a variable directly.
