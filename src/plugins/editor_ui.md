# Editor UI

**The goal:** an editor tool with a window of its own (a tab with buttons, fields and live values), and a details panel that shows a class the way the tool wants, written in Rust.

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicI32, Ordering};

use rusteal_runtime::runtime::slate::{self, Button, HorizontalBox, Slot, TextBlock, VerticalBox};

fn register() {
    let spawned = Arc::new(AtomicI32::new(0));

    let registered = slate::register_tab("LevelTools", "Level tools", move || {
        let count = spawned.clone();
        let shown = spawned.clone();

        VerticalBox::new()
            .slot(TextBlock::new("Spawn helpers"))
            .slot_with(
                Slot::default().padding(4.0),
                HorizontalBox::new()
                    .slot(Button::new().text("Spawn one").on_clicked(move || {
                        count.fetch_add(1, Ordering::SeqCst);
                    }))
                    .slot(TextBlock::new(move || format!("{} spawned", shown.load(Ordering::SeqCst)))),
            )
            .build()
    });

    if let Err(e) = registered {
        log::error!("{e}");
    }
}

rusteal_runtime::on_load!(register);
```

`slate` (since 0.10) builds the editor's own UI toolkit, Slate, from Rust, without a Widget Blueprint. `register_tab` registers a tab: **Window** lists it, and `slate::open_tab("LevelTools")` opens it, from a menu entry for one (see [Editor tools](editor_tools.md#use-case-a-menu-entry)). Its closure builds the tab's content each time the tab opens.

Register tabs from `rusteal_runtime::on_load!`: a library's tabs close and go away when it unloads, and a hot reload registers them again with the new code.

## Widgets

| Widget | What it is | Takes |
|---|---|---|
| `TextBlock::new(text)` | a line of text | a `&str`, a `String`, or a closure giving the text, read every frame |
| `Button::new()` | a button | `.text(...)` or `.content(widget)`, `.on_clicked(closure)`, `.enabled(...)` |
| `CheckBox::new(checked)` | a check box | `.on_changed(|checked| ...)` |
| `EditableTextBox::new(text)` | a text field | `.hint(...)`, `.on_changed(|text| ...)`, `.on_committed(|text| ...)` (Enter, or the focus leaving) |
| `SpinBox::new(value, min, max)` | a number field you drag or type | `.on_changed(|value| ...)` |
| `VerticalBox::new()`, `HorizontalBox::new()`, `ScrollBox::new()` | children top to bottom, left to right, or scrolling | `.slot(child)`, `.slot_with(Slot, child)` |
| `Border::new(child)` | a frame around a child | `.padding(...)` |
| `slate::spacer(width, height)` | empty space | |

`Slot::default()` places a child: `.padding(...)`, `.fill(share)` (the share of the box's free space it takes; zero is its own size), `.h_align(...)` and `.v_align(...)` (`Align::Fill`, `Start`, `Center`, `End`). Every builder ends with `.build()`, which returns a `Widget`, or goes straight into a slot.

A closure a widget calls (a click, a text read every frame) runs on the game thread and is freed with the widget. It cannot hold a `&mut` of a Rust class: share state through `Arc`, or find the object again with `from_obj`.

## Use case: a details panel the tool's way

**The goal:** the details panel of a class shows a row of its own and hides a property the tool sets itself.

```rust
use rusteal_runtime::runtime::details;

details::customize_class::<SpawnPreset>(|layout| {
    layout.hide_property("InternalSeed")?;

    let presets = layout.objects::<SpawnPreset>();

    layout.add_row(
        "Preview",
        TextBlock::new("Instances"),
        Button::new().text("Spawn them").on_clicked(move || {
            for preset in &presets {
                spawn_preset(*preset);
            }
        }),
    )
})?;
```

`details::customize_class::<T>` (since 0.10) runs its closure every time a details panel shows objects of `T` or of a child class. On the layout:

| Method | What it does |
|---|---|
| `objects::<T>()` | the objects the panel shows |
| `add_row(category, name, value)` | a row in `category`, made when the class has none of that name, with a widget on each side |
| `hide_property("Name")` | leaves out the property of that Unreal name |

`details::remove_customization::<T>()` removes it; a library's customizations go away when it unloads, so register them from `on_load!` as well.

`details::view(&object)` gives a details panel of an object as a `Widget`, to place in a tab of your own.

## What is not there yet

- The rest of Slate: lists and trees, combo boxes, images and brushes, styles, windows outside tabs. Build them in a Widget Blueprint (an Editor Utility Widget) meanwhile.
- Property type customizations (one struct's rows wherever it appears) and the details panel's own property rows customized from Rust.
