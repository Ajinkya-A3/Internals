# Processes and Threads — Lifecycle, Context Switching, PID Namespaces, Scheduling

## Process vs Thread

A **process** is an independent unit of execution with its own memory address space (code, data, heap, stack), file descriptors, and OS resources. A **thread** is a unit of execution *within* a process that shares that process's memory address space and file descriptors with all other threads in the same process, but has its own stack, registers, and program counter.

```
Process (own address space)
┌─────────────────────────────────────────────┐
│  Code  |  Data  |  Heap  |  Open File Descs │
│                                             │
│  Thread 1        Thread 2        Thread 3   │
│  ┌────────┐      ┌────────┐      ┌────────┐ │
│  │ Stack  │      │ Stack  │      │ Stack  │ │
│  │ Regs   │      │ Regs   │      │ Regs   │ │
│  │ PC     │      │ PC     │      │ PC     │ │
│  └────────┘      └────────┘      └────────┘ │
│     (all threads share the SAME heap/data)  │
└─────────────────────────────────────────────┘
```

Consequence: creating a thread is much cheaper than creating a process (no new address space to set up), and threads can communicate via shared memory directly — but that shared memory also means threads must coordinate (locks/mutexes) to avoid data races, whereas processes are naturally isolated and must use explicit IPC (pipes, sockets, shared memory segments) to communicate.

## Process lifecycle

```
        fork()/clone()
              │
              ▼
          [ NEW ]
              │
              ▼
         [ READY ]  ◄─────────────────┐
              │                       │
     scheduler picks it       preempted / time
              │                slice expired
              ▼                       │
         [ RUNNING ] ─────────────────┘
              │
    ┌─────────┼──────────────┐
    │         │              │
 blocks on   exits       receives signal
 I/O/lock     │              │
    │         ▼              ▼
    ▼    [ TERMINATED ]  [ handled or killed ]
[ WAITING/BLOCKED ]
    │
 I/O completes
    │
    ▼
[ READY ] (back in the queue)
```

- **NEW** — process created (via `fork()` or `clone()`) but not yet scheduled.
- **READY** — waiting in the scheduler's run queue, able to run but not currently on a CPU.
- **RUNNING** — actually executing on a CPU core right now.
- **WAITING/BLOCKED** — waiting on something external (disk I/O, network, a lock, a `sleep()`) — deliberately *not* consuming CPU while blocked.
- **TERMINATED (zombie until reaped)** — process has exited; its exit status is held until the parent calls `wait()` to reap it (an unreaped process is a "zombie" — a common source of process-table leaks if a parent never reaps its children).

## Context switching

A context switch happens when the CPU stops running one process/thread and starts running another — the kernel must save the current execution state (registers, program counter, stack pointer) and restore the next task's saved state.

```
CPU running Process A
       │
       ▼
  Timer interrupt / blocking call / higher-priority task ready
       │
       ▼
  Kernel saves Process A's CPU state (registers, PC) → its PCB (Process Control Block)
       │
       ▼
  Kernel loads Process B's saved state from its PCB
       │
       ▼
  CPU resumes running Process B
```

Context switches aren't free — they cost CPU cycles (saving/restoring state) and also often **flush CPU caches (L1/L2)** since the new process's working set is different, meaning subsequent memory accesses are slower until the cache "warms up" again. This is exactly why excessive context switching (from too many runnable threads competing for too few cores) shows up as real, measurable overhead in production — high `context switches/sec` in `vmstat`/`pidstat` output is a real performance smell.

Switching threads *within the same process* is cheaper than switching between different processes, because the address space (page tables) doesn't need to change — only the per-thread state (stack, registers) does.

## PID Namespace

A PID namespace gives a process tree its own **isolated view of process IDs**, independent of PIDs in other namespaces.

```
Host PID namespace:
  PID 1 (systemd/init)
   └── PID 4821 (containerd-shim)
         └── PID 4822 (the containerized process, but seen as PID 1 INSIDE its own PID namespace)

Inside the container's PID namespace:
  PID 1 → this same process, but numbered 1 in this namespace's view
```

This is the exact mechanism behind "why does a container think PID 1 is itself?" — every PID namespace has its own PID 1, and the kernel maintains a mapping between the PID as seen inside the namespace and the "real" PID as seen from the host (or parent namespace). PID 1 also has special semantics in Linux — it's responsible for reaping orphaned/zombie processes and doesn't get default-killed the same way ordinary processes do by certain signals, which is why misconfigured containers running an app directly as PID 1 (instead of a proper init like `tini`) can end up with zombie process buildup if the app doesn't reap children itself.

## Scheduling (brief intro — full depth in the CFS README)

The kernel's scheduler decides, among all **READY** processes/threads, which one gets the CPU next and for how long. This decision considers priority, fairness, and (in modern Linux) the CFS algorithm's virtual runtime accounting.

## Interview question: "Why can one pod use multiple CPU cores?"

A Kubernetes Pod (really, its containers) is just a Linux **process** (or several processes/threads) constrained by cgroups — nothing about a process fundamentally limits it to one core. If the process is multi-threaded (or forks multiple worker processes), the Linux scheduler is free to run those threads/processes on **different CPU cores simultaneously** (true parallelism), as long as:

1. The container isn't restricted by `cpuset` pinning to a single core.
2. Its `cpu.max` cgroup quota (if a CPU *limit* is set) allows enough total CPU-time-per-period across cores — a limit of "1 CPU" as a cgroup quota can actually be spent as, say, 0.5 core on two cores simultaneously for half the period, not necessarily one whole core the whole period; the kernel just enforces a total time budget, not which specific core(s) it runs on.

So a pod with `resources.limits.cpu: "2"` and a multi-threaded app can genuinely use 2 full cores' worth of CPU time simultaneously, because the cgroup quota (2 cores' worth of CPU-time per period) is a *budget*, and the CFS scheduler is free to spread that budget across any number of physical cores available to it.
