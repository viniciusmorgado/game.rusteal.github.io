# Platform optimizations

**The goal:** engine settings that one platform needs, applied on that platform only, on every machine you work on and in the game you ship, without you having to remember them.

Some engine settings fix or tune one platform and have no business on the others. Rusteal keeps them in `rusteal.toml`, per platform, and writes them into the engine's own per-platform config files for you.

## What every project gets

The templates ship one Linux setting, and none for Windows or macOS yet:

```toml
[optimizations.linux.Engine.ConsoleVariables]
"r.Vulkan.CpuWaitForFence" = 0
```

On Linux the engine renders with Vulkan. Whenever the swapchain goes out of date (a window is resized or closed, PIE stops, the game switches to fullscreen, alt-tab), the engine recreates it and, by default, waits for the next image with a CPU fence. NVIDIA's Linux driver can crash inside `vkWaitForFences` right then: the editor closes with a segfault in the driver a second or two after `AcquireNextImage() failed due to the outdated swapchain` in the log. With `0`, the wait is done on the GPU with a semaphore, which never calls the crashing function. It costs nothing measurable, and Epic ships the same value on Android.

The crash is not the editor's alone: a packaged game runs the same Vulkan code, so a player on Linux with an NVIDIA card can hit it when resizing or alt-tabbing. That is why the setting ships with the game too. On Windows the engine uses DirectX 12 and on macOS Metal, and the setting is never written there.

## Where it goes

The table's path names the platform, the config file and its section: `[optimizations.<platform>.<File>.<Section>]`. The platforms are `linux`, `windows` and `mac`. Rusteal writes the table of the platform it runs on into `Config/<Platform>/<Platform><File>.ini` of your project, here `Config/Linux/LinuxEngine.ini`, between two comment lines of its own:

```ini
; Rusteal optimizations, written by rusteal from rusteal.toml [optimizations]: begin
[ConsoleVariables]
r.Vulkan.CpuWaitForFence=0
; Rusteal optimizations: end
```

That file is the engine's: it reads it on Linux only, for the editor and for the packaged game alike. Rusteal only rewrites what is between its two lines:

- what you write elsewhere in the file stays as it is;
- a key you set yourself in that file, outside the Rusteal lines, keeps your value, and Rusteal leaves it out of its block.

Commit the file with the project: every Linux machine writes the same block, so it does not change from one developer to the next.

## While you develop

```bash
rusteal watch
```

`rusteal new` writes the optimizations into the new project, and `rusteal watch` writes them again each time it starts. You may work on several machines, Linux, macOS and Windows, or a teammate on another one: whoever starts `rusteal watch` on Linux has the settings, and nobody has to copy anything. The Rusteal CLI built for Windows or macOS has no Linux settings in it at all; it is compiled per platform.

The editor reads config files when it starts: after the first `rusteal watch` of a new setting, restart the editor once. Check that it took in `Saved/Logs/<Project>.log` (`~/Library/Logs/Unreal Engine/<Project>Editor/<Project>.log` on macOS):

```text
LogConfig: Set CVar [[r.Vulkan.CpuWaitForFence:0]]
```

To work without them, for instance to see whether a problem comes from one of them:

```bash
rusteal watch --optimizations off
```

That removes the block, and the file too if Rusteal had created it; the next `rusteal watch` writes it back. `--optimizations 0` is the same. Do not commit the file in that state.

## When you ship

```bash
rusteal package --shipping
```

`rusteal package` writes the optimizations of the platform it packages for, so the game carries them, then puts the project's config files back exactly as they were before the package. Packaging on Windows for Windows, or on macOS for macOS, nothing of Linux goes in: the engine stages `Config/Linux` for Linux builds only, and the Windows and macOS CLIs do not write it anyway.

To ship a game without them:

```bash
rusteal package --shipping --optimizations off
```

## Your own

Add keys and tables to `rusteal.toml`, in the same shape. If the Vulkan crash still showed up, the next console variable to try is the one that recreates the swapchain from scratch instead of reusing the old one:

```toml
[optimizations.linux.Engine.ConsoleVariables]
"r.Vulkan.CpuWaitForFence" = 0
"r.Vulkan.KeepSwapChain" = 0
```

Any config file and section works: `[optimizations.windows.Game.<Section>]` goes into `Config/Windows/WindowsGame.ini`, `[optimizations.mac.Engine.<Section>]` into `Config/Mac/MacEngine.ini`. Keys with dots, and section names with dots or slashes such as `"/Script/Engine.RendererSettings"`, go in quotes. Values are strings, numbers or booleans (`true` is written `True`). The next `rusteal watch` or `rusteal package` writes them; removing a line from `rusteal.toml` removes it from the `.ini` the same way.

- A project with no `[optimizations]` table, one made before Rusteal had them, gets the defaults above.
- An empty table turns them all off for good:

```toml
[optimizations]
```

## In the engine itself

Some crashes happen before any project is open: the Project Browser runs the same Vulkan code, and a project's config does not apply there. For those, the setting can go into the engine install, in the `[ConsoleVariables]` section that `Engine/Config/Linux/LinuxEngine.ini` already has:

```ini
[ConsoleVariables]
; larger timeout since drivers may take longer time
g.TimeoutForBlockOnRenderFence=60000
memory.MemoryPressureCriticalThresholdPercent=10
; Use semaphores instead of fences for swapchain image wait (avoids NVIDIA driver crash in vkWaitForFences after an out-of-date swapchain)
r.Vulkan.CpuWaitForFence=0
```

Know what that means before doing it:

| | The project's file (Rusteal) | The engine's file |
|---|---|---|
| Applies to | that project, its editor and its game | every project and every game packaged with that engine install, and the Project Browser |
| `--optimizations off` | leaves it out | does nothing: the engine's file is staged into every package |
| Survives an engine upgrade or reinstall | yes, it is in the project | no: a new install replaces `Engine/Config`, and the line must be added again |
| On another machine, or on CI | yes, written by `rusteal watch` and `rusteal package` | only if that engine install was changed too |

The project's file is the one that travels with the game: keep it even when the engine has the line. Use `Engine/Config/ConsoleVariables.ini` only for the editor's own debugging settings; its `[Startup]` section is ignored by Shipping builds, so a fix that the game needs does not belong there.
