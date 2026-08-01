# cgroups v2 — The Actual Mechanism Behind Kubernetes Requests and Limits

## Why cgroups exist

Namespaces (see the Namespaces README) give a process an *isolated view* of the system (its own PIDs, network, mounts). But isolation alone doesn't stop one process from consuming all the CPU, memory, or disk I/O on a machine. **cgroups (control groups)** are the kernel mechanism that actually **meters and limits resource usage** for a group of processes — this is the real enforcement layer underneath every Kubernetes request/limit.

## The full chain from Kubernetes YAML to kernel enforcement

```
   Kubernetes Pod spec (resources.requests/limits)
              │
              ▼
        kubelet reads the spec
              │
              ▼
        CRI (Container Runtime Interface) call
              │
              ▼
          containerd
              │
              ▼
             runc          (creates the actual container process)
              │
              ▼
         cgroup v2         (kernel writes: cpu.max, memory.max, etc.)
              │
              ▼
        Linux Kernel        (the kernel scheduler & memory manager
                              actually ENFORCE these numbers)
```

Nothing in this chain is magic — Kubernetes, containerd, and runc are all just **convenience layers that ultimately write specific values into cgroup control files**. The kernel itself has no concept of "Kubernetes Pod" — it only knows about cgroups and the processes assigned to them.

## Key cgroup v2 controllers and files

| File | Purpose |
|---|---|
| `cpu.max` | The CPU quota: `"<quota> <period>"` in microseconds — e.g., `100000 100000` means "100ms of CPU time allowed per 100ms period" (i.e., 1 full core). This is what a Kubernetes CPU **limit** becomes. |
| `cpu.weight` | Relative CPU share (1-10000, default 100) used for proportional scheduling *under contention* — this is what a Kubernetes CPU **request** effectively influences (translated into relative weight for the CFS scheduler when cores are contended). |
| `memory.max` | Hard memory ceiling — exceeding this triggers the **OOM killer** for processes in this cgroup. This is what a Kubernetes memory **limit** becomes. |
| `memory.high` | A *soft* throttling ceiling — exceeding it doesn't trigger OOM kill immediately, but the kernel aggressively reclaims memory (reduces page cache, applies backpressure) from the cgroup to push usage back down, and can slow the cgroup down as a "warning" pressure valve before a hard OOM. |
| `memory.current` | Read-only — current actual memory usage of the cgroup right now (what tools like `kubectl top` ultimately read, transitively). |
| `io.max` | Per-device I/O limits (bytes/sec and/or IOPS) for read/write — enforces disk I/O ceilings on a cgroup. |
| `pids.max` | Maximum number of processes/threads a cgroup may create — protects against fork bombs or runaway thread creation from exhausting the system's PID space. |

## How `cpu.max` actually enforces a limit

```
cpu.max = "50000 100000"   (meaning: 50ms quota per 100ms period → 0.5 CPU limit)

Period timeline (repeats every 100ms):
 |---- 0ms to 50ms: container CAN run, consuming its quota ----|---- 50ms to 100ms: quota exhausted, THROTTLED (cannot run even if CPU is idle) ----|
 |<--------------------------- 100ms period ------------------------------------->|
                                                                                  │
                                                                      new period starts, quota resets
```

If the container's threads try to use more than 50ms of actual CPU time within any single 100ms period, the kernel simply **stops scheduling them** for the remainder of that period — regardless of whether other cores are sitting completely idle. This is exactly why a pod can be "throttled" even on a mostly-idle node — the constraint is a **time budget**, not literal core availability.

## How Kubernetes maps requests/limits to cgroups

```
resources:
  requests:
    cpu: "500m"        →  cpu.weight ≈ proportional share (used only under contention)
  limits:
    cpu: "1"            →  cpu.max = "100000 100000"  (hard 1-core-equivalent quota)
    memory: "512Mi"      →  memory.max = 536870912 bytes (hard ceiling → OOM if exceeded)
```

- **CPU request** → influences relative scheduling weight (fairness under contention), and is also what the Kubernetes **scheduler** uses to decide which node has enough *reservable* capacity — but the request itself is **not** enforced by cgroups as a hard floor or ceiling; it's a scheduling-time promise, not a runtime cap.
- **CPU limit** → becomes a hard `cpu.max` quota — actively enforced by the kernel every period, causing throttling if exceeded.
- **Memory request** → used only for scheduling decisions (same as CPU request) — not enforced at runtime by itself.
- **Memory limit** → becomes `memory.max` — a hard ceiling; exceeding it triggers the OOM killer for that cgroup specifically (killing the offending container, not necessarily the whole node).

## Interview question: "How does Kubernetes enforce limits?"

Kubernetes itself enforces nothing directly at runtime — it's an orchestration/scheduling layer. What actually happens: the kubelet translates a Pod's `resources` spec into cgroup v2 control file values (`cpu.max`, `memory.max`, etc.) when the container is created via containerd/runc, and from that point forward, **the Linux kernel itself** — via the CFS scheduler (for CPU quotas) and the memory management/OOM subsystem (for memory ceilings) — is what actually measures and enforces those limits, completely independent of Kubernetes being involved in each individual scheduling tick or memory allocation.
