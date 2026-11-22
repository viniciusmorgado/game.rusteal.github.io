# Attributes

Every attribute Rusteal's macros accept. Anything not listed here is a compile error, with a message naming what is accepted.

## `#[uclass(...)]`

On a struct: the struct is an Unreal class.

| Argument | Required | What it is |
|---|---|---|
| `parent = Type` | yes | The parent class: an engine class from the bindings, or a Rust class. |
| `implements = [Trait, "path", ...]` | no | Unreal interfaces the class implements: other interfaces by class path, and `#[uinterface]` traits (after 0.7.0). See [Interfaces](../classes/interfaces.md). |
| `config = "Category"` | no | The config category its `Config` properties are read from and saved to. See [Project settings and config](../systems/settings.md). |
| `subobject_class(Name = Type, ...)` | no | Creates subobjects the parent declares as these classes: `subobject_class(CharMoveComp = ClimbingMovement)` (0.10). See [Engine virtuals](../classes/virtuals.md#use-case-a-custom-movement-mode). |

## `#[uproperty(...)]`

On a field of a `#[uclass]` or a `#[ustruct]`: the field is an Unreal property.

| Argument | What it is |
|---|---|
| `EditAnywhere` | Editable on the class defaults and on each instance. |
| `EditDefaultsOnly` | Editable on the class defaults only. |
| `VisibleAnywhere` | Shown, read-only. |
| `BlueprintReadWrite` | Get and Set nodes in Blueprints. |
| `BlueprintReadOnly` | A Get node in Blueprints. |
| `SaveGame` | Written by save game archives. |
| `Config` | Read from and saved to the class's config category. |
| `BindWidget` | On a `UserWidget` class: the widget of the same name in the Widget Blueprint's layout, which must have one. |
| `BindWidgetOptional` | The same, when the layout may leave it out. |
| `BindWidgetAnim` | On a `UserWidget` class: the Widget Blueprint's animation of the same name (a `UObjectRef<WidgetAnimation>`); it makes the property `Transient`. |
| `BindWidgetAnimOptional` | The same, when the Widget Blueprint may not have it. |
| `Transient` | Not saved with the object or its class defaults. |
| `default = value` | The default value, for scalars, engine enums and `#[uenum]`s (not in a `#[ustruct]`). |
| `name = "Name"` | The Unreal name. |
| `category = "Category"` | The category in the Details panel; `"A|B"` for a subcategory. |
| `Replicated` | Sent by the server to the clients (0.10). See [Multiplayer](../systems/multiplayer.md). |
| `ReplicatedUsing = on_rep_x` | Replicated, and `on_rep_x` runs on the client when a new value arrives (0.10). |
| `replication = "OwnerOnly"` | The replication condition, with `Replicated` or `ReplicatedUsing` (0.10). |

See [Properties](../classes/properties.md).

## `#[component(...)]`

On a field of a `#[uclass]`: the field is a default subobject, created with every instance.

| Argument | What it is |
|---|---|
| `root` | The actor's root component. |
| `attach = "field"` | The component it attaches to: a field of the class, of a Rust parent, or an inherited engine component by field-style name (`root_component`, `mesh`). |
| `socket = "Socket"` | The socket on the parent component. |
| `name = "Name"` | The component's name, when it is not the field's. |

See [Components](../classes/components.md).

## A field with no attribute

Rust's alone: any Rust type, a getter returning a clone, a `set_` setter and a `_mut` accessor. Reset by hot reload, invisible to the garbage collector.

## `#[uclass_impl]`

On an `impl` block of a `#[uclass]`: its marked methods are the class's Unreal functions, events, delegates and defaults. A class can have several.

On the implementation of a `#[uinterface]` trait for a `#[uclass]` (`impl Interactable for Door`): the class implements the interface, and every method is its function of the same name, with no attribute.

## `#[uinterface]`

Not released yet: it comes in a release after 0.7.0.

On a trait, with no arguments: the trait is an Unreal interface, Blueprint-implementable, whose functions are Blueprint events. Its functions take `&self` or `&mut self`, then `#[ufunction]` parameter types, return a `#[ufunction]` return type, and have no body; the trait has no generics and no supertraits. See [Interfaces](../classes/interfaces.md).

## `#[ufunction(...)]`

On a method in a `#[uclass_impl]` block.

| Argument | What it is |
|---|---|
| (none), `BlueprintCallable` | A function with execution pins, callable by name. |
| `BlueprintPure` | A function without execution pins. |
| `BlueprintImplementableEvent` | An event the Blueprint child implements; empty body. |
| `BlueprintNativeEvent` | An event with a Rust body children can override. |
| `Override` | An event of the parent class implemented in Rust. |
| `Exec` | Also a console command (with the default kind or `BlueprintPure`). |
| `CallInEditor` | A button in the Details panel, in the editor. |
| `ThreadSafe` | A function the engine may run on a worker thread: it takes the class's `<Class>ThreadSafe` view instead of `self` (0.10). See [Engine virtuals](../classes/virtuals.md#use-case-an-animation-update-on-a-worker-thread). |
| `Server`, `Client`, `NetMulticast` | A remote function, with `Reliable` or `Unreliable` (0.10). See [Multiplayer](../systems/multiplayer.md#use-case-remote-functions). |
| `WithValidation = valid_x` | On a `Server` function: `valid_x` checks the call first (0.10). |
| `BlueprintAuthorityOnly` | Blueprints call it on the server only (0.10). |
| `name = "Name"` | The Unreal name. |
| `category = "..."` | The category in the Palette and the Details panel. |
| `meta(...)` | Function metadata, as C++'s `meta = (...)`. |

`Override`, `BlueprintPure`, `BlueprintImplementableEvent` and `BlueprintNativeEvent` exclude each other. A doc comment is the tooltip; a method with no `self` is a static function; an `async fn` taking `self` by value is a latent node (see [Async code](../engine_api/async.md#async-functions-as-blueprint-nodes)). See [Functions](../classes/functions.md) and [Engine events](../classes/events.md).

## `#[udelegate]`

On a method with an empty body in a `#[uclass_impl]` block: a multicast delegate, assignable from Blueprints. Calling it broadcasts; `<name>_add(closure)` and `<name>_add_ufunction(&target, "Function")` bind Rust listeners, and `<name>_next()` waits for the next broadcast in async code.

| Argument | What it does |
|---|---|
| `name = "Name"` | The Unreal name. |
| `sparse` | A sparse multicast delegate: one byte in the object until it is bound. |
| `single` | A single-cast delegate: one binding, with `<name>_bind(closure)` and `<name>_bind_ufunction(&target, "Function")`; calling it does nothing when it is unbound. | See [Delegates](../engine_api/delegates.md).

## `#[class_defaults]`

On a method in a `#[uclass_impl]` block, returning `RustealResult<()>` or nothing: runs once on the class default object. See [Class defaults](../classes/class_defaults.md).

## `#[ustruct]`

On a struct whose fields are all `#[uproperty]`: an Unreal struct, with a `<Name>Ext` trait of getters and setters. See [Structs and data tables](../classes/structs.md).

## `#[uenum]`

On an enum whose variants carry no data: an Unreal enum, usable in Blueprints, with values up to 254. See [Enums](../classes/enums.md).

## Macros

| Macro | What it does |
|---|---|
| `rusteal_runtime::entry!()` | Once per library, in `lib.rs`: the entry point the plugin calls when it loads the library. |
| `rusteal_runtime::on_load!(function)` | Runs a function every time the library loads, hot reloads included. |
| `log::info!`, `log::warn!`, `log::error!` | Write to the Output Log, through the `log` crate (see [Logging](../iteration/logging.md)). |
