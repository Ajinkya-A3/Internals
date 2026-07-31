# OSI vs TCP/IP Model — Why Layers Exist and How Data Actually Moves

## Why layers exist at all

Networking is broken into layers for one core engineering reason: **separation of concerns**. Each layer solves exactly one problem and exposes a clean interface to the layer above it, without caring how the layer below implements its job.

Practical payoff of this design:

- **Interoperability** — a browser on Windows talks to Nginx on Linux without either side knowing the other's OS, NIC vendor, or cabling.
- **Independent evolution** — you can swap Wi-Fi for fiber (Layer 1/2) without touching your HTTP application code (Layer 7).
- **Troubleshooting boundary** — "is this a DNS problem, a TCP problem, or an app bug?" is a layer question. SREs debug layer-by-layer for exactly this reason (`ping` → `traceroute` → `curl -v` → app logs maps roughly to L3 → L3/4 → L7).
- **Reusability** — TCP doesn't care if it's carrying HTTP, SSH, or SMTP. HTTP doesn't care if it's running over Ethernet or a cellular link.

## OSI Model (7 layers) — conceptual, teaching model

```
Layer 7  Application     HTTP, DNS, SSH, SMTP, gRPC       "What data means"
Layer 6  Presentation    TLS encryption, compression       "Format/encoding"
Layer 5  Session         Sessions, sockets (logical)        "Dialog control"
Layer 4  Transport       TCP, UDP                            "End-to-end delivery"
Layer 3  Network         IP, ICMP, routing                   "Host-to-host addressing"
Layer 2  Data Link       Ethernet, MAC, ARP, switches         "Node-to-node on a wire"
Layer 1  Physical        Cables, radio, voltages, fiber       "Bits on the medium"
```

OSI is a **reference model** — nobody implements a strict 7-layer stack in production. It's mainly used as a shared vocabulary for reasoning and troubleshooting ("that's a Layer 3 issue", "that's an L7 load balancer").

## TCP/IP Model (4-5 layers) — what the real internet runs on

```
Application   HTTP, DNS, SSH, TLS, gRPC     (maps to OSI L5+L6+L7)
Transport     TCP, UDP                       (maps to OSI L4)
Internet      IP, ICMP, routing              (maps to OSI L3)
Link          Ethernet, Wi-Fi, ARP           (maps to OSI L1+L2)
```

This is the model actually implemented in every OS network stack (Linux `netfilter`/sockets, Windows, etc.) and the one referenced in RFCs.

## Side-by-side mapping

| OSI Layer | OSI Name | TCP/IP Layer | Example Protocols |
|---|---|---|---|
| 7 | Application | Application | HTTP, HTTPS, DNS, SSH, FTP, SMTP, gRPC |
| 6 | Presentation | Application | TLS/SSL, JSON/Protobuf encoding, compression |
| 5 | Session | Application | Sockets, TLS session resumption |
| 4 | Transport | Transport | TCP, UDP, QUIC |
| 3 | Network | Internet | IP (v4/v6), ICMP, IPsec, BGP/OSPF (control plane) |
| 2 | Data Link | Link | Ethernet, ARP, VLAN (802.1Q), switches |
| 1 | Physical | Link | Copper, fiber, radio (Wi-Fi), voltage/light signaling |

## How data moves down the stack: Encapsulation

Each layer wraps the layer above's data with its own header (and sometimes trailer). This is **encapsulation**. The receiving side reverses it — **decapsulation** — stripping one header per layer as data moves up.

```
Application data:                              [ HTTP GET /index.html ]
                                                       │
                                                       ▼ (L4 wraps)
Transport (TCP segment):        [TCP Header][ HTTP GET /index.html ]
                                                       │
                                                       ▼ (L3 wraps)
Internet (IP packet):     [IP Header][TCP Header][ HTTP GET /index.html ]
                                                       │
                                                       ▼ (L2 wraps)
Link (Ethernet frame): [Eth Header][IP Header][TCP Header][ HTTP data ][Eth Trailer/FCS]
                                                       │
                                                       ▼ (L1)
Physical: 10101001110001... (bits on the wire/radio)
```

Terminology per layer (important for interviews — precise naming matters):

| Layer | Unit of data (PDU) |
|---|---|
| Application | Data / Message |
| Transport | Segment (TCP) / Datagram (UDP) |
| Internet | Packet |
| Link | Frame |
| Physical | Bits |

## Decapsulation on the receiving end

```
Bits arrive on wire
   │
   ▼
NIC reads Ethernet Frame → strips Eth header → hands IP packet up
   │
   ▼
Kernel IP layer reads IP header → checks destination IP → strips it → hands TCP segment up
   │
   ▼
Kernel TCP layer reads TCP header → checks port, seq/ack, reorders → strips it → hands app data up
   │
   ▼
Application (e.g., Nginx) reads HTTP request bytes
```

At every hop **only the layers needed to route/forward are inspected**:
- A Layer 2 switch only reads the Ethernet header (MAC addresses).
- A Layer 3 router reads up to the IP header (destination IP) to make forwarding decisions — it does not need to open the TCP segment.
- A Layer 4 load balancer (e.g., an L4 ELB/NLB) reads IP + TCP/UDP headers (ports) but not HTTP.
- A Layer 7 load balancer/reverse proxy (e.g., Nginx, Envoy Gateway) reads all the way into the HTTP request — headers, path, cookies — to make routing decisions (this is exactly why Envoy Gateway/Envoy can do host/path-based routing while an NLB cannot).

## Where protocols "belong" (quick reference)

| Layer | Protocols / Technologies |
|---|---|
| Application | HTTP/HTTPS, DNS, SSH, SMTP, FTP, gRPC, WebSocket |
| Transport | TCP, UDP, QUIC (technically sits over UDP but does transport's job) |
| Internet | IPv4, IPv6, ICMP, IGMP, IPsec |
| Link | Ethernet, ARP, PPP, VLAN tagging, Wi-Fi (802.11) |
| Physical | Copper (Cat5e/6), fiber, radio frequencies |

## Interview one-liners

- **"Why does OSI have 7 layers but TCP/IP only 4?"** — OSI is a theoretical teaching/reference model designed before the internet was built; TCP/IP is the practical model the internet was actually engineered around. TCP/IP merges OSI's L5-L7 into one "Application" layer and L1-L2 into one "Link" layer because in practice those distinctions rarely matter for implementation.
- **"What is encapsulation?"** — Each layer adds its own header (metadata) to the payload from the layer above so the receiving peer at the same layer can interpret it correctly, without needing to understand the layers above.
- **"Why can a router forward packets without seeing HTTP data?"** — Because forwarding only requires the IP header (Layer 3). Everything above that is opaque payload to the router.
