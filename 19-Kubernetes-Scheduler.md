# The Kubernetes Scheduler — Filter, Score, Bind, and Why a Pod Doesn't Get Scheduled

## The three-phase pipeline

```
   Unscheduled Pod appears (in the scheduler's watch)
              │
              ▼
     ┌─────────────────┐
     │   FILTER phase  │   "Which nodes are even CAPABLE of running this pod?"
     └─────────────────┘   (a hard yes/no per node — eliminates ineligible nodes)
              │
              ▼
     ┌─────────────────┐
     │   SCORE phase   │   "Of the nodes that passed filtering, which is the BEST fit?"
     └─────────────────┘   (each node gets a numeric score from multiple plugins)
              │
              ▼
     ┌─────────────────┐
     │    BIND phase   │   Scheduler writes the decision back to the API server —
     └─────────────────┘   the Pod's spec.nodeName is set, kubelet on that node takes over
```

## Filter phase — hard constraints, eliminate infeasible nodes

Every node is checked against a series of predicates; **failing any one** removes that node from consideration entirely:

- **Node Resources** — does the node have enough *allocatable* CPU/memory left (after subtracting other pods' *requests*, not actual usage) to satisfy this Pod's requests?
- **Node Affinity / Anti-affinity** — does the node's labels satisfy the Pod's `nodeAffinity` rules (e.g., "must be on a node with `disktype=ssd`")?
- **Taints and Tolerations** — does the node have a taint the Pod hasn't tolerated? (See below — this is a common source of confusion, phrased backwards from affinity.)
- **Volume constraints** — can the required PersistentVolumes actually be attached/mounted on this node (e.g., topology-aware provisioning, node in the right zone for an EBS volume)?
- **Port conflicts** — does the Pod request a hostPort already in use on that node?

```
10 nodes in cluster
     │
     ▼ Filter phase
7 nodes eliminated (insufficient CPU, wrong zone, untolerated taint, etc.)
     │
     ▼
3 nodes remain — feasible candidates
```

## Score phase — soft preferences, rank the survivors

Only nodes that passed filtering are scored (0-100 per plugin, then combined via weights) to pick the *best* among feasible options — this is where "preferences" (as opposed to hard requirements) live:

- **Least/Most Requested Priority** — prefer nodes with more/less remaining capacity (Kubernetes defaults toward spreading load, generally preferring less-utilized nodes).
- **Node Affinity preferred rules** (`preferredDuringSchedulingIgnoredDuringExecution`) — soft preference, adds to the score but doesn't eliminate a node if unmet.
- **Pod Topology Spread Constraints** — score nodes to spread Pods evenly across a topology domain (zone, node, rack) according to `topologySpreadConstraints`, reducing the chance an entire zone outage takes down all replicas of a Deployment.
- **Image locality** — mildly prefer nodes that already have the container image cached, avoiding a pull.

```
3 feasible nodes → scored:
  Node A: 72
  Node B: 91  ◄── highest score, WINS
  Node C: 65
```

## Bind phase

The scheduler commits its decision by writing the chosen node into the Pod's `spec.nodeName` via the API server. From that point, it's the **kubelet** on that specific node (not the scheduler) that actually pulls the image and starts the container (via containerd → runc → cgroups, as covered in the cgroups v2 README).

## Taints and Tolerations (the "push away" mechanism)

```
Node has a TAINT:  key=value:NoSchedule

Pod WITHOUT a matching toleration  → REJECTED from this node (Filter phase failure)
Pod WITH a matching toleration     → ALLOWED to be scheduled here (not forced, just permitted)
```

Taints are a property of the **node**, saying "don't schedule here unless you specifically tolerate this." Tolerations are a property of the **Pod**, saying "I'm okay running on nodes with this specific taint." This is the inverse of affinity (which is the Pod *pulling itself toward* certain nodes) — taints are nodes *pushing Pods away* unless explicitly permitted.

## Node Affinity (the "pull toward" mechanism)

```
requiredDuringSchedulingIgnoredDuringExecution:
  → HARD requirement, filtered — Pod simply won't be scheduled if unmet

preferredDuringSchedulingIgnoredDuringExecution:
  → SOFT preference, scored — influences ranking but doesn't block scheduling if unmet
```

## Topology Spread Constraints

Ensures Pods of the same workload are distributed evenly across a chosen topology key (e.g., `topology.kubernetes.io/zone`, or `kubernetes.io/hostname`), with a configurable `maxSkew` (the maximum allowed difference in Pod count between the most- and least-loaded domains) — directly relevant to availability: without this, all replicas of a Deployment could accidentally land in a single AZ, defeating the purpose of running multiple replicas at all.

## Priority Classes and Preemption

Pods can be assigned a `PriorityClass` (a numeric priority value). When a higher-priority Pod can't be scheduled because the cluster is full, the scheduler can **preempt** (evict) lower-priority Pods on a node to make room — the evicted Pods go back to Pending and get rescheduled elsewhere (or stay Pending if nothing fits).

```
Cluster is full. A critical high-priority Pod needs scheduling.
              │
              ▼
Scheduler identifies lower-priority Pod(s) on some node whose eviction
would free enough resources for the high-priority Pod
              │
              ▼
Lower-priority Pod(s) evicted (graceful termination) ──► high-priority Pod scheduled
```

## Interview question: "Why wasn't this pod scheduled?"

Walk through the pipeline as a diagnostic checklist (this is exactly how you'd debug it live with `kubectl describe pod` and `kubectl get events`):

1. **Check Filter-phase failures first** (these show up directly in Pod events, e.g., "0/5 nodes are available: 3 Insufficient cpu, 2 node(s) had taint {...} that the pod didn't tolerate").
   - Insufficient CPU/memory requests available on any node?
   - A required node affinity rule nobody satisfies?
   - A taint nobody tolerates?
   - A PersistentVolume that can't be bound/attached where the Pod could otherwise run?
2. **If it passed filtering but is still Pending**, check whether it's a scheduling *queue* backlog, or whether a **PodDisruptionBudget** or resource quota (namespace-level `ResourceQuota`) is blocking it indirectly.
3. **Check priority/preemption** — is a lower-priority pod that *should* be preempted actually protected by a PodDisruptionBudget, preventing preemption from completing?

In short: "not scheduled" almost always traces back to a Filter-phase rejection across every node in the cluster — the fix is either freeing capacity, relaxing an affinity/taint requirement, or adding priority/preemption to make room.
