# Linux Namespaces — The Isolation Half of "What Is a Container?"

## The core idea

A container is not a special kernel object — it's just an ordinary Linux process that has been placed inside a set of **namespaces** (which control *what it can see*) and **cgroups** (which control *what it can use*, see the cgroups v2 README). Namespaces are entirely about **isolated views** of shared kernel resources — nothing is physically duplicated, the kernel just presents a filtered/private view to processes inside a given namespace.

```
                     Linux Kernel (single instance, shared by everyone)
                                    │
        ┌───────────────────────────┼─────────────────────────────┐
        │                           │                             │
  PID namespace A              PID namespace B              PID namespace C
  (Container 1 sees             (Container 2 sees             (Host sees
   only ITS OWN                  only ITS OWN                  EVERY process,
   process tree,                 process tree,                 including both
   starting at PID 1)            starting at PID 1)             containers')
```

## The six namespace types most relevant here

| Namespace | Isolates | Practical effect |
|---|---|---|
| **PID** | Process IDs | Each namespace has its own PID 1 and process tree — processes in one PID namespace can't see or signal processes in another. |
| **NET** | Network stack | Each namespace gets its own network interfaces, IP addresses, routing tables, and port space — this is *why* two containers can both bind to port 8080 without conflicting, and it's the basis of every Kubernetes Pod's network identity. |
| **IPC** | Inter-process communication primitives | Isolates System V IPC objects and POSIX message queues (shared memory segments, semaphores) — a process can't accidentally attach to another namespace's shared memory segment. |
| **UTS** | Hostname and NIS domain name | Lets a container have its own hostname (`hostname` command inside a container returns something different from the host), independent of the actual host machine's hostname. |
| **Mount** | Filesystem mount points | Each namespace can have a completely different view of the filesystem tree (different root, different mounted volumes) — this is the basis of a container's isolated filesystem (its own `/`, unaware of the host's actual filesystem layout, aside from explicitly bind-mounted volumes). |
| **User** | User and group ID mappings | Lets a process be "root" (UID 0) *inside* its own namespace while actually mapping to an unprivileged UID on the host — a major security hardening technique (rootless containers), since a "root" container process gains no real root privileges on the host if it somehow escapes. |

## How a Pod's network namespace actually works (worth internalizing)

```
Host network namespace
   │
   └── veth pair (virtual Ethernet cable) connects host to:
          │
          Pod's network namespace
             (has its own eth0, its own IP, its own routing table)
```

This is exactly why every container **in the same Kubernetes Pod shares one network namespace** (they share `localhost`, ports, and the same IP) — a Pod, by definition, is a group of containers sharing a single network (and typically IPC) namespace, while each container still gets its own PID and mount namespaces. This is the real technical answer to "what is a Pod" beyond the marketing description — it's a shared network/IPC namespace wrapping one or more separately PID/mount-namespaced containers.

## Interview question: "Why do containers think PID 1 is themselves?"

Because of **PID namespace isolation**: when a container is created, the kernel spins up a new PID namespace for it, and the very first process launched inside that new namespace is automatically assigned **PID 1 within that namespace** — regardless of what its "real" PID is from the host's (or parent namespace's) point of view. The kernel maintains a translation: the exact same underlying process can simultaneously be, say, PID 4822 as seen from the host's PID namespace, and PID 1 as seen from inside its own container's PID namespace — both are true at once, just relative to which namespace is doing the looking.

This also explains a subtle but important operational detail: **PID 1 has special kernel behavior** — it's expected to reap zombie/orphaned child processes, and some signals behave differently toward PID 1 (e.g., default signal dispositions can be ignored for PID 1 unless the process explicitly installs a handler). This is exactly why running an application directly as PID 1 inside a container — without a minimal init process like `tini` or `dumb-init` in front of it — can lead to zombie process accumulation or improper signal handling (e.g., the app never receiving `SIGTERM` properly on `kubectl delete pod`, forcing Kubernetes to eventually send `SIGKILL` after the grace period expires).

## Namespaces vs cgroups — the one-sentence distinction to nail in an interview

**Namespaces control what a process can *see*** (its own PIDs, network, filesystem, hostname); **cgroups control what a process can *use*** (CPU time, memory, I/O bandwidth). A container is the combination of both: an isolated *view* of the system, constrained by an enforced *resource budget*.
