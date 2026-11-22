# Profiling

**The goal:** see where a frame's time and memory go in Rust code, with Unreal's own tools: Unreal Insights' timing view, the `stat` commands and the low-level memory tracker.

```rust
use rusteal_runtime::runtime::profiling;
use rusteal_runtime::runtime::{stat_scope, trace_scope};

fn plan_paths(&mut self) {
    trace_scope!("Squad.PlanPaths");
    stat_scope!("Squad path planning");
    let _memory = profiling::memory_tag("Squad");

    for unit in &self.units {
        trace_scope!("Squad.PlanPath");
        plan(unit);
    }
}
```

All three (since 0.10) last until the end of the block, on the thread that made them:

| What | Where it shows |
|---|---|
| `trace_scope!("Name")` | A CPU event named `Name` in **Unreal Insights** > **Timing Insights**, nested under the engine's events: run the game or the editor with `-trace=cpu` (or `-trace=default`), and open the trace in Unreal Insights. `profiling::CpuScope::new(name)` takes a name computed at run time. |
| `stat_scope!("Name")` | A cycle stat of the `Rusteal` group: `stat Rusteal` in the console. `profiling::Stat::new(name)` makes one to keep. |
| `profiling::memory_tag("Tag")` | The engine allocations made meanwhile counted under `Tag` by the low-level memory tracker: run with `-llm` and see `stat LLM` or Insights' memory view. Rust's own allocations are not engine allocations. |

`trace_scope!` and `stat_scope!` register their name once per call site. Shipping builds compile the engine's profilers out, and the scopes then only make a call that returns.
