# Async code

**The goal:** game logic that waits, a door that opens after a delay, a level that loads before the boss appears, a message fetched from a server, written as one function that reads top to bottom instead of a chain of callbacks and timers.

```rust
use std::time::Duration;

use rusteal_runtime::runtime::{SoftObjectRef, UObjectRef, http, task, timer};

#[uclass_impl]
impl IntroGameMode {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        let me = self.as_ref();
        let welcome = Self::welcome(me, self.welcome_sound(), self.welcome_url(), self.welcome_message());

        let _task = task::spawn_local(&me, welcome);
    }
}

impl IntroGameMode {
    async fn welcome(
        me: UObjectRef<GameModeBase>,
        sound: SoftObjectRef<SoundBase>,
        url: String,
        fallback: String,
    ) {
        let sound = sound.load_async().await.ok();

        timer::sleep(&me, Duration::from_secs_f32(1.5)).await;

        let message = match http::get(&url).timeout(Duration::from_secs(5)).send().await {
            Ok(response) if response.is_success() => response.text(),
            _ => fallback,
        };

        let world: UObjectRef<Object> = me.upcast_to();

        if let Some(sound) = sound {
            GameplayStatics::play_sound2_d(world, sound, None, None, None, None, None, None);
        }

        KismetSystemLibrary::print_string(world, Some(&message), None, None, &FLinearColor::default(), Some(8.0), None);
    }
}
```

This is the intro template's welcome (since 0.10), in its game mode: it loads a sound in the background, waits a second and a half of game time, asks a server for a message, then plays the sound and shows the message, or the fallback when the server does not answer. Each `.await` lets the game run its frames until what it waits for is done.

## Tasks

`task::spawn_local(&owner, future)` runs a future on the game thread, polled once per frame when it can go on. It is the place where engine calls are allowed, so an async function calls the engine as any other code does.

| Function | What it does |
|---|---|
| `task::spawn_local(&owner, future)` | Runs `future` until it finishes, until `owner` is destroyed, or until the library unloads, whichever comes first. |
| `task::spawn_local_unowned(future)` | The same, with no owner. |
| `task::next_frame().await` | Goes on at the next frame. |
| `task::run_blocking(work).await` | Runs `work` on one of the engine's worker threads and goes on with its result (`None` if it panicked). |

`spawn_local` returns a `JoinHandle`: awaiting it gives the task's result, or `None` when the task was cancelled, and `abort()` cancels it. Dropping the handle lets the task go on. `abort_handle()` gives an `AbortHandle`, which cancels the task from any thread or callback.

A task is cancelled at an `.await`: when its owner is destroyed, the future is dropped there and the rest never runs. Give a task the object it works for as its owner, the actor or component whose state it changes.

### What a task holds

A future may live for many frames, while the engine runs other code on the same object: a tick, an overlap, a Blueprint. So a future takes the object as a handle, a `UObjectRef` or the class itself by value, never `&self` or `&mut self`:

```rust
fn unlock(&mut self) -> RustealResult<()> {
    let me = self.as_ref();
    let door = Door::from_obj(me)?;

    let _task = task::spawn_local(&me, Self::open_after(door, Duration::from_secs(2)));
    Ok(())
}

async fn open_after(door: Door, delay: Duration) {
    timer::sleep(&door.as_ref(), delay).await;
    door.open();
}
```

`Door::from_obj` gives the class by value, from its handle. The task's owner is the same object: the task stops when it is destroyed, so the class the future holds is never used on a dead object. A future that holds another object finds it again after each `.await`, with `Other::from_obj(handle)`, which fails when it is gone.

Do not keep a reference into the class (from an `x_mut()` accessor, or a borrow of a field) across an `.await`: take it again after.

## Waiting

