# Storage — Filesystems, Page Cache I/O, Container Layers, and Kubernetes Volumes

## Filesystem & inode basics

A filesystem organizes raw disk blocks into files and directories. An **inode** is the actual data structure holding a file's metadata (permissions, owner, size, timestamps, and pointers to the actual data blocks on disk) — notably, **the filename itself is not stored in the inode**; it lives in the containing directory's entry, which just maps a name to an inode number.

```
Directory entry:           Inode table:                    Data blocks on disk:
 "app.log" → inode 4821     inode 4821: {size, perms,  ──►   [block 501][block 502][block 503]
                                          owner, timestamps,
                                          pointers to blocks}
```

This separation is *why* a hard link (`ln`, not `ln -s`) can give two different filenames pointing to the exact same inode/data — deleting one filename doesn't delete the underlying data until the inode's link count drops to zero and no process still holds it open.

## Page Cache, Buffered I/O, and Direct I/O

By default, file I/O in Linux goes through the **page cache** (see the Linux Memory README) — this is called **buffered I/O**: reads populate the cache, writes go to cache first and get flushed to disk asynchronously (or on `fsync`).

```
Buffered I/O (default):
  App write() ──► Page Cache (fast, in RAM) ──► (asynchronously) ──► Disk

Direct I/O (O_DIRECT flag):
  App write() ─────────────────────────────────────────────────► Disk directly
                    (bypasses page cache entirely)
```

**Direct I/O** bypasses the page cache entirely — an application (commonly databases like PostgreSQL, which implement their **own** sophisticated buffer/cache management) uses `O_DIRECT` when it wants precise, predictable control over exactly what's cached and when data hits physical disk, rather than trusting the kernel's generic caching heuristics — avoiding "double caching" (once in the app's own buffer pool, again redundantly in the kernel's page cache) and getting more deterministic write-durability guarantees.

## OverlayFS — how container images actually work

Container images are built from **layers**, and OverlayFS is the mechanism that stacks them into a single unified filesystem view without duplicating data.

```
                    ┌───────────────────────────┐
                    │   Container (writable)    │  ← "upper" layer: only THIS
                    │   layer — new/changed     │    layer is writable; changes
                    │   files go here           │    here don't affect the image
                    ├───────────────────────────┤
                    │   Image layer 3 (app code)│  ← read-only
                    ├───────────────────────────┤
                    │   Image layer 2 (deps)    │  ← read-only
                    ├───────────────────────────┤
                    │   Image layer 1 (base OS) │  ← read-only
                    └───────────────────────────┘
                              │
                    OverlayFS merges all of these
                    into ONE unified view the
                    container process actually sees
```

Key behavior: **copy-on-write**. If a container modifies a file that exists in a read-only lower layer, OverlayFS transparently copies that file up into the writable upper layer first, then applies the modification there — the original lower layers are never touched, which is exactly why many containers can share the same underlying image layers on disk (huge storage/pull efficiency) while each still gets to "modify" files independently and safely.

This copy-on-write "upper" layer is also the **container's writable layer/ephemeral storage** — data written here is lost when the container is removed (unless it's backed by a mounted volume instead), which is the direct technical reason containers are meant to be treated as ephemeral/stateless by default.

## Container filesystem in practice

```
Container process's view:  /  (single unified root, via OverlayFS merge)
                           /app     ← from an image layer
                           /etc     ← from a base image layer, possibly modified (COW)
                           /data    ← a MOUNTED volume (bypasses OverlayFS entirely,
                                       backed by real persistent storage instead)
```

Any path explicitly mounted as a **volume** (Kubernetes `volumeMounts`) is **not** part of the OverlayFS union — it's a direct mount of whatever backing storage the volume represents, which is exactly why data written to a mounted volume path survives container restarts/recreation, while data written anywhere else in the container's filesystem does not.

## Persistent Volumes (PV) and CSI

Kubernetes separates the **request for storage** from **how that storage is actually provisioned**:

```
   PersistentVolumeClaim (PVC)          "I need 10Gi of storage, ReadWriteOnce"
              │
              ▼  (bound to)
   PersistentVolume (PV)                "Here's an actual 10Gi volume that satisfies it"
              │
              ▼  (provisioned/attached by)
   CSI (Container Storage Interface) driver
              │
              ▼
   Actual backing storage (EBS volume, NFS share, Ceph, etc.)
```

- A **PVC** is a Pod-author's *request* for storage with certain characteristics (size, access mode) — it doesn't know or care what's underneath.
- A **PV** is the actual piece of storage (can be pre-provisioned by an admin, or dynamically created on-demand by a `StorageClass` + CSI driver when a PVC is created).
- **CSI (Container Storage Interface)** is the standardized plugin interface that lets Kubernetes talk to *any* storage backend (AWS EBS, GCP Persistent Disk, Ceph, NFS, etc.) through a common set of operations (`CreateVolume`, `AttachVolume`, `MountVolume`, ...) — without Kubernetes core needing built-in, backend-specific code for every possible storage vendor. This is exactly what lets a cluster provision an EBS-backed PV on EKS transparently, the moment a PVC referencing an EBS-backed `StorageClass` is created.

Access modes matter for scheduling too — a `ReadWriteOnce` (RWO) volume can only be mounted read-write by a single node at a time, which is why Pods needing that same PVC must be scheduled onto the same node (or the same Pod must terminate/detach before another can attach) — a common StatefulSet scheduling constraint.

## Interview framing

- **"Why is a container's filesystem layered?"** — For storage and distribution efficiency: shared base layers (OS, common dependencies) can be cached and reused across many images/containers without duplication, and OverlayFS's copy-on-write means each running container only needs its own thin writable layer for actual changes, not a full private copy of the entire image.
- **"Why does data disappear when a container restarts, unless it's a mounted volume?"** — Because anything written outside a mounted path lives only in the container's OverlayFS writable ("upper") layer, which is torn down along with the container itself; only explicitly mounted volumes point to storage that outlives the container's lifecycle.
- **"What does CSI actually decouple?"** — It decouples Kubernetes' core code from any specific storage vendor's implementation details, letting new storage backends plug in via a standard interface instead of requiring changes to Kubernetes itself.
