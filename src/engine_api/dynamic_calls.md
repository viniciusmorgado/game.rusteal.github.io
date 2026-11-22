# Dynamic calls

**The goal:** call a function the bindings do not have: one a Blueprint declares, one of a module the project does not bind, one of a plugin's Rust class.

```rust
use rusteal_runtime::runtime::DynamicCall;

let mut call = DynamicCall::new(&door, "OpenDoor")?;
call.set("Speed", 2.0f32)?;
let result = call.call()?;
let opened: bool = result.get("ReturnValue")?;
```

`DynamicCall` calls any reflected function by name, with the engine's reflection, as a Blueprint node does:

| Method | What it does |
|---|---|
| `DynamicCall::new(&object, "Function")` | Finds the function on the object's class, or `Err(FunctionNotFound)`. |
| `call.set("Param", value)` | Sets a scalar parameter by its Unreal name. |
| `call.set_value("Param", &value)` | Sets a parameter of any type a [`#[ufunction]`](../classes/functions.md#parameters-and-return-values) takes: a `String`, a `Text`, a `Vec<T>`, a `HashMap<K, V>`, an `OwnedStruct<T>`, a soft reference. |
| `call.set_struct("Param", &owned_struct)` | Sets a struct parameter. |
| `call.call()` | Calls it, and returns the result. |
| `result.get::<T>("Name")` | Reads a scalar return value (`"ReturnValue"`) or out parameter. |
| `result.value::<T>("Name")` | Reads a return value or out parameter of any type `set_value` takes. |

`set` and `get` pass values as they are in memory: `bool`, numbers, `f32` for `float`, `f64` for `double`, an object's handle (`object.raw()`), an `FName`'s handle. `set_value` and `value` (since 0.10) convert, as a function's parameters do:

```rust
let mut call = DynamicCall::new(&quest_log, "AddEntry")?;
call.set_value("Title", &Text::from("The lost key"))?;
call.set_value("Tags", &vec![FName::new("Main")])?;
let result = call.call()?;
let entries: Vec<String> = result.value("Entries")?;
```

## Use case: a function of a Blueprint

**The goal:** Rust code calls a function a designer wrote in a Blueprint.

```rust
let mut call = DynamicCall::new(&actor, "PlayIntro")?;
call.set("Delay", 0.5f32)?;
call.call()?;
```

The function name and the parameter names are the ones the Blueprint shows. A Blueprint function renamed in the editor breaks the call at runtime, with a `FunctionNotFound` error, not at compile time.

## When to prefer something else

Dynamic calls skip the compiler's checks: a wrong name or a wrong type is found when the game runs. Prefer:

- the bindings, for engine functions (bind the module in `rusteal.toml` if it is missing; see [Binding more of the engine](../cli/rusteal_toml.md));
- a `BlueprintImplementableEvent`, when Rust calls into a Blueprint child of a Rust class (see [Functions](../classes/functions.md));
- an interface, when the object may be one of several classes (see [Interfaces](../classes/interfaces.md)).
