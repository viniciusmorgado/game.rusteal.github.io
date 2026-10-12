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
| `--static` | Links the Rust code into the game's executable instead of shipping it as libraries (below). |
| `--platform <name or file>` | The platform to package for (below); the one the CLI runs on by default. |
| `--optimizations off` | Leaves out the [platform optimizations](../cli/optimizations.md), which the game carries by default. |

## Linking the Rust code into the game

```bash
rusteal package --static
rusteal package --static --shipping
```

By default the packaged game ships the Rust code as it runs in the editor: one dynamic library for the game and one per runtime plugin, loaded when the game starts. With `--static`, the game's crate and every runtime plugin's crate are compiled together into one static library, optimized as a whole, and linked into the game's executable: the package holds no Rust library at all, and nothing can replace one.

That is how consoles take native code, so a game that packages with `--static` on PC has already done most of what a console build needs. The editor is unchanged: it still loads the libraries and hot reloads them.

What happens:

- the editor's libraries are built in release and the game is cooked with them, as always;
- the Rust crates are compiled again, together, into `Intermediate/Rusteal/Static/` (the first time takes a few minutes; the crates you depend on are compiled once for it, and the build keeps several gigabytes in `Intermediate/Rusteal/Static/target/`, which can be deleted at any time);
- before the game starts using it, each library checks that every class, struct, enum, interface and `on_load!` hook it registers made it into the executable, and refuses to start, in the log, if one did not.

Two things to know:

- **Each crate with Rust classes belongs to one library.** A helper crate in your game's `Rust/` workspace is part of the game's library; the same crate used by the game and by a plugin is refused, since its classes would be registered twice.
- **Plugins made with Rusteal before 0.9** stage their dynamic library in the package anyway, unused: `rusteal package --static` lists them. Their `Build.cs` adds the library itself; replace those lines with `UnrealBuildTool.Rules.Rusteal.AddRustLibrary(this, Target, PluginDirectory, "rusteal_<Plugin>");`, as a new plugin does.

## Packaging for another platform

```bash
rusteal platform list
rusteal package --static --platform LinuxArm64
rusteal package --static --platform ~/platforms/my-console.toml
```

A platform is a definition: the name the engine gives it, the Rust target its code is compiled for, the archive's name, and what the toolchain needs. Linux, LinuxArm64, Win64 and Mac are built in; `rusteal platform show <name>` prints one as a file:

```toml
[platform]
name = "Linux"
static_archive = "librusteal_static.a"

[platform.dynamic_library]
prefix = "lib"
extension = "so"
```

Any other platform comes from a definition file: given as `--platform <file>`, or found by its name in `~/.config/rusteal/platforms/` (`%APPDATA%\rusteal\platforms\` on Windows) or in a directory listed in `RUSTEAL_PLATFORM_PATH`. Besides the fields above, a definition can set `rust_target` (a target triple, or a target JSON file next to the definition), `toolchain` (a rustup toolchain, such as a nightly one), `build_std` (the standard library crates to build for the target), `cargo_args`, `rustflags`, `env`, `system_libraries` (linked with the archive) and `uat_args` (added to `BuildCookRun`).

A platform other than the one the CLI runs on is packaged with `--static` only. The engine must support packaging for it from this machine, with its SDK installed.

## Before packaging

- **Assets only Rust loads by path.** The cooker packages the assets that something references. An asset that only `load_object` loads, by a path in Rust code, is not found: list its directory in **Edit > Project Settings**, **Packaging**, **Additional Asset Directories to Cook**. Assets assigned to properties in Blueprints or levels are found on their own.
- **The default maps.** **Project Settings > Maps & Modes** names the map the game opens (**Game Default Map**); a packaged game does not start from the level open in the editor.
- **Hot reload is for the editor.** A packaged game loads its libraries once, at start.

> **Screenshot:** Project Settings, Packaging, with a directory in Additional Asset Directories to Cook. *(images/shipping/additional_directories.png)*

## Running it

The packaged game is in the output directory, under the platform's name: run the executable there (`MyGame.sh` on Linux, `MyGame.exe` on Windows, the `MyGame.app` bundle on macOS, `MyGame-Mac-Shipping.app` for a `--shipping` build).

On macOS with Xcode 27, Unreal Engine 5.8 cannot finalize a game's `.app` by itself (`Failed to finalize the .app with Xcode`, with or without Rusteal); `rusteal package` works around it and says so in its output. Packaging with the engine's own tools (the editor's **Platforms** menu, or `RunUAT` by hand) still fails until Epic fixes it ([the report](https://forums.unrealengine.com/t/5-8-mac-failed-to-finalize-the-app-with-xcode-on-xcode-27-xcodebuild-started-without-stdin-bad-file-descriptor-in-touch-ubt-generated-tiles/2839864)).

## Sharing a Mac build

By default the `.app` is signed to run on the Mac that built it only ("Sign to Run Locally", an ad-hoc signature): on that Mac it opens, but copied to another Mac, downloaded or sent through a chat, macOS's Gatekeeper refuses to open it, because Apple cannot check who made it.

- **For a tester you trust**, they can open it anyway: Control-click the app, **Open**, then confirm, or **System Settings > Privacy & Security > Open Anyway** after the first refusal. From a terminal, `xattr -dr com.apple.quarantine MyGame.app` removes the mark the download left.
- **For players**, the game has to be signed with a Developer ID from a paid Apple Developer account and notarized by Apple. Set the team in **Project Settings > Platforms > Xcode Projects** (`CodeSigningTeam` in `[/Script/MacTargetPlatform.XcodeProjectSettings]` of `Config/DefaultEngine.ini`), package, then send the app to Apple's notary service (`xcrun notarytool submit`) and attach the ticket to it (`xcrun stapler staple MyGame.app`).

## When something is missing

A packaged game that starts but misses Rust behavior usually lacks a library or an asset:

- the log (`Saved/Logs/` next to the executable, `~/Library/Logs/<Project>/` on macOS, in a Development build) shows whether the Rusteal libraries loaded, and their errors;
- an asset that is in the editor but not in the package is the case above: add its directory to cook;
- in a `--static` package, a line such as `MyGame: 2 of its registrations were not linked into the game: class:my_game:Door, ...` names Rust types the linker left out; report it as a Rusteal bug, with the log.
