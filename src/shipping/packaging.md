# Packaging a game

**The goal:** a build of the game that runs without the editor, to try it, share it or ship it.

```bash
rusteal package
rusteal package --shipping --output ~/Builds/MyGame
```

`rusteal package` builds every Rust library with the release profile, then cooks and packages the game for the platform it runs on, with the engine's own `BuildCookRun`. The packaged game stages each Rust library it loads: the game's and its runtime plugins'.

| Flag | What it does |
|---|---|
| (none) | A **Development** build, into `<project>/Packaged`. |
| `--shipping` | A **Shipping** build: no console, no logging, fully optimized. |
| `--output <dir>` | Where the packaged game goes. |
| `--optimizations off` | Leaves out the [platform optimizations](../cli/optimizations.md), which the game carries by default. |

## Before packaging

- **Assets only Rust loads by path.** The cooker packages the assets that something references. An asset that only `load_object` loads, by a path in Rust code, is not found: list its directory in **Edit > Project Settings**, **Packaging**, **Additional Asset Directories to Cook**. Assets assigned to properties in Blueprints or levels are found on their own.
- **The default maps.** **Project Settings > Maps & Modes** names the map the game opens (**Game Default Map**); a packaged game does not start from the level open in the editor.
- **Hot reload is for the editor.** A packaged game loads its libraries once, at start.

> **Screenshot:** Project Settings, Packaging, with a directory in Additional Asset Directories to Cook. *(images/shipping/additional_directories.png)*

## Running it

The packaged game is in the output directory, under the platform's name: run the executable there (`MyGame.sh` on Linux, `MyGame.exe` on Windows, the `MyGame.app` bundle on macOS).

On macOS with Xcode 27, Unreal Engine 5.8 cannot finalize a game's `.app` by itself (`Failed to finalize the .app with Xcode`, with or without Rusteal); `rusteal package` works around it and says so in its output. Packaging with the engine's own tools (the editor's **Platforms** menu, or `RunUAT` by hand) still fails until Epic fixes it.

## Sharing a Mac build

By default the `.app` is signed to run on the Mac that built it only ("Sign to Run Locally", an ad-hoc signature): on that Mac it opens, but copied to another Mac, downloaded or sent through a chat, macOS's Gatekeeper refuses to open it, because Apple cannot check who made it.

- **For a tester you trust**, they can open it anyway: Control-click the app, **Open**, then confirm, or **System Settings > Privacy & Security > Open Anyway** after the first refusal. From a terminal, `xattr -dr com.apple.quarantine MyGame.app` removes the mark the download left.
- **For players**, the game has to be signed with a Developer ID from a paid Apple Developer account and notarized by Apple. Set the team in **Project Settings > Platforms > Xcode Projects** (`CodeSigningTeam` in `[/Script/MacTargetPlatform.XcodeProjectSettings]` of `Config/DefaultEngine.ini`), package, then send the app to Apple's notary service (`xcrun notarytool submit`) and attach the ticket to it (`xcrun stapler staple MyGame.app`).

## When something is missing

A packaged game that starts but misses Rust behavior usually lacks a library or an asset:

- the log (`Saved/Logs/` next to the executable, `~/Library/Logs/<Project>/` on macOS, in a Development build) shows whether the Rusteal libraries loaded, and their errors;
- an asset that is in the editor but not in the package is the case above: add its directory to cook.
