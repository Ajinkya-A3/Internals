# IP Addressing — IPv4, CIDR, Subnetting, NAT

## IPv4 basics

An IPv4 address is 32 bits, written as 4 decimal octets (0-255 each):

```
   192   .   168   .    1    .    42
 11000000.10101000.00000001.00101010
```

Every address logically splits into a **network portion** and a **host portion** — the whole point of subnetting is deciding where that split happens.

## CIDR (Classless Inter-Domain Routing)

CIDR notation (`IP/prefix-length`) replaced the old rigid Class A/B/C system, letting network size be defined flexibly instead of only in fixed increments.

```
192.168.1.0/24
             │
             └─ first 24 bits = network portion, remaining 8 bits = host portion

/24  → 2^8  = 256 addresses  (254 usable hosts)
/16  → 2^16 = 65,536 addresses
/28  → 2^4  = 16 addresses   (14 usable hosts)
/32  → 1 address (a single host route)
```

More prefix bits = smaller network, fewer prefix bits = larger network. This is precisely how Kubernetes CNI plugins carve up pod CIDR ranges per node (e.g., a `/24` per node out of a larger `/16` VPC or cluster CIDR).

## Subnetting

Subnetting splits one larger network into smaller, isolated broadcast domains — for security segmentation, reducing broadcast traffic, and matching IP allocation to actual topology (e.g., separate subnets per Availability Zone in a VPC).

```
Parent network: 10.0.0.0/16   (65,536 addresses)

Split into subnets:
  10.0.0.0/24    → AZ-a public subnet   (256 addresses)
  10.0.1.0/24    → AZ-a private subnet
  10.0.2.0/24    → AZ-b public subnet
  10.0.3.0/24    → AZ-b private subnet
  ...
```

Every subnet reserves:
- **Network address** (all host bits = 0) — identifies the subnet itself, not assignable to a host.
- **Broadcast address** (all host bits = 1) — used to send a message to every host on that subnet simultaneously.

So a `/24` has 256 total addresses, but only 254 are usable for hosts.

## Broadcast address

The broadcast address is the "send to everyone on this subnet" address. For `192.168.1.0/24`, that's `192.168.1.255`. A broadcast frame reaches every host in the same Layer 2 broadcast domain but is **not** forwarded across routers by default — routers deliberately don't forward broadcast traffic between subnets, which is one reason subnetting reduces broadcast noise as a network grows.

## Private vs Public IP

| Range | Scope |
|---|---|
| `10.0.0.0/8` | Private (RFC 1918) |
| `172.16.0.0/12` | Private (RFC 1918) |
| `192.168.0.0/16` | Private (RFC 1918) |
| Everything else routable | Public (globally unique, routable across the internet) |

Private IPs are **not routable on the public internet** — routers on the internet backbone will drop packets addressed to them. They're reused independently inside millions of separate private networks (your home network, your office, a VPC), which is only possible because they never need to be globally unique — only unique *within* their own private network.

## NAT & PAT

**NAT (Network Address Translation)** lets many devices with private IPs share one (or a few) public IP(s) to reach the internet, translating addresses at the boundary (e.g., your home router, or a VPC's NAT gateway).

**PAT (Port Address Translation)**, the far more common variant (also called NAPT or "NAT overload"), additionally rewrites the **source port** so many internal devices can share a *single* public IP simultaneously — the router keeps a translation table mapping `(private IP, private port) ↔ (public IP, public port)` for every active connection.

```
Internal: 192.168.1.10:51234 ──┐
Internal: 192.168.1.11:51234 ──┼──► NAT/PAT translation table ──► Public: 203.0.113.5:40001
Internal: 192.168.1.12:51234 ──┘                                  Public: 203.0.113.5:40002
                                                                   Public: 203.0.113.5:40003

Return traffic to 203.0.113.5:40001 is translated back to 192.168.1.10:51234
```

Without PAT, you'd need one public IP per simultaneously NAT'd device — PAT is why an entire household or office can share one public IP.

## Default Gateway

The default gateway is the router a host sends traffic to whenever the destination isn't on its own local subnet — effectively "if I don't know where to send this, send it to my gateway and let it figure out the rest."

```
Host: 192.168.1.42/24
Default gateway: 192.168.1.1

Destination 192.168.1.50 → same subnet → send directly (ARP for its MAC, no gateway needed)
Destination 8.8.8.8       → different subnet → send to default gateway (192.168.1.1) for forwarding
```

## Interview one-liners

- **"What does /24 mean?"** — The first 24 bits are the fixed network portion; the remaining 8 bits identify hosts within that network, giving 256 total addresses (254 usable).
- **"Why can't two different offices both use 10.0.0.0/8 and still communicate directly?"** — Because private IP ranges are only guaranteed unique *within* their own network — if both networks were merged or connected via VPN without renumbering, you'd get IP address collisions (a real, common problem when connecting VPCs or company networks via VPN/peering).
- **"NAT vs PAT?"** — NAT translates addresses one-to-one or many-to-few; PAT (the common case) additionally uses port numbers so many devices can share a single public IP simultaneously.
