# Memory Requests and Limits — Why Memory Can't Be "Paused"

## Memory Request — scheduler-only, same as CPU

```
resources:
  requests:
    memory: "256Mi"
```

Identical role to a CPU request: it's a **scheduling-time reservation**, telling the scheduler "don't place this Pod on a node that doesn't have at least 256Mi of allocatable memory left, after subtracting other pods' requests." Like CPU requests, a memory request is **not** enforced as a runtime floor or ceiling by itself — the container can use less (or, up to the limit, more) than its request at runtime.

## Memory Limit — a hard ceiling, enforced very differently from CPU

```
resources:
  limits:
    memory: "512Mi"
```

This becomes the cgroup's `memory.max` — a genuine hard ceiling. But unlike CPU, **memory cannot simply be paused and resumed** when a process wants more than is available. There's no equivalent to "wait for the next period" — if a process needs a memory page *right now* to continue, and it's not available, something has to give immediately.

```
Container requests more memory than memory.max allows
              │
              ▼
            Kernel
              │
              ▼
      OOM (Out-Of-Memory condition triggered
           for this cgroup specifically)
              │
              ▼
      Kill Process
     (the OOM killer selects a process within
      the offending cgroup — typically the largest
      consumer — and sends it SIGKILL, immediately)
```

There is no "throttle and retry later" option for memory the way there is for CPU — this is the central asymmetry between the two resource types, and it's exactly why Kubernetes treats a container hitting its memory limit ("OOMKilled", visible in `kubectl describe pod`) as a **restart event**, not a transient slowdown the way CPU throttling is.

## `memory.high` — the underused soft ceiling (worth knowing for depth)

Before the hard `memory.max` OOM-kill boundary, cgroup v2 also exposes `memory.high` — a **soft** limit that triggers aggressive reclaim and throttling-like backpressure (the kernel will try to reclaim page cache and slow the cgroup down) *before* resorting to killing anything. Kubernetes doesn't expose this directly via a simple field today in the same way as `limits.memory` → `memory.max`, but it's important conceptually: it shows that the kernel *does* have a "soft warning" mechanism for memory, it's just that Kubernetes' memory `limit` maps to the hard ceiling (`memory.max`), not the soft one.

## Interview question: "Why is memory non-compressible?"

Because a memory allocation is not something that can be **delayed without failure** — when a program needs a byte of RAM to store live state right now (e.g., writing to an array, allocating a new object), there is no safe way to say "wait 50ms and try again" the way there is with CPU scheduling. CPU throttling works because *time itself* is the resource being rationed, and time can always be deferred to later. Memory isn't like time — it's a fixed, physical capacity, and once it's genuinely exhausted, the only options are: reclaim memory from somewhere (evict page cache, swap — if enabled and configured, which Kubernetes normally disables), or **forcibly free memory by killing something**. There's no graceful "come back later" path once truly out of memory, which is exactly why the enforcement mechanism for memory limits is fundamentally more violent (OOM kill) than for CPU limits (throttle and resume).

## Why this asymmetry actually matters operationally

- Sizing CPU limits too tight causes **latency** (throttling) — annoying, visible in metrics, but self-healing once load drops.
- Sizing memory limits too tight causes **crashes** (OOMKilled) — a hard failure, pod restart, potential data loss for in-flight requests, and if it happens repeatedly, `CrashLoopBackOff`.

This asymmetry is *the* reason SRE guidance typically leans toward being more conservative/generous with memory limits than CPU limits — the failure mode for getting memory wrong is categorically worse than the failure mode for getting CPU wrong, so memory sizing deserves more careful profiling (see the Workload Optimization README) before setting a hard ceiling.
