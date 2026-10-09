# Prerequisites

Rusteal needs three things on the machine: Unreal Engine 5.8, Rust 1.88 or newer, and the C++ toolchain the engine builds with. The versions below come from the engine's own requirements.

## Linux (x64)

| Dependency | Version | Where from | What for |
|---|---|---|---|
| A C toolchain | any | `build-essential` (Debian, Ubuntu), `gcc` (Fedora), `base-devel` (Arch) | the linker Rust uses (`cc`) |
| Rust | stable, 1.88 or newer | [rustup](https://rustup.rs) | installing the `rusteal` CLI, building your game crate |
| Unreal Engine | 5.8 | Epic's Linux build ([unrealengine.com/linux](https://www.unrealengine.com/linux)) or built from source | the game |

The engine brings the rest: the clang toolchain the Unreal Build Tool compiles with and the .NET SDK that runs it.

## Windows (x64)

| Dependency | Version | Where from | What for |
|---|---|---|---|
| Visual Studio | Community, Professional or Enterprise; 2022 17.8 or newer, or 2026 18.0 or newer | [visualstudio.microsoft.com](https://visualstudio.microsoft.com) | the C++ compiler the engine uses, and the linker Rust uses |
| Rust | stable, 1.88 or newer, `x86_64-pc-windows-msvc` | [rustup](https://rustup.rs) (`rustup-init.exe`) | installing the `rusteal` CLI, building your game crate |
| Unreal Engine | 5.8 | the Epic Games Launcher | the game |

Install Visual Studio before Rust: rustup uses its build tools for the MSVC toolchain.

### Visual Studio workloads

In the Visual Studio Installer (**Modify** on an existing install), the **Workloads** tab needs two workloads:

| Workload | Section | What the engine takes from it |
|---|---|---|
| **Desktop development with C++** | Desktop & Mobile | MSVC, the C++ compiler and linker; the Windows SDK |
| **.NET desktop development** | Desktop & Mobile | the .NET Framework SDK |

> **Screenshot:** the Workloads tab of the Visual Studio Installer with both workloads checked. *(images/getting_started/vs_workloads.png)*

Then, in the **Individual components** tab, check that these are selected:

- **MSVC v143 x64/x86 build tools 14.44** for Visual Studio 2022, or **MSVC x64/x86 build tools 14.50** for Visual Studio 2026. The engine refuses 14.39 to 14.43, 14.44 before 14.44.35211 and 14.50 before 14.50.35723;
- **Windows 11 SDK 10.0.22621** (10.0.19041 at least);
- **.NET Framework 4.6.2 targeting pack**.

> **Screenshot:** the Individual components tab with the MSVC build tools, the Windows SDK and the .NET Framework targeting pack selected. *(images/getting_started/vs_components.png)*

### The Rust host

Rust must use the MSVC host. `rustup show` prints:

```text
Default host: x86_64-pc-windows-msvc
```

If your default toolchain is a `-windows-gnu` one, for another project, it can stay. Set the default host and install the MSVC toolchain:

```powershell
rustup set default-host x86_64-pc-windows-msvc
rustup toolchain install stable-x86_64-pc-windows-msvc
```

Then pin it in your game project with a `rust-toolchain.toml`, or for the directory that holds your projects:

```powershell
rustup override set stable-x86_64-pc-windows-msvc --path <dir>
```

## macOS

Rusteal has not been tested on macOS yet.
