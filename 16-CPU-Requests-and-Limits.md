# CPU Requests and Limits — Scheduling Promise vs Kernel Enforcement

This is one of the most misunderstood areas in Kubernetes precisely because **requests and limits are answered by two completely different subsystems** — the scheduler (a decision made once, at Pod placement time) and the kernel (an ongoing enforcement mechanism, every single scheduling period). Conflating the two is the source of most confusion.

## CPU Request — a scheduling-time decision, not a runtime cap

```
resources:
  requests:
    cpu: "500m"     ("500 millicores" = half a CPU core)
```

`request = 500m` means: **"When deciding which node to place this Pod on, reserve 0.5 CPU worth of capacity against that node's allocatable CPU."** It is purely a **bin-packing input for the scheduler** — the kube-scheduler will not place this Pod on a node whose *remaining* allocatable CPU (after subtracting other pods' requests) is less than 500m.

```
Node has 4 allocatable cores.
Already scheduled pods have requested: 1.5 cores total.
Remaining "reservable" capacity: 2.5 cores.

New pod requests 500m → fits (0.5 ≤ 2.5) → scheduler CAN place it here.
```

**Critically: the request does not limit how much CPU the container can actually consume at runtime.** If the node has spare, unused CPU capacity at any given moment, a container can burst well above its request — the request is a *reservation for scheduling purposes*, not a runtime ceiling. (It does, secondarily, influence the container's relative `cpu.weight` for **fair scheduling under contention** — see the CFS README — but that's a proportional-fairness mechanism, not a hard cap.)

## CPU Limit — kernel enforcement via cgroups

```
resources:
  limits:
    cpu: "1"        (1 full core, expressed as a quota)
```

`limit = 1 CPU` becomes a hard `cpu.max` cgroup quota: **"100ms of CPU time allowed per 100ms period."** This *is* actively enforced by the kernel, every single period, regardless of what else is happening on the node.

```
      Period (repeats every 100ms)
 |------ 0-100ms: container consumes its full 100ms quota ------|
                                                                   │
                                                          Quota exhausted
                                                                   │
                                                                   ▼
                                                    Kernel: THROTTLE
                                                    (container cannot run
                                                     again until the NEXT
                                                     period starts, even if
                                                     8 other cores sit idle)
                                                                   │
                                                          New period begins
                                                                   │
                                                                   ▼
                                                      Container can run again
```

```
   Kernel
      │
      ▼
   Throttle
      │
      ▼
    Wait
      │
      ▼
  Run again  (next period)
```

## Why throttling happens even when the node "looks fine"

This is the single most common CPU-limit gotcha in production: a container can be throttled — appearing sluggish, with high request latency — **even though `top`/`kubectl top node` shows the node has plenty of spare CPU capacity.** Throttling is purely about *this specific cgroup's time budget for the current 100ms window*, not about overall node CPU availability. A burst of work that needs, say, 300ms of CPU time within one 100ms wall-clock window (e.g., a multi-threaded app spiking across several cores momentarily) will get throttled the instant its 100ms quota for that period is exhausted, no matter how idle the rest of the machine is.

This is measurable directly via the cgroup's `cpu.stat` file (`nr_throttled`, `throttled_usec`) — and in Prometheus/cAdvisor metrics as `container_cpu_cfs_throttled_seconds_total` — a metric every SRE tuning CPU limits should watch closely, since a nonzero, climbing throttling counter is direct evidence that a limit is actively hurting latency.

## Interview Q&A

**"Why is CPU a compressible resource?"**
Because CPU time can be delayed without destroying anything — a throttled process simply waits and resumes later; no data is lost, nothing crashes, only latency increases. This "can be paused and resumed safely" property is exactly what "compressible" means, and it's what makes CPU throttling (rather than killing) a safe enforcement strategy. (Full depth in the Compressible vs Non-Compressible README.)

**"Why is throttling better than killing?"**
Killing a process destroys in-flight work, connections, and potentially application state — a severe, disruptive failure mode. Throttling just delays execution: the process resumes exactly where it left off once the next period's quota becomes available. Since CPU is compressible (time can be deferred safely), throttling is the proportionate enforcement mechanism — reserve killing (OOM) for genuinely non-compressible resources like memory, where there's no safe way to "pause" an allocation that's already exceeded available capacity.

**"What's the practical difference between request and limit, one sentence each?"**
Request = "how much I need reserved to be scheduled" (scheduler-only, no runtime enforcement). Limit = "the hard ceiling the kernel will actually enforce every period, via cgroup CFS quota, throttling the container if it tries to exceed it."
