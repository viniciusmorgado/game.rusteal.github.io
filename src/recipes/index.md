# Gameplay recipes

Complete use cases, each one taken from a Rusteal template, where it runs as part of a playable game. To see one working, create the template it comes from and play it:

| Recipe | Template it comes from |
|---|---|
| [Character movement and input](input.md) | every template |
| [Cameras](cameras.md) | `third-person`, `first-person`, `third-person --variant side-scrolling` |
| [Animation](animation.md) | `third-person --variant combat`, `third-person --variant platforming` |
| [Health, damage and death](combat.md) | `first-person --variant shooter`, `third-person --variant combat` |
| [Projectiles and weapons](projectiles.md) | `first-person --variant shooter`, `top-down --variant twin-stick` |
| [Pickups](pickups.md) | `first-person --variant shooter`, `third-person --variant side-scrolling` |
| [Platforming](platforming.md) | `third-person --variant platforming`, `third-person --variant side-scrolling` |
| [Physics](physics.md) | `third-person --variant combat` |
| [UI](ui.md) | `first-person --variant horror`, `top-down --variant strategy`, `third-person --variant combat` |
| [AI](ai.md) | `first-person --variant shooter`, `third-person --variant combat`, `top-down --variant strategy` |
| [Point and click](point_and_click.md) | `top-down` |
| [Effects and sound](effects.md) | `top-down` |
| [Vehicles](vehicles.md) | `vehicle`, `vehicle --variant time-trial` |
| [Game flow](game_flow.md) | every template |
| [Saving the game](save_game.md) | |
| [An inventory](inventory.md) | the `inventory` plugin template |

```bash
rusteal new CombatLab --template third-person --variant combat
```

The code in these recipes is trimmed to what the use case needs. The template's source, in `Rust/<crate>/src/`, has the rest.
