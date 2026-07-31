# Linux Memory — Virtual Memory, Page Cache, Swap, and the OOM Killer

## The layered picture

```
   Virtual Memory
        │  (every process gets its own private address space,
        │   mapped by the kernel to physical RAM via page tables)
        ▼
   Page Cache
        │  (kernel opportunistically caches file data in
        │   otherwise-unused RAM, for fast re-reads)
        ▼
   Anonymous Memory
        │  (memory NOT backed by a file — heap, stack — the
        │   actual "private" working data of a process)
        ▼
   Swap
        │  (when physical RAM is under real pressure, inactive
        │   anonymous pages can be written out to disk to free RAM)
        ▼
   OOM Killer
        │  (last resort: if memory is genuinely exhausted and
        │   nothing can be reclaimed/swapped fast enough, kill
        │   a process to free memory immediately)
        ▼
   Huge Pages / THP
           (larger page sizes to reduce page-table management
            overhead for memory-intensive workloads)
```

## Virtual Memory

Every process sees its own private, contiguous **virtual address space** — it never directly addresses physical RAM. The kernel's **page tables** translate virtual addresses to physical addresses (or to "not currently in RAM, fault and fetch" markers) transparently.

```
Process A's virtual address 0x4000  ──► Page table ──► Physical RAM address 0x7F2A1000
Process B's virtual address 0x4000  ──► Page table ──► Physical RAM address 0x9C3B4000
   (same virtual address, different processes, mapped to completely different physical RAM)
```

This is what makes process isolation possible at the memory level — no process can accidentally (or maliciously, without kernel cooperation) read another process's physical memory, since it only ever deals in its own virtual addresses.

## Page Cache — "why does Linux use free memory as cache?"

Whenever a process reads a file from disk, the kernel keeps a copy of those disk blocks in RAM (the **page cache**) — so a subsequent read of the same data can be served instantly from RAM instead of hitting the (much slower) disk again.

```
First read of file.txt:
  Disk ──(slow)──► Page Cache ──► Process

Second read of the same file.txt:
  Page Cache ──(fast, no disk I/O)──► Process
```

Critically, this cache uses **memory that would otherwise just sit idle** — Linux treats "free" memory as wasted opportunity, so it aggressively fills unused RAM with cache. This is exactly why `free -m` on a healthy Linux box often shows very little "free" memory and a large "cache/buffer" figure — that's not a problem, it's the kernel being efficient. **Crucially, page cache is reclaimable on demand** — if an application needs that RAM for a real allocation, the kernel simply evicts cache pages (no data loss, since it's just a cached copy of something still safely on disk) to make room, instantly.

This is precisely why "available" memory (what's actually usable, including reclaimable cache) is a very different, more useful number than "free" memory (literally-idle-right-now memory) — and why container memory limits/OOM decisions are based on working-set-aware metrics (see the Workload Optimization README) rather than naive free-memory numbers.

## Anonymous Memory

Memory **not** backed by a file on disk — this is a process's heap, stack, and any anonymous `mmap` allocations. Unlike page cache (which can simply be dropped and re-read from its file later), anonymous memory holds live, non-recoverable program state — if it needs to be reclaimed under pressure, it can only go to **swap**, not simply be discarded.

## Swap

When physical RAM is genuinely under pressure and page cache reclaim alone isn't enough, the kernel can write out inactive anonymous pages to a swap device (disk), freeing RAM for more urgent use — at the cost of a page fault (and a slow disk read) if that swapped-out data is needed again later.

```
RAM under pressure
       │
       ▼
Kernel picks inactive anonymous pages ──► writes them to SWAP (disk) ──► frees RAM
       │
       ▼ (later, if that memory is accessed again)
Page fault ──► read back from swap ──► slow, but data is recovered correctly
```

**Kubernetes traditionally disables swap on nodes entirely** (and historically refused to even start the kubelet if swap was enabled, though recent versions have added limited, opt-in swap support) — because swap makes memory-limit enforcement unpredictable: a container hitting its `memory.max` might get silently, unpredictably slow (thrashing to disk) instead of being cleanly OOM-killed, which undermines the whole point of a hard, deterministic resource ceiling in an orchestrated environment.

## OOM Killer

When memory is truly exhausted — no more page cache to reclaim, swap exhausted or disabled — the kernel's OOM killer selects a process to terminate immediately, based on a heuristic "badness" score (roughly: memory footprint, adjustable via `oom_score_adj`) to free memory as fast as possible. In a cgroup-constrained container context, this operates **per-cgroup** — a container exceeding its own `memory.max` gets its own process(es) killed, without needing to touch the rest of the node (unless the *node itself* runs out of memory globally, triggering a system-wide OOM kill that can select any process, not just ones in the offending cgroup).

## Huge Pages and THP (Transparent Huge Pages)

Standard pages are typically 4KB — for memory-intensive workloads (databases, JVMs with large heaps), managing millions of tiny page-table entries adds real CPU overhead (page table walks, TLB misses). **Huge Pages** (e.g., 2MB or 1GB pages) reduce the number of page-table entries needed for the same amount of memory, cutting that overhead.

- **Explicit Huge Pages** — reserved upfront by an administrator/orchestrator, guaranteed available, requires the application to explicitly use them.
- **THP (Transparent Huge Pages)** — the kernel automatically tries to use huge pages transparently, without application changes — convenient, but can occasionally cause latency spikes (e.g., during huge-page compaction/defragmentation under memory pressure), which is why some latency-sensitive database workloads explicitly **disable** THP and manage huge pages manually instead.

## Interview Q&A

**"Why does Linux use free memory as cache?"**
Because unused RAM provides zero benefit sitting idle, while using it to cache recently-read file data can make subsequent reads dramatically faster — and since page cache is instantly reclaimable the moment an application needs that RAM for a real allocation, there's no real downside to filling "free" memory with cache opportunistically.

**"What's the practical difference between page cache and anonymous memory, in terms of what happens under pressure?"**
Page cache can simply be dropped (it's just a copy of data still safely on disk, re-readable later) — cheap, safe, instant reclaim. Anonymous memory holds unique, non-recoverable program state, so reclaiming it requires either swapping it to disk (slow, and disabled by default in most Kubernetes clusters) or, if that's not viable, invoking the OOM killer.
