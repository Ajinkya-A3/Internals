# DNS Deep Dive — How a Name Becomes an IP Address

## The core idea

DNS is a distributed, hierarchical, heavily cached database that maps human-friendly names (`google.com`) to machine-usable addresses (IPs) and other records. It's distributed and hierarchical so no single server has to know every domain on earth — responsibility is delegated down a tree.

## The resolution hierarchy

```
                        [ . ]  Root servers (13 logical root server clusters worldwide)
                          │
                    "who handles .com?"
                          │
                          ▼
                   [ .com TLD servers ]
                          │
              "who is authoritative for google.com?"
                          │
                          ▼
              [ google.com Authoritative Name Servers ]
                          │
              "what's the A record for google.com?"
                          │
                          ▼
                  93.184.x.x  (the actual IP)
```

## Full recursive resolution walkthrough

```
 Browser/OS
     │  "what's the IP for google.com?"
     ▼
 Stub resolver (your OS) ──► Recursive Resolver (e.g., ISP resolver, 8.8.8.8, 1.1.1.1)
                                     │
                    ┌────────────────┼────────────────┐
                    │ (cache miss — resolver does the legwork on your behalf)
                    ▼
             Root Server ("go ask .com TLD servers")
                    │
                    ▼
             .com TLD Server ("go ask ns1.google.com — the authoritative server")
                    │
                    ▼
             Authoritative Server for google.com ("here's the A record: 142.250.72.14")
                    │
                    ▼
             Recursive Resolver caches this (for the record's TTL) and returns it to your OS
                    │
                    ▼
             Browser now has the IP and proceeds to open a TCP connection
```

Key distinction: the **recursive resolver** does all the iterative back-and-forth walking down the tree on your behalf; your OS/browser just asks it once and waits for a final answer. This is why it's called "recursive" from the client's perspective and "iterative" from the resolver's perspective (the resolver iteratively queries root → TLD → authoritative).

## Caching and TTL

Every DNS record has a **TTL (Time To Live)**, in seconds, that tells resolvers/browsers how long they may cache the answer before re-querying.

```
Record: google.com  A  142.250.72.14  TTL=300

t=0s     query made, resolver caches answer, valid until t=300s
t=150s   another request comes in → served straight from cache, no network query
t=310s   TTL expired → resolver must query authoritative server again
```

- **Low TTL** (e.g., 60s) → faster failover/change propagation, but more query load and slightly slower average lookups (less cache hit rate).
- **High TTL** (e.g., 86400s / 1 day) → efficient caching, but slow to propagate changes (e.g., during a DNS-based failover or IP migration, clients may keep hitting the old IP until TTL expires) — this is a classic gotcha during blue/green DNS cutovers.

Caching happens at multiple layers: browser cache → OS resolver cache → recursive resolver cache → (rarely) intermediate ISP caches — each layer can serve a cached answer and skip everything below it.

## Record types

| Record | Purpose | Example |
|---|---|---|
| **A** | Maps a hostname to an IPv4 address | `google.com → 142.250.72.14` |
| **AAAA** | Maps a hostname to an IPv6 address | `google.com → 2607:f8b0::200e` |
| **CNAME** | Alias — points one hostname to another hostname (not directly to an IP) | `www.example.com → example.com` |
| **NS** | Delegates a (sub)domain to specific authoritative name servers | `example.com → ns1.exampledns.com` |
| **MX** | Specifies mail servers responsible for accepting email for the domain, with priority | `example.com → 10 mail.example.com` |
| **SRV** | Generic service location record — host + port for a specific service (protocol, port, weight, priority) | used for things like SIP, XMPP, some Kubernetes headless-service discovery |
| **TXT** | Arbitrary text — commonly used for domain verification and email anti-spoofing (SPF/DKIM/DMARC) | `v=spf1 include:_spf.google.com ~all` |
| **PTR** | Reverse DNS — maps an IP address back to a hostname (used for reverse lookups, e.g., mail server reputation checks) | `14.72.250.142.in-addr.arpa → google.com` |

Important CNAME rule: a name that has a CNAME record **cannot** have other records (like MX) at the same name — this is why the root of a domain (the "apex"/naked domain) traditionally can't be a CNAME, which is why cloud providers invented "ALIAS"/"ANAME"-style records to work around it.

## Root servers vs TLD servers vs authoritative servers

- **Root servers** — the top of the hierarchy. They don't know specific domains; they know which TLD servers to delegate to (`.com`, `.org`, `.io`, etc.). There are 13 logical root server identities (`a.root-servers.net` through `m.root-servers.net`), each backed by many physical/anycast servers globally.
- **TLD servers** — know which authoritative servers are responsible for each specific domain under that TLD (e.g., `.com` TLD servers know that `google.com`'s authoritative servers are `ns1.google.com`, etc.), but not the actual records.
- **Authoritative servers** — the actual source of truth for a domain's records. This is where `A`, `MX`, `TXT`, etc. records actually live and are served from. In your Kubernetes/cloud world, this is the equivalent of Route53/Cloudflare DNS holding the real records for your domain, with cert-manager's DNS01 challenge writing TXT records here for Let's Encrypt validation.

## Interview question: "How does a browser resolve google.com?"

Full answer, layer by layer:
1. Browser checks its own DNS cache.
2. If empty, checks OS-level resolver cache and `/etc/hosts`.
3. If still empty, OS sends a query to the configured recursive resolver.
4. The recursive resolver checks its own cache; if it's a miss, it iteratively queries root → `.com` TLD → `google.com`'s authoritative servers.
5. The authoritative server returns the A/AAAA record.
6. The recursive resolver caches the answer for the TTL duration and returns it to the OS, which returns it to the browser.
7. The browser now has an IP and proceeds to open a TCP connection to it.

## Practical relevance to your stack

- **cert-manager + Cloudflare DNS01**: automates writing a temporary TXT record to prove domain ownership to Let's Encrypt, without needing to expose an HTTP endpoint (useful for wildcard certs, which DNS01 supports but HTTP01 does not).
- **Kubernetes internal DNS (CoreDNS)**: Services get names like `my-service.my-namespace.svc.cluster.local`, resolved by CoreDNS running as cluster DNS — same recursive/authoritative concepts apply, just scoped to the cluster.
