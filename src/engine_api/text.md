# Text and localization

Unreal has two string types. `FString` is plain text, `String` and `&str` in Rust. `FText` is text shown to the player: it carries what the localization system needs to translate it, and it is `Text` in Rust (since 0.10).

## Text

A `Text` is the engine's own `FText`, held by Rust: cloning it is cheap, and dropping it releases it.

| Expression | What it is |
|---|---|
| `Text::from("Score")`, `Text::from_string(&s)` | A text that is never translated, such as a player's name or a number already formatted. |
| `Text::localized("Hud", "Score", "Score")` | A text with a namespace and a key, which the localization data translates; the last argument is the source string, shown when there is no translation. |
| `Text::empty()`, `Text::default()` | The empty text. |
| `text.to_string()`, `format!("{text}")` | The string shown, in the current language. |
| `text.is_empty()`, `a == b` | |

Engine functions that take an `FText` take `impl Into<Text>`, so a `&str`, a `String` or a `Text` all work; those that give one give a `Text`:

```rust
label.set_text("Game over");
label.set_text(format!("Score: {score}"));
label.set_text(Text::localized("Hud", "GameOver", "Game over"));

let shown: Text = label.get_text();
```

An `FText` parameter with a default value in C++ is a required one in Rust: pass `""` for the empty text.

## Translating texts

The engine translates a text by its namespace and key, from the localization data of the project, which its text gatherer builds from C++ (`LOCTEXT`, `NSLOCTEXT`), assets and String Tables.

```rust
use rusteal_runtime::runtime::loctext;

label.set_text(loctext!("Hud", "GameOver", "Game over"));
```

`loctext!(namespace, key, source)` (since 0.10) is a `Text::localized` the gatherer sees: on every `rusteal build`, Rusteal reads the `loctext!` calls of the game's crate and of its plugins' crates and writes them as `NSLOCTEXT` lines to `Source/<Project>/RustealText.h`, a header nothing includes, which the gather reads with the project's C++. The **Localization Dashboard**'s **Gather Text** then finds them with **Gather from Text Files** searching `Source`, its default. The three arguments are string literals; a `Text::localized` built from variables is translated as well, but the gather does not find it.

The other way is a String Table: an asset the gatherer reads, with a key per text. Create it in the editor (**Content Browser** › **Add** › **Miscellaneous** › **String Table**), add the texts, and read them by the table's id and the key:

```rust
let title = KismetTextLibrary::text_from_string_table(table_id, "Title");
```

where `table_id` is `FName::new(path).handle()`, `path` being the table's path as **Copy Reference** in its context menu in the **Content Browser** gives it, without the `StringTable'...'` around it.

`KismetTextLibrary::get_text_id(&text)` gives a text's namespace and key, to check where a text comes from.
