# HTTPS & TLS — How Encryption Actually Gets Established

## The core problem TLS solves

Two strangers (browser, server) who've never met need to agree on a shared secret key, over a network that anyone can eavesdrop on, **without ever transmitting that secret in a form an eavesdropper could steal.** TLS solves this using asymmetric cryptography for the handshake, then switches to symmetric cryptography for the actual data — for very deliberate performance reasons.

## Symmetric vs Asymmetric encryption

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared secret key, used by both sides | A key *pair*: public key (shareable) + private key (secret) |
| Speed | Fast (AES-GCM can do gigabytes/sec) | Slow (100-1000x slower for equivalent security) |
| Problem it solves | Bulk data encryption | Secure key exchange / identity verification, without ever sharing a secret |
| Example algorithms | AES, ChaCha20 | RSA, ECDHE (Elliptic Curve Diffie-Hellman) |

## Why TLS switches to symmetric encryption after the handshake

Asymmetric crypto is computationally expensive — encrypting an entire video stream or API payload with RSA would be unusably slow. So TLS uses asymmetric crypto **only** to solve the hard problem (safely agreeing on a secret with a stranger over a public network), then hands off to fast symmetric encryption for the actual bulk data transfer, using the secret both sides now share. Best of both worlds: security where it's hard, speed where it matters.

## The TLS 1.2/1.3-style handshake (conceptual)

```
Client                                          Server
  │──── ClientHello ──────────────────────────►│
  │      (TLS versions supported, cipher       │
  │       suites, SNI = "which hostname",      │
  │       ALPN = "which app protocol")          │
  │                                              │
  │◄──── ServerHello + Certificate ─────────────│
  │      (chosen cipher suite, server's         │
  │       certificate signed by a trusted CA)   │
  │                                              │
  │──── Key exchange (e.g., ECDHE) ────────────►│
  │◄──── Key exchange ───────────────────────────│
  │      Both sides independently derive the    │
  │      SAME symmetric session key without     │
  │      ever transmitting it on the wire       │
  │                                              │
  │──── Finished (encrypted) ──────────────────►│
  │◄──── Finished (encrypted) ───────────────────│
  │                                              │
  │   ═══ All further data: fast symmetric ═══  │
  │        encryption (AES-GCM / ChaCha20)       │
```

TLS 1.3 streamlined this to effectively **one round trip** (down from two in TLS 1.2), and supports **0-RTT resumption** for previously-visited servers (send encrypted application data in the very first flight, at the cost of some replay-attack risk for non-idempotent requests).

## Certificates & the Certificate Authority (CA) chain

A certificate binds a public key to an identity (a domain name) and is only trustworthy because it's **signed by a Certificate Authority** that the client already trusts (root CAs are pre-installed in browsers/OSes).

```
Root CA (trusted, pre-installed in OS/browser trust store)
     │  signs
     ▼
Intermediate CA
     │  signs
     ▼
Your server's certificate (e.g., google.com, cert-manager-issued)
```

When a browser connects, it verifies this **chain of trust** up to a root CA it already trusts — that's the whole basis of "why should I trust this stranger's public key?" Without this chain, anyone could hand out a self-signed cert claiming to be `google.com` and there'd be no way to tell.

This is exactly what **cert-manager** automates in your Kubernetes stack: it requests certificates from Let's Encrypt (a CA), proves domain ownership (via the DNS01 challenge with Cloudflare, writing a temporary TXT record — required for wildcard certs), and gets back a cert signed by a chain browsers already trust.

## Public key vs Private key

- **Public key** — shareable, embedded in the certificate, used by others to encrypt data meant only for you, or to verify a signature you made.
- **Private key** — never shared, kept secret on the server, used to decrypt data encrypted with your public key, or to sign data (proving *you* — and only you — produced it).

## Cipher suites

A cipher suite is the bundle of algorithms negotiated for a specific connection, e.g.:

```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
     │        │        │        │
     │        │        │        └─ Hash function (for integrity/HMAC)
     │        │        └────────── Symmetric cipher for bulk encryption
     │        └─────────────────── Authentication/signature algorithm
     └──────────────────────────── Key exchange algorithm
```

Modern best practice (TLS 1.3) narrows this drastically to a small, vetted set of secure combinations — TLS 1.3 actually removed support for older, weaker algorithms (like RSA key exchange without forward secrecy) entirely.

## ALPN (Application-Layer Protocol Negotiation)

An extension in the TLS handshake (`ClientHello`) letting the client and server agree on the *application* protocol to use over the encrypted connection — e.g., `h2` (HTTP/2) vs `http/1.1` — **before** any HTTP-level data is exchanged, avoiding a separate negotiation round trip. This is how a browser and server agree to speak HTTP/2 instead of HTTP/1.1 without extra latency.

## SNI (Server Name Indication)

Also sent in `ClientHello`, in **plaintext** (a historical privacy gap partially addressed by newer Encrypted ClientHello/ECH efforts) — it tells the server *which hostname* the client is trying to reach, **before** the TLS handshake completes. This is essential for shared-IP hosting: many domains can sit behind one IP/load balancer, and SNI is how the server (or an Envoy Gateway/Nginx in front of it) knows which certificate to present for that specific request.

## Perfect Forward Secrecy (PFS)

PFS means each session's symmetric key is derived from **ephemeral** key exchange values (e.g., ECDHE — the "E" stands for Ephemeral) that are discarded after the session ends and never reused. The consequence: even if an attacker later steals the server's long-term private key, they **cannot** decrypt previously captured traffic, because the actual session keys were never derived from — or stored alongside — that long-term key. Without PFS (older RSA key exchange), a single compromised private key could retroactively decrypt years of recorded traffic.

## Interview one-liners

- **"Why not just use asymmetric encryption for everything?"** — Too slow for bulk data; asymmetric crypto is reserved for the hard problem of agreeing on a secret safely, then handed off to fast symmetric crypto for actual data.
- **"What does a certificate actually prove?"** — That a CA the client already trusts has verified this public key belongs to this specific domain — nothing more.
- **"What's the difference between SNI and ALPN?"** — SNI says *which hostname* you're trying to reach (for cert selection on multi-tenant servers); ALPN says *which application protocol* to speak (HTTP/1.1 vs HTTP/2) — both negotiated during the same handshake, for different purposes.
- **"What does Perfect Forward Secrecy actually protect against?"** — Retroactive decryption of past sessions if a server's long-term private key is later compromised.
