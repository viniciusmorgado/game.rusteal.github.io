# UI

Rusteal UI follows one pattern: a Rust class whose parent is `UserWidget` declares what the widget can be told, as `BlueprintImplementableEvent`s; a **Widget Blueprint** child of it draws the layout in the UMG designer and implements those events; the game creates the widget from a class a Blueprint assigns.

## Use case: a counter on screen

**The goal:** a pickups counter that appears with the first pickup and updates with each one.

The widget class:

```rust
use bindings::umg::UserWidget;

#[uclass(parent = UserWidget)]
pub struct CoinsUI {}

#[uclass_impl]
impl CoinsUI {
    #[ufunction(BlueprintImplementableEvent)]
    pub fn update_coins(&self, amount: i32) {}
}
```

The game mode creates it and tells it the count:

```rust
use bindings::engine::{GameModeBase, GameplayStatics};
use bindings::umg::UserWidgetExt;

#[uclass(parent = GameModeBase)]
pub struct CoinGameMode {
    #[uproperty(EditAnywhere)]
    user_interface_class: SubclassOf<CoinsUI>,

    #[uproperty]
    user_interface: UObjectRef<CoinsUI>,

    #[uproperty(BlueprintReadOnly)]
    coins: i32,
}

#[uclass_impl]
impl CoinGameMode {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        if let Err(e) = self.create_user_interface() {
            ulog!(LOG_WARNING, "[MyGame] UI: {e}");
        }
    }
}

impl CoinGameMode {
    fn create_user_interface(&mut self) -> RustealResult<()> {
        let player = GameplayStatics::get_player_controller(self.as_ref().upcast_to(), 0);
        let widget = create_widget_of_class(&player, self.user_interface_class())?;
        self.set_user_interface(widget);
        Ok(())
    }

    pub fn add_coin(&mut self) -> RustealResult<()> {
        let coins = self.coins() + 1;
        self.set_coins(coins);

        let ui = self.user_interface();

        if coins == 1 {
            ui.upcast_to::<UserWidget>().checked()?.add_to_viewport(Some(0));
        }

        CoinsUI::from_obj(ui)?.update_coins(coins);

        Ok(())
    }
}
```

- `create_widget_of_class(&owning_player, class)` is C++'s `CreateWidget<T>(PlayerController, Class)`.
- `add_to_viewport` shows it over the whole screen; `add_to_player_screen` over the owning player's part of a split screen; `remove_from_parent` hides it again.
- `CoinsUI::from_obj(widget)` reaches the Rust class of the widget, whose events run the Widget Blueprint's graph.

### In the editor

1. In the **Content Browser**, right-click and choose **User Interface > Widget Blueprint**, and name it `UI_Coins`.
2. Open it, switch to **Graph** (top right), click **Class Settings** in the toolbar, and in the **Details** panel set **Parent Class** to **CoinsUI**.
3. Switch to **Designer**, add a **Canvas Panel** and a **Text** inside it. Select the **Text**, check **Is Variable** in the **Details** panel and name it `CoinsText`.
4. Switch to **Graph**. Right-click and add **Event Update Coins**. From it, get **CoinsText**, call **SetText (Text)**, and convert **Amount** to text with **To Text (Integer)**.
5. **Compile** and **Save**.
6. Open the game mode's Blueprint child, and in **Class Defaults** set **User Interface Class** to **UI_Coins**.

> **Screenshot:** UI_Coins's Graph with Event Update Coins setting the CoinsText text. *(images/recipes/ui_update_coins.png)*

## Use case: a widget that listens to the character

**The goal:** a stamina meter that updates whenever the character's stamina changes.

The horror template has the player controller create the widget on possess, and hand the character to it:

```rust
#[uclass(parent = UserWidget)]
pub struct HorrorUI {}

#[uclass_impl]
impl HorrorUI {
    #[ufunction(BlueprintImplementableEvent, name = "BP_SprintMeterUpdated")]
    pub fn bp_sprint_meter_updated(&self, percent: f32) {}

    #[ufunction(BlueprintImplementableEvent, name = "BP_SprintStateChanged")]
    pub fn bp_sprint_state_changed(&self, sprinting: bool) {}
}
```

```rust
#[ufunction(Override)]
fn receive_possess(&mut self, possessed_pawn: UObjectRef<Pawn>) {
    let me: UObjectRef<PlayerController> = self.as_ref();

    if !self.horror_ui().is_valid()
        && let Ok(widget) = create_widget_of_class(&me, self.horror_ui_class())
        && let Ok(checked) = widget.checked()
    {
        checked.add_to_player_screen(Some(0));
        self.set_horror_ui(widget);
    }
}
```

The character then calls the widget's events when the stamina changes. A [delegate of your own](../engine_api/delegates.md) (`#[udelegate]`) on the character does the same job and lets any number of widgets, Blueprint or Rust, listen.

## Use case: a health bar above a character

**The goal:** a bar that floats over each enemy.

```rust
use bindings::umg::{WidgetComponent, WidgetComponentExt};

#[uclass(parent = Character)]
pub struct Enemy {
    #[component(attach = "root_component")]
    life_bar: WidgetComponent,

    #[uproperty]
    life_bar_widget: UObjectRef<LifeBar>,
}

#[uclass_impl]
impl Enemy {
    #[ufunction(Override)]
    fn receive_begin_play(&mut self) {
        if let Ok(widget) = self
            .life_bar()
            .and_then(|bar| bar.checked())
            .and_then(|bar| bar.get_user_widget_object().cast::<LifeBar>())
        {
            self.set_life_bar_widget(widget);
        }
    }
}
```

The Blueprint child sets the **Widget Class** of the **Life Bar** component to a Widget Blueprint whose parent is `LifeBar`, and its **Space** to **Screen**. Updating it is `LifeBar::from_obj(self.life_bar_widget())?.set_life_percentage(hp / max_hp)`.

## Use case: drawing on the HUD

**The goal:** a selection box drawn while dragging the mouse.

```rust
use bindings::engine::{HUD, HUDExt};

#[uclass(parent = HUD)]
pub struct StrategyHUD {
    #[uproperty(EditAnywhere)]
    selection_box_color: OwnedStruct<FLinearColor>,

    drawing: bool,
    box_start: DVec2,
    box_size: DVec2,
}

#[uclass_impl]
impl StrategyHUD {
    #[ufunction(Override)]
    fn receive_draw_hud(&mut self, _size_x: i32, _size_y: i32) {
        if !self.drawing() {
            return;
        }

        let Ok(me) = self.as_ref().checked() else {
            return;
        };

        let (start, size) = (self.box_start(), self.box_size());
        me.draw_rect(&self.selection_box_color(), start.x as f32, start.y as f32, size.x as f32, size.y as f32);
    }
}
```

`receive_draw_hud` runs every frame; `draw_rect`, `draw_text`, `draw_line` and `draw_texture` draw on the canvas. `get_actors_in_selection_rectangle` returns the actors inside a screen rectangle, which is how the strategy template selects units with a box. The game mode's Blueprint child sets **HUD Class** to the HUD's Blueprint.
