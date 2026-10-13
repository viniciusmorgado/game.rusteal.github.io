# Installing the CLI

Everything you do with Rusteal goes through one command, `rusteal`. Install it from crates.io:

```bash
cargo install rusteal
```

The binary carries the Unreal plugins and the project templates inside it: there is nothing else to download.

## Telling Rusteal where the engine is

The first command that needs the engine asks where it is installed:

```text
Unreal Engine root (the directory with Engine/ and Templates/):
```

Answer with the directory that holds `Engine/`, for example the engine's install directory from the Epic Games Launcher on Windows and macOS (`/Users/Shared/Epic Games/UE_5.8` by default on macOS), or the directory you extracted or built the engine in on Linux. Rusteal checks that the directory has `Engine/Build/BatchFiles` and asks again if it does not.

The answer is saved in the CLI's configuration file, `~/.config/rusteal/config.toml` on Linux, `~/Library/Application Support/rusteal/config.toml` on macOS, `%APPDATA%\rusteal\config.toml` on Windows, under `[engine].path`. It is the only setting Rusteal keeps per machine: everything else lives in the project. To point Rusteal at another engine, edit that file.

## Checking the version

```bash
rusteal --version
```

A project is tied to the Rusteal version that created it. See [Versions and upgrades](versions.md).
