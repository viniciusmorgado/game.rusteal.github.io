# Multiplayer

**The goal:** a game whose state the server owns and the clients see: a replicated health bar, a door every player sees open, a weapon a client fires through the server, a score the server announces to everyone.

Rusteal replicates as Unreal does. The server is the authority; an actor that replicates is created on the clients by the engine, with its replicated properties; and functions marked as remote run on another machine.

## Use case: replicated properties

```rust
use bindings::engine::{Actor, ActorExt, SceneComponent};

#[uclass(parent = Actor)]
pub struct ControlPoint {
    #[component(root)]
    root: SceneComponent,

    #[uproperty(BlueprintReadOnly, ReplicatedUsing = on_rep_progress)]
    progress: f32,

    #[uproperty(Replicated)]
    team: String,

    #[uproperty(Replicated, replication = "OwnerOnly")]
    reward: i32,
}

#[uclass_impl]
impl ControlPoint {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        self.as_ref().checked()?.set_replicates(true);
        Ok(())
    }

    #[ufunction]
    fn on_rep_progress(&mut self, old_progress: f32) {
        if self.progress() >= 1.0 && old_progress < 1.0 {
            play_capture_sound(self.as_ref());
        }
    }
}
```

`#[uproperty]` takes (since 0.10):

| Argument | What it does |
|---|---|
| `Replicated` | The server sends the property's value to the clients when it changes. |
| `ReplicatedUsing = on_rep_x` | The same, and the client calls the `#[ufunction]` `on_rep_x` when a new value arrives; it may take the value it had before. |
| `replication = "OwnerOnly"` | The condition: `InitialOnly`, `OwnerOnly`, `SkipOwner`, `SimulatedOnly`, `AutonomousOnly`, `SimulatedOrPhysics`, `InitialOrOwner`, `Custom`, `ReplayOrOwner`, `ReplayOnly`, `SimulatedOnlyNoReplay`, `SimulatedOrPhysicsNoReplay`, `SkipReplay`, `Dynamic` or `Never`, the names of C++'s `COND_` values. |

The actor replicates when `set_replicates(true)` is in its class defaults, as `bReplicates = true` is in a C++ constructor. Every property type replicates: numbers, strings, enums, object references, structs and arrays. Change replicated properties on the server, with `has_authority()` to know where the code runs; a client's change stays on that client.

## Use case: remote functions

```rust
#[uclass_impl]
impl Blaster {
    #[ufunction(Server, Reliable, WithValidation = valid_aim)]
    fn server_fire(&mut self, pitch: f32, yaw: f32) {
        self.spawn_projectile(pitch, yaw);
        self.multicast_muzzle_flash();
    }

    fn valid_aim(&self, pitch: f32, _yaw: f32) -> bool {
        pitch.abs() <= 90.0
    }

    #[ufunction(NetMulticast, Unreliable)]
    fn multicast_muzzle_flash(&mut self) {
        play_muzzle_flash(self.as_ref());
    }

    #[ufunction(Client, Reliable)]
    fn client_out_of_ammo(&mut self) {
        show_empty_clip_hint(self.as_ref());
    }
}
```

| Argument | Where the body runs when the function is called |
|---|---|
| `Server` | On the server, called from the client that owns the actor. |
| `Client` | On the client that owns the actor, called from the server. |
| `NetMulticast` | On the server and every client, called from the server. |
| `Reliable` or `Unreliable` | Whether the call always arrives, in order, or may be dropped (one of them is required). |
| `WithValidation = valid_x` | The server calls `valid_x` with the same parameters first; `false` drops the call and disconnects the client that sent it, as C++'s `_Validate` does. |
| `BlueprintAuthorityOnly` | On any `#[ufunction]`: Blueprints call it on the server only. |

Calling the method (`self.server_fire(pitch, yaw)`, or on a handle from `from_obj`) asks the engine to send the call: on the machine it lands on, the method's body runs. A remote function takes `&self` or `&mut self`, returns nothing and has no out parameters; it is callable from Blueprints too.

Who owns an actor decides where `Server` and `Client` calls go: the player controller, its pawn, and an actor spawned with `spawn_actor_with_owner` or given an owner with `set_owner`.

The engine's own remote functions are in the bindings as well (since 0.10), for instance `PlayerController::server_change_name`.

## Use case: replicated components and objects

```rust
use bindings::engine::{ActorComponent, ActorComponentExt};
use bindings::manual::world_ext::new_object;
use bindings::rusteal::RustealReplicatedObject;
use rusteal_runtime::runtime::net;

#[uclass(parent = ActorComponent)]
pub struct Ammo {
    #[uproperty(Replicated)]
    rounds: i32,
}

#[uclass_impl]
impl Ammo {
    #[class_defaults]
    fn class_defaults(&mut self) -> RustealResult<()> {
        self.as_ref().checked()?.set_is_replicated(true);
        Ok(())
    }
}

#[uclass(parent = RustealReplicatedObject)]
pub struct Loadout {
    #[uproperty(Replicated)]
    slots: UeArray<i32>,
}

fn give_loadout(soldier: UObjectRef<Soldier>) -> RustealResult<()> {
    let loadout = new_object::<Loadout>(&soldier)?;
    Soldier::from_obj(soldier)?.set_loadout(loadout);
    net::add_replicated_subobject(&soldier, &loadout, 0)
}
```

A component replicates its own replicated properties when `set_is_replicated(true)` is in its class defaults. A plain object replicates when its class derives from `RustealReplicatedObject` (since 0.10), its outer is an actor, and the server registers it with `net::add_replicated_subobject(&actor, &object, condition)`; a replicated property of the actor that refers to it reaches the clients as that same object. Its remote functions go through its actor's connection. `net::remove_replicated_subobject` stops it.

## Push model and Iris

Rust properties replicate under the engine's two optional systems as they do by default:

- **Push model** (`Net.IsPushModelEnabled=1`, which Lyra turns on): a replicated property is sent when it is marked as changed, and the generated setters mark it, as do the accessors of replicated arrays, maps and sets (`values().push(...)`).
- **Iris** (the Iris plugin, `net.Iris.UseIrisReplication=1` or `-UseIrisReplication=1`): properties, components, subobjects and remote functions replicate through it. A failed `WithValidation` drops the call without disconnecting the client under Iris, as it does for C++.

## A dedicated server

```bash
rusteal package --server
```

`--server` (since 0.10) packages a dedicated server instead of the game: it writes `Source/<Project>Server.Target.cs` when the project has none, and runs Unreal's packaging with `-server -noclient`. A dedicated server is built with an engine built from source: an installed engine (the Epic Games Launcher's, or a binary release) has no server target, and `--server` says so. The server loads the same Rust library as the game; `KismetSystemLibrary::is_dedicated_server` tells server code from client code.

Play In Editor runs a dedicated server and clients in one editor process: **Play** > **Net Mode** > **Play As Client**, with **Number of Players** set to 2 or more.

## What is not there yet

- Code that is left out of clients or servers at compile time: the same Rust library runs everywhere.
- `rusteal package --server` has not been tested yet: it needs an engine built from source.
- Custom serialization (`NetSerialize`, `FFastArraySerializer`): use a C++ struct for it.
- A template that plays over the network: the templates are single player.
