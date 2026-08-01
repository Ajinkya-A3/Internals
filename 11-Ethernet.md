# Ethernet — MAC Addresses, Frames, Switches, and Broadcast/Collision Domains

## MAC Address

A MAC (Media Access Control) address is a 48-bit hardware address burned into (or assigned to) a network interface, written as 6 hex octets:

```
AA:BB:CC:DD:EE:FF
└──┬──┘  └──┬──┘
 Vendor    Unique device
 (OUI)     identifier
```

Unlike IP addresses (logical, can change based on network location), MAC addresses are meant to be globally unique and tied to the physical hardware — this is why ARP exists at all: to bridge the logical (IP) world to the physical (MAC) world.

## Ethernet Frame structure

```
┌───────────┬───────────┬──────┬────────────────────┬─────┐
│ Dest MAC  │ Src MAC   │ Type │       Payload        │ FCS │
│ (6 bytes) │ (6 bytes) │(2B)  │  (up to 1500 bytes)  │(4B) │
└───────────┴───────────┴──────┴────────────────────┴─────┘
```

- **Dest/Src MAC** — who this frame is from and to, at Layer 2.
- **Type/Length** — what's encapsulated inside (e.g., `0x0800` = IPv4, `0x86DD` = IPv6, `0x0806` = ARP).
- **Payload** — the actual data (e.g., an IP packet), up to the MTU (1500 bytes standard, "jumbo frames" go up to ~9000).
- **FCS (Frame Check Sequence)** — a CRC checksum trailer used to detect transmission errors; a corrupted frame is simply dropped by the receiving NIC.

## Switch vs Hub

| | Hub (legacy, essentially obsolete) | Switch |
|---|---|---|
| Operates at | Layer 1 (pure electrical repeater) | Layer 2 (reads MAC addresses) |
| Behavior | Repeats every incoming signal out **every** other port | Learns which MAC is on which port, forwards **only** to the correct port |
| Collision domain | All ports share ONE collision domain | Each port is its own collision domain |
| Efficiency | Poor — floods everything, causes lots of collisions | Efficient — intelligent, targeted forwarding |

```
HUB:                                    SWITCH:
  A ─┐                                    A ─┐
  B ─┼─ (everyone hears everything) ─┐    B ─┼─ (switch learns: MAC-A on port1,
  C ─┘                               │    C ─┘   MAC-C on port3 — forwards
  D ─────────────────────────────────┘    D ─────only to the right port)
```

A switch builds a **MAC address table** by observing the source MAC of every incoming frame and remembering which port it arrived on — so future frames destined for that MAC only get forwarded out that one specific port, instead of flooded everywhere (flooding only happens for genuinely unknown destination MACs, or broadcasts).

## Broadcast domain vs Collision domain

These are two different, commonly confused concepts:

```
                    ┌──────────────────────────────────┐
                    │         Broadcast Domain         │
                    │   (bounded by a ROUTER/L3 device)│
                    │                                  │
                    │   ┌──────────┐    ┌──────────┐   │
                    │   │ Switch A │    │ Switch B │   │
                    │   └──────────┘    └──────────┘   │
                    │   each switch PORT is its own    │
                    │   collision domain               │
                    └──────────────────────────────────┘
```

- **Collision domain** — the set of devices that could cause a collision if they transmit at the exact same time (relevant to old shared-medium/half-duplex Ethernet, like hubs, or old coax runs). Every switch port is its own collision domain in modern full-duplex switched networks — collisions in practice are essentially a non-issue on modern switched Ethernet.
- **Broadcast domain** — the set of devices that receive a Layer 2 broadcast frame (destination MAC `FF:FF:FF:FF:FF:FF`, e.g., ARP requests). Switches **forward** broadcasts to all ports (that's the point of a broadcast) — only a **router** (a Layer 3 boundary) stops a broadcast from propagating further. This is exactly why subnetting reduces broadcast domain size and noise as a network scales.

## VLAN basics

A VLAN (Virtual LAN) lets a single physical switch be logically partitioned into multiple **separate broadcast domains**, without needing separate physical switches.

```
Physical Switch
 ┌─────────────────────────────────────────┐
 │  Port 1 (VLAN 10) ── Port 2 (VLAN 10)   │  ← same broadcast domain
 │  Port 3 (VLAN 20) ── Port 4 (VLAN 20)   │  ← different broadcast domain
 │  Port 5 (Trunk, carries VLAN 10 & 20    │
 │           tagged, to another switch)    │
 └─────────────────────────────────────────┘
```

- Frames are tagged with a **VLAN ID** (802.1Q tagging, a 12-bit field allowing up to 4094 VLANs) as they cross **trunk** links between switches, so the receiving switch knows which VLAN (broadcast domain) each frame belongs to.
- Devices in different VLANs cannot talk to each other without going through a Layer 3 device (a router, or a "Layer 3 switch"/router-on-a-stick) — exactly the same rule as separate physical subnets, because a VLAN essentially *is* a separate logical subnet/broadcast domain, just implemented on shared physical hardware.
- Practical use: segmenting traffic by department, environment (prod/dev), or security zone on the same physical switching infrastructure — conceptually parallel to how Kubernetes network policies or separate VPC subnets isolate traffic logically without separate physical hardware.

## Interview one-liners

- **"Why does a switch stop broadcast storms less than a hub, but not eliminate broadcasts entirely?"** — A switch is still a Layer 2 device — it *intentionally* forwards broadcasts to every port because that's the defined behavior of a broadcast; only a router (Layer 3) actually blocks broadcast propagation into a different network.
- **"Collision domain vs broadcast domain — what's the actual difference?"** — Collision domain = who could physically collide during a shared-medium transmission (mostly irrelevant now, since switch ports + full duplex eliminated collisions); broadcast domain = who receives broadcast frames, bounded only by routers, not switches.
- **"What problem do VLANs solve?"** — They let you create multiple isolated broadcast domains on the same physical switching hardware, avoiding the cost of separate physical switches per network segment while still enforcing traffic isolation.
