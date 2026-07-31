# The Packet Journey — "What Happens When You Type google.com and Press Enter?"

This is the single most common networking interview question because it forces you to touch every layer, every protocol, and every piece of infrastructure in one narrative. Master this and most other questions become sub-questions of this story.

## The full journey, end to end

```
 You type "google.com" and hit Enter
            │
            ▼
 ┌───────────────────────────┐
 │ 1. Browser checks caches   │  browser cache → OS cache → hosts file
 └───────────────────────────┘
            │ (cache miss)
            ▼
 ┌───────────────────────────┐
 │ 2. DNS Resolution          │  Recursive resolver → Root → TLD → Authoritative
 └───────────────────────────┘
            │ (now we have an IP, e.g. 142.250.72.14)
            ▼
 ┌───────────────────────────┐
 │ 3. ARP (if needed)         │  Resolve default gateway's MAC address
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 4. TCP 3-Way Handshake     │  SYN → SYN-ACK → ACK  (with the server IP:443)
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 5. TLS Handshake           │  ClientHello → ServerHello+Cert → Key exchange → Finished
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 6. HTTP Request            │  GET / HTTP/2  over the encrypted TLS tunnel
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 7. Routing across networks │  Home router → ISP → Internet backbone (BGP)
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 8. Load Balancer           │  L4/L7 LB picks a healthy backend (e.g. Envoy/NLB)
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 9. Reverse Proxy / Nginx   │  TLS termination (if not done earlier), routing rules
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 10. Application Server     │  Node/Java/Go app processes the request
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 11. Database                │  Query executes, result returned
 └───────────────────────────┘
            │
            ▼
 ┌───────────────────────────┐
 │ 12. Response travels back  │  App → Nginx → LB → Internet → ISP → Router → Browser
 └───────────────────────────┘
            │
            ▼
 Browser parses HTML, fires more requests for CSS/JS/images (repeat steps 2-12
 for each new host, often reusing existing TCP/TLS connections via keep-alive)
            │
            ▼
 Browser renders the page
```

## Step-by-step detail

### 1. Local cache checks (before any network call)
The browser checks, in order: its own DNS cache → the OS-level DNS cache → the `/etc/hosts` (or `C:\Windows\System32\drivers\etc\hosts`) file. If any of these has a fresh entry, DNS resolution is skipped entirely.

### 2. DNS Resolution
If no cache hit, the OS's stub resolver sends a query to the configured **recursive resolver** (your ISP's resolver, or something like `8.8.8.8`/`1.1.1.1`). That resolver:

1. Asks a **root server** ("who handles `.com`?") → gets referred to the `.com` **TLD server**.
2. Asks the TLD server ("who is authoritative for `google.com`?") → gets referred to Google's **authoritative name server**.
3. Asks the authoritative server for the **A/AAAA record** → gets back the actual IP address.
4. Caches the result for the record's **TTL** and returns the IP to your OS.

(Full depth in the dedicated DNS README.)

### 3. ARP — getting to the first hop
Your machine now knows the destination *IP*, but Ethernet frames need a *MAC address*. Since the destination (Google's server) is nowhere near your local subnet, your OS instead resolves the MAC address of your **default gateway** (your router) via ARP — "who has 192.168.1.1? Tell 192.168.1.42" — and sends the frame there. The router will handle everything beyond your subnet.

### 4. TCP Three-Way Handshake
Before any data flows, your OS establishes a reliable TCP connection to the server's IP on port 443:

```
Client                          Server
  │──────── SYN (seq=x) ───────►│
  │◄──── SYN-ACK (seq=y,ack=x+1)│
  │──────── ACK (ack=y+1) ─────►│
        Connection established
```

(Full mechanics in the TCP Internals README.)

### 5. TLS Handshake
Once TCP is up, the browser and server negotiate an encrypted channel:

- **ClientHello**: supported TLS versions, cipher suites, and — critically — **SNI** (Server Name Indication, so the server knows *which* certificate to present when many sites share one IP).
- **ServerHello + Certificate**: server sends its certificate (chained to a trusted CA) and picks a cipher suite.
- **Key exchange** (commonly ECDHE for Perfect Forward Secrecy): both sides derive a shared symmetric session key without ever transmitting it.
- **Finished**: both sides switch to fast symmetric encryption (AES-GCM, ChaCha20) for the rest of the session.

(Full depth in the HTTPS/TLS README.)

### 6. HTTP Request
Only now does the actual application-layer request go out, inside the encrypted tunnel:

```
GET / HTTP/2
Host: google.com
Accept: text/html
...
```

### 7. Routing across the internet
The packet leaves your home router (NAT translates your private IP to your public IP), reaches your **ISP**, and from there traverses the internet backbone. Between autonomous networks (ISPs, cloud providers), **BGP** decides the path a packet takes hop by hop — there's no end-to-end "connection" at this layer, just successive routers each making a local best-next-hop decision based on their routing table (longest prefix match).

### 8. Load Balancer
Traffic arrives at the destination's edge — commonly a cloud **Layer 4 load balancer** (e.g., an AWS NLB) that does fast IP/port-based distribution across healthy targets, or a **Layer 7 load balancer/gateway** (e.g., Envoy Gateway, an ALB, or Nginx acting as LB) that can also inspect the HTTP request (Host header, path) to route intelligently — this is exactly the layer where a Kubernetes ingress/Gateway API resource does its job.

### 9. Reverse proxy
Nginx (or Envoy) may terminate TLS here (if it wasn't terminated at the LB), apply routing rules, rate limiting, and forward the request over an internal network to the actual application pod/container.

### 10. Application server
Your app code runs: authentication, business logic, and it typically needs data.

### 11. Database
The app queries a database (e.g., Postgres/Aurora) over its own TCP connection (often pooled and long-lived, unlike the ephemeral per-request browser connection), gets results back.

### 12. Response path
The response retraces the same path in reverse: DB → app → Nginx → LB → internet → ISP → your router → your machine, re-encrypted at every hop where TLS was terminated, and re-encapsulated at every layer going back down and up the stack.

### Rendering
The browser parses the returned HTML and discovers references to CSS, JS, images, fonts — each of which may require its own DNS lookup (unless already cached) and its own TCP/TLS handshake (unless reusing a persistent HTTP/1.1 keep-alive connection or multiplexed HTTP/2/HTTP/3 stream) before the page is fully rendered.

## Why this question is asked so often

It's a single narrative that naturally forces you to demonstrate:
- DNS internals
- TCP handshake and reliability
- TLS/security fundamentals
- Routing and how packets actually traverse independent networks
- Load balancing and reverse proxy behavior (very relevant to your EKS/Envoy Gateway/ArgoCD stack)
- Where a request could fail at each hop, which is exactly how you'd debug a real production incident (DNS failure vs TCP timeout vs TLS handshake failure vs 5xx from the app vs slow DB query)

## Debugging mental model (SRE angle)

| Symptom | Likely layer/step to check |
|---|---|
| `curl: could not resolve host` | DNS (step 2) |
| `Connection timed out` | Routing/firewall/security group (steps 3-4, 7-8) |
| `Connection refused` | Nothing listening on that port at destination (step 4/9) |
| `SSL handshake failed` | TLS (step 5) — cert mismatch, SNI issue, cipher mismatch |
| HTTP 502/504 from LB | Backend/app unhealthy or slow (steps 8-10) |
| HTTP 500 | Application bug (step 10) |
| Slow response, healthy connection | Database query or downstream dependency (step 11) |
