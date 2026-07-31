# ARP — Address Resolution Protocol: Turning an IP into a MAC Address

## The problem ARP solves

IP addresses are how Layer 3 (routing) identifies hosts logically. But actually delivering a frame on a local network (Ethernet/Wi-Fi) requires a **MAC address** — the Layer 2 hardware address. ARP is the glue between the two: **given an IP address on the same local subnet, find the corresponding MAC address.**

```
"I know the destination's IP (192.168.1.50), but to actually put a frame
 on the wire, I need to know: whose network card is that?"
                          │
                          ▼
                   ARP solves this
```

## How ARP works, step by step

```
Host A (192.168.1.10, MAC AA:AA:AA:AA:AA:AA)
Host B (192.168.1.20, MAC BB:BB:BB:BB:BB:BB)

Step 1 — Host A broadcasts an ARP Request to EVERYONE on the local subnet:
   ┌─────────────────────────────────────────────────┐
   │  Broadcast (dest MAC = FF:FF:FF:FF:FF:FF)         │
   │  "Who has 192.168.1.20? Tell 192.168.1.10"        │
   └─────────────────────────────────────────────────┘
                          │
        Every host on the subnet receives this frame,
        but only Host B recognizes its own IP and responds

Step 2 — Host B sends an ARP Reply, UNICAST directly back to Host A:
   ┌─────────────────────────────────────────────────┐
   │  Unicast (dest MAC = AA:AA:AA:AA:AA:AA)           │
   │  "192.168.1.20 is at BB:BB:BB:BB:BB:BB"           │
   └─────────────────────────────────────────────────┘

Step 3 — Host A now caches this mapping (ARP cache) and can send
         Ethernet frames directly addressed to BB:BB:BB:BB:BB:BB
```

The request is a **broadcast** (nobody knows in advance who owns that IP, so everyone must hear it), but the reply is a **unicast** (the responder knows exactly who asked).

## Why the ARP cache exists

Broadcasting an ARP request for every single packet would be wildly wasteful — it would flood the local network constantly. So each host maintains an **ARP cache** (a table of recently resolved IP→MAC mappings) with a timeout (commonly minutes), so repeated communication with the same neighbor reuses the cached MAC instead of re-broadcasting.

```
$ arp -a
? (192.168.1.1) at aa:bb:cc:dd:ee:ff [ether] on eth0
? (192.168.1.20) at bb:bb:bb:bb:bb:bb [ether] on eth0
```

Entries expire and get re-resolved periodically — this also naturally handles cases where a device's NIC changes (new hardware, VM migration) since a stale cache entry will eventually time out and be re-queried.

## What happens for hosts on the SAME subnet vs a DIFFERENT subnet

This is a critical distinction interviewers probe:

```
Same subnet (192.168.1.10 → 192.168.1.20, both /24):
   Host A ARPs directly for 192.168.1.20's MAC
   Frame is addressed directly to Host B's MAC
   Delivered directly, no router involved

Different subnet (192.168.1.10 → 8.8.8.8):
   Host A does NOT try to ARP for 8.8.8.8 directly (it's not on the local subnet)
   Instead, Host A ARPs for its DEFAULT GATEWAY's MAC (e.g., 192.168.1.1)
   Frame is addressed to the gateway's MAC, but the IP header still says
   destination = 8.8.8.8 — the gateway/router will handle forwarding from there
```

This is exactly why "how IP becomes MAC" and "what's a default gateway for" are related questions — the decision of *who to ARP for* is made by comparing the destination IP against your own subnet mask: same subnet → ARP for the destination directly; different subnet → ARP for the gateway instead, and let routing take over beyond that hop.

## Gratuitous ARP (bonus, often comes up)

A host can send an ARP announcement without being asked ("here's my own IP-to-MAC mapping, whether you asked or not") — used to update everyone's ARP caches proactively, e.g., after a failover event (a virtual IP moving to a new physical host, like in keepalived/VRRP setups) so the network immediately learns the new MAC for that IP instead of waiting for cache timeout.

## Interview one-liners

- **"How does IP become MAC?"** — Via ARP: broadcast a request asking "who has this IP," the owner replies directly with its MAC, and the mapping gets cached locally.
- **"Why does the ARP cache exist?"** — To avoid re-broadcasting an ARP request for every single packet — cached mappings are reused until they expire, dramatically reducing broadcast traffic.
- **"What happens inside the same subnet?"** — Direct ARP resolution for the destination host's own MAC address, then direct frame delivery — no gateway/router involvement needed.
- **"What happens across subnets?"** — The sender ARPs for its default gateway's MAC (not the final destination's), sends the frame to the gateway, and routing (Layer 3) takes over from there.
