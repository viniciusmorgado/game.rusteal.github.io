# Performance

What Rust code pays to reach the engine, measured by the engine test harness (`cargo xtask engine-test` prints it), in the editor's development build, on Linux:

| Operation | Time |
|---|---|
| An addition in Rust, for scale | 0.2 ns |
| An engine function through the bindings (`KismetMathLibrary::add_int_int`) | 2.3 ns |
| The same function through reflection, building a `DynamicCall` for each call (what a dynamic call, and an event raised by name, do) | 928 ns |
| Reading and writing a Rust class's `#[uproperty]` | 4.4 ns |
| `UeArray::set_all` then `to_vec` of 100,000 `i32` | 923 µs (about 4.6 ns per element each way) |
| `UeArray::get`, per element | 34 ns |

What follows from them:

- A call through the bindings is a call into C++ through a function pointer: call engine functions from Rust freely, as C++ does.
- Reflection is hundreds of times slower: `DynamicCall`, functions called by name and events raised through `ProcessEvent` are for what the bindings do not reach, not for every frame of every actor.
- Read an engine array once (`to_vec`, `bulk_iter`) rather than element by element: `get` crosses into the engine each time.

The numbers come from one machine and a development build; measure your own hot paths with [Profiling](../iteration/profiling.md).

A Rust slice over an engine array's memory, without a copy, is not offered: any engine call made while the slice exists could reallocate the array, which safe Rust code cannot be allowed to see.
