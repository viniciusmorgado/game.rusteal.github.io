# blank

The engine's Blank template and one Rust class, `HelloActor`, to see the whole loop work before anything else. It is the default: `rusteal new MyGame` makes it.

```bash
rusteal new MyGame --template blank
```

From `TP_Blank`, the engine template it is made of.

## Playing it

Open `<Name>.uproject`, drag a **HelloActor** from the **Place Actors** panel into the level and press **Play**: the Output Log shows `[<Name>] Hello from Rust`.

> **Screenshot:** the blank template's HelloActor message in the Output Log. *(images/templates/blank.png)*

## Its classes

The class is in `Rust/<crate>/src/hello.rs`, the place to start writing your own.

| Class | Parent | File |
|---|---|---|
| `HelloActor` | `Actor` | `src/hello.rs` |
