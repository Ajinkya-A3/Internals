# Routing — How Packets Find Their Way Across Networks

## The core job of a router

A router's only job at Layer 3: look at a packet's **destination IP**, consult its **routing table**, and forward the packet out the correct interface toward the next hop — one hop at a time, with no end-to-end knowledge of the full path. This is fundamentally different from a switch, which forwards based on MAC addresses *within* a single broadcast domain.

## Routing table & longest prefix match

A routing table is a list of network prefixes and which interface/next-hop to send matching traffic to.

```
Destination           Next Hop         Interface
0.0.0.0/0              192.168.1.1      eth0        (default route — catch-all)
10.0.0.0/8             10.0.0.1         eth1
10.1.2.0/24            10.1.2.1         eth2
```

When a packet destined for `10.1.2.55` arrives, the router matches against **every** entry, and picks the **most specific** one — this is **longest prefix match**:

```
10.1.2.55 matches:
  0.0.0.0/0     (0 bits matched  — least specific)
  10.0.0.0/8    (8 bits matched)
  10.1.2.0/24   (24 bits matched — MOST specific → this one wins)
```

Longest prefix match is why you can have a broad default route *and* specific overrides for particular subnets, and the specific one always takes priority regardless of table order.

## Default gateway (recap in routing context)

The `0.0.0.0/0` entry is literally the default gateway concept expressed as a routing table rule: "if nothing more specific matches, send it here." Every host and router typically has one.

## Static vs Dynamic routing

| | Static Routing | Dynamic Routing |
|---|---|---|
| How routes are added | Manually configured by an admin | Automatically learned via a routing protocol exchanging info with neighbors |
| Adapts to failures? | No — a manual change is required | Yes — protocol detects topology changes and recalculates paths |
| Best for | Small, stable, simple networks (e.g., a single office, a VPC with few subnets) | Large, complex, or frequently changing networks (ISPs, multi-region infrastructure) |
| Overhead | None (no protocol traffic) | Some CPU/bandwidth spent on route advertisements and recalculation |

## BGP (Border Gateway Protocol) — high level

BGP is the routing protocol that runs the **internet's backbone** — it's how independent networks (**Autonomous Systems**, each with its own AS number) exchange reachability information with each other.

```
AS 100 (your ISP) ──BGP peering──► AS 200 (Tier 1 provider) ──BGP peering──► AS 300 (Google)

Each AS announces: "I can reach these IP prefixes, here's the AS-path to get there"
Neighboring AS's choose the best path based on policy (not just shortest path —
BGP is highly policy-driven: cost, business relationships, path length, etc.)
```

Key properties:
- BGP is a **path-vector protocol** — routes carry the list of AS numbers a packet would traverse, and loops are avoided by simply rejecting any route whose AS-path already contains your own AS.
- Route selection is heavily influenced by **business/peering agreements**, not purely technical shortest-path — an ISP might prefer a longer path because it's a settlement-free peering link versus a paid transit link.
- BGP is why the internet has no single point of control — it's a mesh of independently operated, independently trusted networks constantly exchanging "here's what I can reach" announcements.
- This is directly relevant to real production incidents: a **BGP route leak or hijack** (a network incorrectly announcing routes it shouldn't) can cause traffic for a legitimate destination to be misdirected globally — a well-known class of internet-wide outage.

## OSPF (Open Shortest Path First) — basic

OSPF is a **link-state, interior gateway protocol** — used *within* a single organization's network (an Autonomous System), not between organizations like BGP.

```
Each router floods "link-state advertisements" describing its directly connected links
                              │
                              ▼
Every router builds an identical map of the entire network topology
                              │
                              ▼
Each router independently runs Dijkstra's shortest-path algorithm
                              │
                              ▼
Result: the actual shortest path (by configured cost/metric) to every destination
```

Contrast with BGP:

| | OSPF | BGP |
|---|---|---|
| Scope | Inside one organization (interior) | Between organizations (exterior) |
| Metric | Cost based on link speed/bandwidth | Policy-driven (AS-path, business rules), not pure shortest-path |
| Convergence | Fast (floods full topology, everyone computes shortest path) | Slower, more conservative (internet-scale stability matters more than speed) |
| Algorithm | Dijkstra (shortest path first) | Path-vector, best-path selection with many tunable attributes |

## Interview one-liners

- **"What is longest prefix match and why does it matter?"** — When multiple routing table entries could match a destination IP, the router always picks the entry with the most specific (longest) prefix, letting specific overrides coexist with a broad default route.
- **"Why does the internet use BGP instead of something like OSPF everywhere?"** — OSPF assumes a single administrative trust domain computing pure shortest paths; the internet is thousands of independently operated networks with business relationships and policies that override "shortest path," which is exactly what BGP's path-vector, policy-driven model is designed for.
- **"What's a default route?"** — The `0.0.0.0/0` catch-all routing table entry used when no more specific route matches the destination.
