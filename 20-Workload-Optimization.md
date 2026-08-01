# Workload Optimization — Sizing Requests/Limits Like an SRE, Not a Guesser

## The core principle

**Never pick requests and limits randomly or from a rule of thumb.** Every number in a `resources` block should trace back to actual observed behavior of the workload — CPU profiling, memory profiling, and statistical analysis of real usage over time. Guessing leads to either wasted capacity (over-provisioned, expensive) or throttling/OOMKills (under-provisioned, unreliable) — both are avoidable with the right observability discipline.

## CPU Profiling — "Where is CPU time actually spent?"

```
Application running
       │
       ▼
  Profiler samples the call stack periodically (or on specific events)
       │
       ▼
  Aggregates: which functions/code paths consumed the most CPU time
       │
       ▼
  Flamegraph visualizes this: wider = more time spent
```

**Tools:**
- **`pprof`** (Go's built-in profiler, also usable for some other runtimes) — samples CPU usage and generates call-graph profiles; commonly exposed via an HTTP endpoint (`/debug/pprof/profile`) in Go services for on-demand profiling in production.
- **`perf`** (Linux's system-wide profiler) — works at the kernel level, can profile *any* process (even without language-specific instrumentation), sampling hardware performance counters and call stacks — the go-to tool when you don't have an in-app profiler, or need to see kernel-side time (syscalls, context switches) alongside application code.
- **Flamegraphs** — the standard visualization for either tool's output:

```
Flamegraph (each box = a function; width = proportion of samples/time; stacked = call depth)

┌────────────────────────────────────────────────────────────┐
│                        main()                              │
├───────────────────────┬────────────────────────────────────┤
│      handleRequest()    │         backgroundJob()          │
├────────────┬─────────────┼───────────────┬─────────────────┤
│ parseJSON()│ dbQuery()   │  computeX()   │      gcPause    │
└────────────┴─────────────┴───────────────┴─────────────────┘
     narrow          WIDE ← most CPU time spent here (dbQuery)
```

The widest boxes are where CPU time is actually going — this is how you find "this innocuous-looking function is secretly 40% of our CPU budget" instead of guessing based on code that *looks* expensive.

## Memory Profiling

Several distinct things get conflated under "memory usage" — precision matters here:

| Concept | What it actually measures |
|---|---|
| **Heap** | Memory the application's runtime has allocated for dynamic objects — what a language-level memory profiler (e.g., Go's `pprof` heap profile, Java heap dumps) shows you. |
| **Allocation rate** | How *fast* the application is allocating (allocations/sec) — a high allocation rate, even if objects are short-lived, drives GC pressure and CPU overhead, independent of steady-state heap size. |
| **Garbage Collection (GC)** | The runtime's process for reclaiming memory from objects no longer referenced — GC pauses (stop-the-world or concurrent) show up as latency spikes, and excessive GC activity is often a symptom of undersized heap limits relative to allocation rate. |
| **Leaks** | Memory that keeps growing indefinitely because objects are unintentionally kept reachable (e.g., an ever-growing cache with no eviction, a forgotten event listener) — heap usage that never returns to baseline after load subsides is the classic symptom. |
| **RSS (Resident Set Size)** | Total physical memory actually mapped for the process, including heap **and** shared libraries **and** page cache pages currently resident — this is what the kernel/cgroup actually measures against `memory.max`. |
| **Cache** | Page cache pages the process has touched (e.g., reading files) — technically reclaimable under memory pressure, but counted in `memory.current`/RSS-adjacent metrics, which is a frequent source of "why does my container show way more memory than my app's heap size?" confusion. |
| **Working Set** | The subset of memory a process is *actively* using right now (as opposed to total RSS, which may include cold, rarely-touched pages) — closer to "what does this process really need to keep running smoothly," and what `kubectl top` and cAdvisor's `container_memory_working_set_bytes` metric attempt to approximate (this, not raw RSS, is what typically drives OOM-kill decisions in Kubernetes). |

## RSS vs Cache vs Working Set — why this distinction actually matters

A container's `memory.current` can look alarmingly high purely because of **page cache** (e.g., it read a lot of files, and the kernel opportunistically kept those pages cached for speed) — this cache is normally **reclaimable** under pressure and doesn't indicate a leak or a real problem. This is exactly why Kubernetes and cAdvisor prefer **working set** as the memory metric that drives OOM/eviction decisions, rather than raw RSS — working set attempts to exclude easily-reclaimable cache, giving a more accurate picture of memory the workload genuinely can't give up.

## Right-Sizing — using data, not peaks

```
CPU usage over a week (illustrative):

  Peak:        ▄▄▄            ▄▄
              █████          ████
  95th %ile:  █████▄▄▄▄▄▄▄▄▄▄████▄▄▄
  Average:    ████████████████████████
             └──────────────────────────► time

Sizing off PEAK alone → wildly over-provisioned most of the time (wasted cost)
Sizing off AVERAGE alone → throttled/OOM during every legitimate spike
Sizing off 95th percentile (with headroom) → the generally recommended middle ground
```

- **Average CPU** tells you steady-state cost — useful for capacity planning and cost estimation, but dangerously misleading if used alone for limits (it ignores legitimate bursts).
- **95th percentile** is the standard target for setting **requests** (and often informs limits with added headroom) — it captures "normal" behavior including most bursts, while treating true outliers as acceptable edge cases rather than the baseline to provision for.
- **Peak** usage alone is a poor sizing basis on its own — a single anomalous spike (a cold start, a GC pause, a traffic anomaly) can massively over-inflate a limit if you provision purely for the worst moment ever observed.

## Tools for right-sizing

- **VPA (Vertical Pod Autoscaler) recommendations** — Kubernetes' own mechanism for observing actual historical usage and recommending request/limit values, without necessarily auto-applying them (VPA can run in "recommendation only" mode, which is often the safest way to use it alongside manually reviewed rollouts).
- **Goldilocks** (Fairwinds) — a tool that visualizes VPA recommendations across a namespace/cluster in a friendly dashboard, making "which workloads are over/under-provisioned" immediately visible at a glance.
- **Prometheus metrics** — the raw data source underlying all of this: `container_cpu_usage_seconds_total` (rate = actual CPU usage), `container_memory_working_set_bytes`, `container_cpu_cfs_throttled_seconds_total` (direct evidence of CPU limit pain), combined with `histogram_quantile()` queries to compute your own 95th-percentile sizing directly from real cluster data rather than relying purely on external tooling.

## Interview framing (this is what separates junior from experienced answers)

"I'd never set requests/limits from a gut feeling or a generic rule of thumb. I'd pull the workload's actual CPU and memory usage from Prometheus over a representative period (ideally including peak traffic windows, not just quiet periods), look at the 95th percentile rather than the average or the single peak, cross-check with VPA/Goldilocks recommendations, and specifically watch `container_cpu_cfs_throttled_seconds_total` and OOMKill events after any change to confirm the new sizing is actually correct in practice — sizing is an iterative, observability-driven process, not a one-time guess."