| Await | Goes on when |
|---|---|
| `timer::sleep(&object, duration)` | `duration` of the world's time has passed: it stops while the game is paused and follows time dilation. |
| `timer::sleep_real(duration)` | `duration` of the engine's clock has passed, paused or not. |
| `delegate.next()` | the delegate fires, with its parameters (see [Delegates](delegates.md#use-case-waiting-for-a-delegate)). |
| an engine latent function | the engine's latent action finishes (below). |

## The engine's latent functions

The engine's latent Blueprint nodes, the ones with a clock icon, are `async` in Rust (since 0.10): `KismetSystemLibrary::delay`, `delay_until_next_frame`, `load_asset`, `move_component_to`, `GameplayStatics::load_stream_level`, `unload_stream_level`, and the others. They return a future of their outputs and no longer take the `FLatentActionInfo` that Blueprint hides.

```rust
KismetSystemLibrary::move_component_to(
    door,
    &FVector::from_dvec3(open_offset),
    &FRotator::from_rotator(Rotator::ZERO),
    true,
    true,
    0.6,
    false,
    EMoveComponentAction::Move,
)
.await;

GameplayStatics::load_stream_level(world, FName::new("Arena").handle(), true, false).await;
```

The call starts when the future is first awaited. Dropping it before it finishes, as a cancelled task does, aborts the engine's latent action.

## Loading assets

| Function | What it loads |
|---|---|
| `soft.load_async().await` | the asset of a `SoftObjectRef<T>`, as a `UObjectRef<T>`; an error when there is no such asset or it is not a `T` |
| `soft_class.load_async().await` | the class of a `SoftClassRef<T>`, as a `SubclassOf<T>` |
| `loading::load_all(&[soft, ...]).await` | several assets in one request, a result for each |
| `loading::load_primary_asset::<T>("Type:Name", &bundles).await` | a primary asset with its bundles, through the asset manager |

The asset manager loads them in the background, while the game runs. Nothing keeps a loaded asset in memory but what holds it: store it in a `#[uproperty]`, or pin it, before the next garbage collection. Dropping the future cancels the request.

## HTTP requests

```rust
let response = http::post("https://api.example.com/score")
    .header("Content-Type", "application/json")
    .body(format!("{{\"score\":{score}}}"))
    .timeout(Duration::from_secs(5))
    .send()
    .await?;

if response.is_success() {
    let reply = response.text();
}
```

`http::get`, `http::post` and `http::request(method, url)` build a request sent through the engine's HTTP module (since 0.10). The response has `status`, `headers` (and `header("name")`, which ignores case), `body` and `text()`. An answer with an error status is an `Ok` response; no answer at all, a refused connection or a timeout, is an `Err`. Dropping the future cancels the request.

## Async functions as Blueprint nodes

**The goal:** a Blueprint calls a Rust function that takes time, as it calls **Delay**: the node waits, then its output runs.

```rust
#[uclass_impl]
impl Door {
    #[ufunction]
    async fn open_slowly(self, seconds: f32) {
        timer::sleep(&self.as_ref(), Duration::from_secs_f32(seconds)).await;
        self.open();
    }
}
```

An `async #[ufunction]` (since 0.10) is a latent node in Blueprints, with the clock icon. It takes `self`, the class by value, for the reasons in [What a task holds](#what-a-task-holds), returns nothing (or `RustealResult<()>`, whose error is logged) and has no out parameters: the node's outputs are set when it starts, and its output pin runs when the function finishes. A second call while the first runs does nothing, as with any latent node; when the Blueprint aborts the node, or the object is destroyed, the task is cancelled and the output pin does not run. Rust code calls it as any async function: `door.open_slowly(2.0).await`. Its outputs go in properties of the class, which the Blueprint reads after the node.

In the editor, in a Blueprint's **Event Graph**, drag a wire from a `Door` reference and search for **Open Slowly**; connect its output pin to what runs after the door is open.

> **Screenshot:** the Open Slowly latent node in an Event Graph, with its clock icon. *(images/engine_api/async_latent_node.png)*

## Blueprint async actions

The engine's async action nodes, **Async Save Game to Slot**, **Async Load Game from Slot**, **Download Image** and the others, are a factory function and output delegates in Rust. Take the delegate's `next()`, activate the action, and await:

```rust
let action = AsyncActionHandleSaveGame::async_save_game_to_slot(world, save, "Slot1", 0).checked()?;
let completed = action.completed().next();
action.activate();

let (_save, saved) = completed.await?;
```

## Hot reload

A hot reload unloads the library: its tasks are dropped, at their `.await`, with everything they hold. The engine's latent actions and requests started by them are cancelled.
