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
| `#[ufunction(Override)]` | Rust code for an event of the parent class. See [Engine events](events.md). |
| `#[ufunction(Exec)]` | A console command too. See [Console commands and variables](../systems/console.md). |

`name = "..."` gives the Unreal name when the method's name in PascalCase is not it. Any other argument is a compile error.

## Parameters and return values

A function takes scalars (`bool`, `u8`, `i32`, `i64`, `f32`, `f64`), objects (`UObjectRef<T>`), classes (`SubclassOf<T>`) and structs (`UStructRef<T>`, a reference into the call's parameters; `to_owned()` keeps a copy), and returns a scalar, an object or a class. A `BlueprintImplementableEvent` takes structs as `&OwnedStruct<T>`.

Strings and arrays are not function parameters yet. A Blueprint that needs to pass text to Rust can set a `String` property and call a function that reads it.

## Names

The Unreal name is the method's name in PascalCase: `fire` is `Fire`, `do_jump_start` is `DoJumpStart`. A method named after a Rust keyword uses a raw identifier: `fn r#move` is `Move`, `fn r#use` is `Use`. When PascalCase gets it wrong, `name` fixes it:

```rust
#[ufunction(name = "SpawnNPC")]
fn spawn_npc(&mut self) {
}
```

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
