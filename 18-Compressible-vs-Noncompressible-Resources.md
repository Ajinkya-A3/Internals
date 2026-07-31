# Compressible vs Non-Compressible Resources — The Concept That Ties Everything Together

This is the unifying idea behind why CPU limits throttle but memory limits kill — and it's exactly the kind of conceptual framing that signals deeper understanding in an interview, because it explains *why* Kubernetes' two most common resource types are enforced so differently, rather than just reciting that they are.

## The one-question test: "Can this resource wait?"

For any resource, ask: **if a process wants more of this resource than is currently available, can the request simply be delayed until later without anything failing?**

- If **yes** → the resource is **compressible**. Delay is a safe, recoverable response.
- If **no** → the resource is **non-compressible**. Something must fail (or be forcibly freed) right now.

## CPU — Compressible

```
Process wants CPU
       │
       ▼
     Wait   (nothing breaks while waiting — the process is simply
       │      not executing for a while; its state is fully preserved)
       ▼
   Run later
```

The kernel can always say "not right now, try again in the next scheduling period" — the process's memory, open files, and state are completely undisturbed while it waits. Nobody dies; the only cost is **latency** (the work takes longer wall-clock time to complete, but it *does* complete correctly). This is precisely why cgroup CPU enforcement (`cpu.max`) is implemented as **throttling** (delay and resume) rather than killing — CPU's compressibility makes delay a perfectly safe enforcement strategy.

## Memory — Non-Compressible

```
Process needs 2 GB
Available: only 1 GB
       │
       ▼
Kernel CANNOT say "wait, try again later" — the process is trying
to actively use that memory RIGHT NOW (write data into it, etc.)
       │
       ▼
  Something must give immediately:
  reclaim memory from elsewhere (evict cache), or...
       │
       ▼
      OOM Killer
```

Unlike a CPU time-slice, a memory allocation can't be "deferred a few milliseconds and retried" in a way that preserves correctness for an actively-running process expecting that memory to be there. Once genuinely exhausted (after cache reclaim options are used up), the kernel's only remaining lever is to **forcibly free memory by terminating a process** — which is why memory limit violations manifest as sudden, hard kills (`OOMKilled`) rather than graceful slowdowns.

## Other resources, using the same test

| Resource | Can it wait? | Compressible? | What happens when exhausted |
|---|---|---|---|
| **CPU** | Yes — time can always be deferred | Compressible | Throttling (delay, resume later) |
| **Memory** | No — an active allocation can't be paused | Non-compressible | OOM kill |
| **Disk I/O** | Mostly yes — an I/O request can be queued and delayed (though very slow I/O can cascade into application-level timeouts, which is a *secondary* failure, not the kernel's direct doing) | Mostly compressible | Throttled/queued via `io.max`, though sustained saturation can still cause app-level failures indirectly |
| **Network bandwidth** | Yes — packets can be queued or dropped and, for TCP, retransmitted; the transport layer is explicitly designed to tolerate delay | Compressible | Queuing, traffic shaping, or packet drops (handled gracefully by TCP's congestion control — see the TCP Internals README) |
| **Ephemeral storage** (disk *space*, not I/O throughput) | No — if a Pod's writable layer or emptyDir genuinely runs out of disk space, there's no "wait and get more space later" | Non-compressible | Kubernetes evicts the Pod (or the write simply fails at the filesystem level) once ephemeral storage usage exceeds its limit |

Notice the pattern: resources measured in **time/rate** (CPU, network bandwidth, disk I/O throughput) tend to be compressible — you can always slow down the rate and catch up later. Resources measured in **fixed capacity/space** (memory, disk space) tend to be non-compressible — there's a hard ceiling with no "slower is fine" option once it's hit.

## Interview one-liners

- **"What makes a resource compressible?"** — The ability to delay access to it without causing failure — the consumer can simply wait and try again later, fully preserving correctness.
- **"Why does Kubernetes throttle CPU but kill on memory?"** — Because the enforcement mechanism must match the resource's nature: CPU's compressibility makes "pause and resume" a safe strategy (only latency suffers), while memory's non-compressibility means the kernel has no safe "pause" option once exhausted — it must free memory immediately, which means killing a process.
- **"Is disk I/O compressible or not?"** — Mostly compressible (I/O can be queued/throttled via `io.max` without immediate failure), but disk *space* itself is non-compressible — you can't "wait" for more space to magically appear the way you can wait for more I/O bandwidth to free up.
