# Troubleshooting

Problems a game developer runs into, what causes them, and the fix. The Output Log is the first place to look: Rusteal logs under `LogRusteal`.

## Building

**`rusteal build` says the bindings have not been generated yet.**
The project was never built on this machine, or `Rust/bindings/` was deleted. Run `rusteal build --all`.

**The Unreal Build Tool says another instance is running.**
It runs one build at a time per engine: the editor compiling, or a build of another project. Wait for it to finish and build again.

**Link errors in a generated wrapper after adding a module to `rusteal.toml`.**
A function the engine reflects but does not export. Add it to `[codegen.blocklist] functions` as `"Class.Function"` and run `rusteal build --all`.

**The CLI refuses the project because of its version.**
The project and the CLI are at different Rusteal versions. See [Versions and upgrades](../getting_started/versions.md).

**On Windows, Rust fails to link or the engine refuses the compiler.**
Check that Rust uses the MSVC host and that the Visual Studio components match the ones in [Prerequisites](../getting_started/prerequisites.md).

## In the editor

**On Linux, the editor closes with a segfault in the NVIDIA driver, after a window closes or PIE stops.**
The crash report shows `vkWaitForFences` under `FVulkanSwapChain::AcquireImageIndex`, and the log has `AcquireNextImage() failed due to the outdated swapchain` just before. Run `rusteal watch` once, so it writes the [platform optimizations](../cli/optimizations.md), and restart the editor. If it happens in the Project Browser, before a project is open, add the line to the engine's own config, as that page shows.

**A Rust class does not show in the Place Actors panel or the class picker.**
The library did not load, or the class is not compiled in: check the Output Log for the library's load messages and errors, and that the class's module is declared in `lib.rs` (`mod my_class;`).

**A change does not show after `rusteal build`.**
Check the Output Log for the reload. If `Rusteal.AutoReload` is `0`, run `Rusteal.Reload`. A changed `#[ustruct]` needs a restart.

**A value set in `class_defaults` does not apply to a Blueprint.**
The Blueprint child overrides it. In its **Class Defaults**, the value shows a reset arrow; click it to go back to the Rust default.

**The Rust code of an event does not run.**
A Blueprint child that implements the same event in its graph replaces it. Remove the event node from the Blueprint, or give the Blueprint a `BlueprintImplementableEvent` of its own.

**A Blueprint node is in error after a reload.**
The Rust function or property it used was removed or renamed. Fix the node or put the function back.

## Playing

**Input does nothing.**
The Output Log says which part is missing: no mapping contexts added (the player controller's Blueprint child has none, or the game mode does not use that controller), or a function name in `bind_action` that does not exist. See [Character movement and input](../recipes/input.md).

**`spawn_actor` logs that the actor has no root component.**
The spawned class has no scene component as root, so the transform is ignored. Give it a `#[component(root)]`, or spawn a class that has one.

**An engine call fails with `ObjectDestroyed`.**
The object was destroyed, or the reference was never set (a property the Blueprint child did not fill). Check the Blueprint's **Class Defaults**, and handle the error where destruction is expected.

**A panic message in the Output Log.**
Rust code panicked; the editor caught it and went on. The message and the location are in the log: fix the code and build again.

**A StateTree logs that it could not find its context actor.**
It started before its controller possessed the pawn. Start it in the controller's `receive_possess`. See [AI](../recipes/ai.md).

## Packaging

**The packaged game misses an asset.**
Only Rust code loads it, by path. Add its directory to **Additional Asset Directories to Cook**. See [Packaging a game](../shipping/packaging.md).
