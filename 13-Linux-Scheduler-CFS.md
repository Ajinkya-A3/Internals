# The Linux Scheduler (CFS) — Fairness, Virtual Runtime, and Throttling

## The goal CFS was designed for

CFS (Completely Fair Scheduler), Linux's default scheduler for normal (non-realtime) tasks, tries to give every runnable task a **fair share of CPU time proportional to its weight**, over time — as if there were an idealized processor that could run all runnable tasks simultaneously at a fraction of full speed each. It's an approximation of that ideal using time-slicing on real, finite cores.

## Virtual runtime (vruntime) — the core mechanism

Every runnable task accumulates a **vruntime** — a measure of how much CPU time it has consumed, weighted by its priority/weight. CFS always picks the task with the **lowest vruntime** to run next (it's "owed" the most CPU relative to how much it's already gotten).

```
Run queue, ordered by vruntime (a red-black tree internally):

Task A: vruntime = 100  ◄── lowest vruntime → CFS picks this one next
Task B: vruntime = 150
Task C: vruntime = 300

After Task A runs for a bit, its vruntime increases:

Task B: vruntime = 150  ◄── now the lowest → CFS picks this one next
Task A: vruntime = 210
Task C: vruntime = 300
```

This is what makes it "completely fair" — over time, every task's vruntime converges, meaning every task gets a share of the CPU proportional to its weight, and no single task can be starved or hog the CPU indefinitely.

## CPU shares / weight

A task's **niceness** (or a cgroup's `cpu.weight`, formerly `cpu.shares`) determines how fast its vruntime accumulates relative to others — a higher-priority/higher-weight task's vruntime grows *slower* per unit of actual CPU time consumed, so it gets picked more often.

```
Higher weight (e.g., nice = -10, or cpu.weight = 200):
   vruntime grows SLOWLY per real second of CPU used → gets scheduled MORE often

Lower weight (e.g., nice = 10, or cpu.weight = 50):
   vruntime grows QUICKLY per real second of CPU used → gets scheduled LESS often
```

Importantly: **shares/weight are only relative and only matter under contention.** If Task A has 2x the weight of Task B, and both want the CPU, A gets ~2x the CPU time. But if the CPU is idle otherwise, either task can use as much as it wants regardless of weight — shares don't cap usage, they only govern proportional access *when there's competition*.

## Time slices

Rather than fixed time slices (old O(1) scheduler-era approach), CFS computes a dynamic slice length based on a target latency (how long, ideally, before every runnable task gets at least one turn) divided among the number of runnable tasks — so time slices shrink as more tasks compete, and grow when fewer are competing, keeping a roughly constant "everyone gets a turn within X ms" guarantee.

## CPU affinity

CPU affinity pins a process/thread to run only on a specific subset of CPU cores (`taskset`/`sched_setaffinity`, or Kubernetes' `cpuset` cgroup controller when using the **static CPU Manager policy**).

```
Without affinity: task can run on ANY of cores 0-7, scheduler picks freely
With affinity pinned to cores 2-3: task can ONLY ever run on cores 2 or 3
```

Why this matters in practice:
- **Cache locality** — a task that keeps migrating between cores loses its warm L1/L2 cache each time; pinning keeps it on the same core(s), improving cache hit rates for latency-sensitive workloads.
- **NUMA locality** (see below) — pinning to cores on the same NUMA node as the task's memory avoids expensive cross-node memory access.
- Kubernetes' CPU Manager `static` policy exclusively pins whole cores to Guaranteed-QoS pods requesting integer CPUs — trading scheduling flexibility for predictable, cache-friendly performance (common for latency-critical workloads).

## NUMA (basic)

On multi-socket servers, memory is physically attached to specific CPU sockets — a CPU can access its **local** memory fast, but accessing another socket's memory ("remote" memory) is slower, since it has to traverse an interconnect between sockets.

```
┌──────────────────┐        interconnect         ┌──────────────────┐
│   NUMA Node 0    │◄───────────────────────────►│   NUMA Node 1    │
│  CPU 0-7         │                             │  CPU 8-15        │
│  Local RAM bank A│                             │  Local RAM bank B│
└──────────────────┘                             └──────────────────┘

CPU 0 accessing RAM bank A → fast (local)
CPU 0 accessing RAM bank B → slower (remote, crosses interconnect)
```

For performance-sensitive workloads, both CPU *and* memory should ideally be allocated from the **same NUMA node** — this is what Kubernetes' Topology Manager coordinates (aligning CPU Manager's core pinning with memory/device locality) for latency-critical pods.

## Interview Q&A

**"How does Linux decide which process runs?"**
CFS maintains a red-black tree of runnable tasks ordered by vruntime, and always dispatches the task with the lowest vruntime — the one that has proportionally received the least CPU time so far relative to its weight — approximating an idealized "everyone gets a fair, weighted share simultaneously" processor.

**"Why can one process get less CPU?"**
Either it has a lower weight/niceness (so its vruntime accumulates faster per unit of real time, making it "owe less" and get scheduled less often), or it's cgroup-constrained by a CPU quota/limit that caps its total allotted CPU-time per period regardless of what the scheduler would otherwise give it.

**"Why does CPU throttling happen?"**
Throttling is separate from CFS's fairness scheduling — it's the cgroup **quota enforcement** mechanism (`cpu.max`/`cpu.cfs_quota_us`): if a container has consumed its entire allotted CPU-time budget for the current period (e.g., 100ms of every 100ms period, for a 1-core limit), the kernel simply refuses to schedule it again until the next period starts — even if CPU cores are sitting idle. This is a very common, often misunderstood cause of latency spikes in Kubernetes: a pod can be throttled (and appear "slow") **even though the node has spare CPU capacity**, purely because its own cgroup quota for the current period is exhausted. (Full mechanics in the cgroups v2 and CPU Requests/Limits READMEs.)
