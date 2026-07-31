# HTTP — From HTTP/1.1 to HTTP/3

## What HTTP fundamentally is

HTTP is a stateless, request-response application-layer protocol. "Stateless" means the server treats every request independently by default — nothing is remembered between requests unless the application layers something on top (cookies, sessions, tokens).

## HTTP/1.1

```
Client                                Server
  │──── GET /page HTTP/1.1 ─────────►│
  │      Host: example.com            │
  │      Connection: keep-alive        │
  │◄──── HTTP/1.1 200 OK ─────────────│
  │      Content-Length: 1234          │
  │      (body)                        │
  │  (connection stays open for reuse) │
```

- Introduced **persistent connections** (keep-alive) by default, so a new TCP handshake isn't required for every single request — a huge improvement over HTTP/1.0's one-request-per-connection model.
- Problem: **Head-of-line (HOL) blocking at the application layer.** Requests on a single connection are processed strictly in order unless you use pipelining (rarely used, and still blocks on the *first* response). This is why browsers historically opened 6+ parallel TCP connections per host — a workaround, not a fix.

## HTTP/2

```
              Single TCP connection
  ┌───────────────────────────────────────────┐
  │  Stream 1: GET /style.css   ───────────►   │
  │  Stream 2: GET /app.js      ───────────►   │  All multiplexed
  │  Stream 3: GET /logo.png    ───────────►   │  over ONE connection
  │  Responses can arrive in ANY order,        │
  │  interleaved as frames, then reassembled   │
  └───────────────────────────────────────────┘
```

Key improvements over HTTP/1.1:
- **Multiplexing** — many requests/responses share a single TCP connection concurrently as independent "streams," eliminating application-layer HOL blocking (no more waiting for response #1 before response #2 can be sent).
- **Header compression (HPACK)** — repetitive headers (cookies, user-agent, etc.) are compressed across requests instead of resent in full every time.
- **Server push** (largely deprecated in practice now, browsers dropped support) — server could proactively send resources it knows the client will need.
- **Binary framing** instead of HTTP/1.1's plain-text protocol — more efficient to parse, less error-prone.

Remaining problem: HTTP/2 fixed *application-layer* HOL blocking, but it still rides on a single TCP connection — so a **lost TCP packet still blocks all streams** on that connection until it's retransmitted (TCP itself enforces strict byte-order delivery to the app, regardless of which HTTP/2 stream that byte belonged to). This is **transport-layer HOL blocking**, and it's exactly what HTTP/3 was designed to solve.

## HTTP/3

```
       Built on QUIC (runs over UDP, not TCP)
  ┌───────────────────────────────────────────┐
  │  Stream 1 ──►  (independent, own ordering) │
  │  Stream 2 ──►  (packet loss here doesn't   │
  │  Stream 3 ──►   block Streams 1 & 2!)      │
  └───────────────────────────────────────────┘
```

- Built on **QUIC**, which runs over UDP instead of TCP, giving it full control over reliability and congestion control at the application layer instead of being locked into the kernel's TCP behavior.
- Each stream has **independent loss recovery** — a lost packet only stalls the one stream it belonged to, not the entire connection. This finally eliminates transport-level HOL blocking.
- **Faster connection setup** — QUIC combines the transport handshake and TLS 1.3 handshake into effectively one round trip (sometimes zero extra round trips for resumed connections via 0-RTT), versus HTTP/2's separate TCP handshake + TLS handshake.
- Built-in **connection migration** — a QUIC connection is identified by a connection ID, not just IP:port, so switching networks (Wi-Fi → cellular) doesn't necessarily break an in-progress connection, unlike TCP where a changed IP kills the connection.

## Persistent connections & keep-alive

`Connection: keep-alive` (HTTP/1.1 default) tells both sides to reuse the same TCP connection for multiple requests instead of tearing it down and re-handshaking every time — saves a full TCP handshake (and TLS handshake, if HTTPS) per request. HTTP/2 and HTTP/3 take this further by default via multiplexing, so reuse isn't even a special case — it's the norm.

## Methods

| Method | Purpose | Idempotent? | Safe? |
|---|---|---|---|
| GET | Retrieve a resource | Yes | Yes |
| POST | Create a resource / submit data | No | No |
| PUT | Replace a resource entirely | Yes | No |
| PATCH | Partially update a resource | No (usually) | No |
| DELETE | Remove a resource | Yes | No |
| HEAD | Like GET but headers only, no body | Yes | Yes |
| OPTIONS | Discover allowed methods/CORS preflight | Yes | Yes |

**Safe** = doesn't change server state. **Idempotent** = calling it once or N times has the same effect on server state (important for retry logic — safe to blindly retry a `PUT`/`DELETE` on network failure, risky to blindly retry a `POST`, since it might create duplicates).

## Headers, Cookies, Sessions

- **Headers** carry metadata: `Content-Type`, `Content-Length`, `Authorization`, `Cache-Control`, `Host`, etc.
- **Cookies** (`Set-Cookie` from server, `Cookie` from client) are the classic mechanism to add state on top of stateless HTTP — the server hands the client a token, the client sends it back on every subsequent request.
- **Sessions** are typically server-side state keyed by a session ID stored in a cookie — the actual session data (user identity, cart contents) lives in the server (or a shared store like Redis), and the cookie is just a pointer to it.

## Status codes

| Range | Meaning |
|---|---|
| 1xx | Informational (e.g., 101 Switching Protocols) |
| 2xx | Success (200 OK, 201 Created, 204 No Content) |
| 3xx | Redirection (301 Moved Permanently, 302 Found, 304 Not Modified) |
| 4xx | Client error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests) |
| 5xx | Server error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout) |

502/503/504 are especially relevant for reverse proxy/LB debugging: **502** = upstream sent an invalid response (or nothing listening), **503** = service intentionally unavailable/overloaded (health check failing), **504** = upstream didn't respond in time.

## Chunked encoding

`Transfer-Encoding: chunked` lets a server stream a response of *unknown total length* — the body is sent in a series of length-prefixed chunks, terminated by a zero-length chunk, instead of requiring an upfront `Content-Length`. Useful for dynamically generated content (e.g., streaming an LLM response token by token).

## Compression

`Content-Encoding: gzip` / `br` (Brotli) — the server compresses the response body, negotiated via the client's `Accept-Encoding` request header. Reduces bytes over the wire at the cost of CPU to compress/decompress.

## Range requests

`Range: bytes=500-999` lets a client request only a portion of a resource — critical for resumable downloads and video seeking (server responds `206 Partial Content` if it supports ranges, indicated by `Accept-Ranges: bytes`).

## Idempotency (practical importance)

Idempotency matters most for **retry safety**. If a client times out waiting for a response to a `POST /charge-card`, blindly retrying could double-charge the customer — because POST isn't guaranteed idempotent. This is why many APIs add an explicit `Idempotency-Key` header so the server can recognize and dedupe a retried request, decoupling retry-safety from the HTTP method itself.
