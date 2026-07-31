# TCP Internals — Handshakes, Reliability, Flow & Congestion Control

TCP's entire job is to turn an unreliable, best-effort IP network into a **reliable, ordered, byte-stream** service for applications. Every mechanism below exists to serve that one goal.

## Three-Way Handshake (connection establishment)

```
Client                                  Server
  │                                        │
  │──────── SYN (seq = x) ───────────────►│   "I want to talk, my starting seq is x"
  │                                        │
  │◄─────── SYN-ACK (seq = y, ack = x+1) ─│   "OK, here's mine (y), got yours (x+1 expected next)"
  │                                        │
  │──────── ACK (ack = y+1) ─────────────►│   "Got it, confirmed"
  │                                        │
  │         connection ESTABLISHED         │
```

Why three steps and not two? Because TCP is **full-duplex** — both sides need to prove they can send *and* receive, and both sides need to agree on each other's **initial sequence number (ISN)**. Two-way isn't enough to confirm the server's SYN was received by the client.

## Four-Way Termination (connection close)

TCP connections close independently in each direction (it's full-duplex, so either side can finish sending while still receiving):

```
Client                                  Server
  │──────── FIN ──────────────────────►│   "I'm done sending"
  │◄─────── ACK ────────────────────────│
  │                                      │   (server may still send data)
  │◄─────── FIN ─────────────────────────│  "I'm done sending too"
  │──────── ACK ──────────────────────►│
  │        connection CLOSED             │
```

Because each direction closes separately, it takes 4 messages (not 3) — a combined FIN+ACK is possible as an optimization but isn't guaranteed.

## Sequence Numbers & ACKs

Every byte sent over a TCP connection is numbered. This is how TCP achieves ordering and reliability:

- Each segment's TCP header carries a **sequence number** = the byte offset of the first byte in that segment.
- The receiver sends back an **ACK number** = the next byte it expects (i.e., "I've received everything up through X, send me X+1 next").
- If ACKs stop advancing or a gap is detected, the sender knows something was lost.

```
Sender sends:      [Seq=1000, len=500]  [Seq=1500, len=500]  [Seq=2000, len=500]
                          │                      │ (lost)              │
Receiver gets:      Seq=1000 ✓            ---- LOST ----        Seq=2000 (out of order, buffered)
Receiver ACKs:      ACK=1500              ACK=1500 (dup)        ACK=1500 (dup, again)
```

Three duplicate ACKs for the same number is a strong signal to the sender: "segment starting at 1500 never arrived — retransmit it" (this is **fast retransmit**, faster than waiting for a timeout).

## Sliding Window & Flow Control

TCP doesn't wait for an ACK after every single segment (that would be painfully slow). Instead, the sender can have multiple segments "in flight" up to the size of the **receive window (rwnd)** advertised by the receiver.

```
        |<---------------- Window Size ---------------->|
Sent+ACKed | Sent, awaiting ACK | Can send now | Can't send yet (outside window)
```

- The **receive window** is how the receiver does **flow control** — it tells the sender "don't send me more than N unacknowledged bytes, because that's all the buffer space I have right now." This prevents a fast sender from overwhelming a slow receiver.
- As ACKs come in for already-sent data, the window "slides" forward, opening room for new bytes to be sent.
- **Window scaling** (a TCP option negotiated at handshake time) lets the window exceed the original 16-bit field's ~64KB limit — essential for high-bandwidth, high-latency links (e.g., cross-region traffic) to keep the pipe full.

## Congestion Control (different problem from flow control)

Flow control protects the *receiver*. Congestion control protects the *network* from being overwhelmed by the sender — regulated by a separate value, the **congestion window (cwnd)**, that the sender maintains based on observed network conditions (not what the receiver tells it).

Classic phases (TCP Reno/CUBIC-style, simplified):

```
cwnd
 │                              ______  <- congestion detected, cwnd halved
 │                        _____/       \______
 │                  _____/                     \_____ (congestion avoidance:
 │            _____/                                   linear growth)
 │      _____/  <- slow start (exponential growth)
 │_____/
 └──────────────────────────────────────────────────► time
```

1. **Slow start** — cwnd starts small and doubles roughly every RTT (exponential growth) until it hits a threshold or loss is detected.
2. **Congestion avoidance** — after the threshold, growth becomes linear (additive increase) to probe capacity more cautiously.
3. **On loss** — cwnd is slashed (multiplicative decrease), and the cycle repeats. This is the classic **AIMD** (Additive Increase, Multiplicative Decrease) pattern.

The actual amount TCP can send at any instant is `min(cwnd, rwnd)` — whichever is smaller: network congestion state or receiver's buffer capacity.

## Retransmission — how loss is detected and fixed

Two mechanisms trigger retransmission:

1. **Timeout (RTO — Retransmission Timeout)**: if no ACK arrives within the calculated RTO (based on measured RTT and its variance), the sender assumes the segment was lost and resends it. This is the slow path — cwnd resets aggressively here.
2. **Fast retransmit**: if the sender receives **3 duplicate ACKs** for the same sequence number, it doesn't wait for the timeout — it immediately retransmits the missing segment. Much faster recovery, smaller cwnd penalty.

```
        Seq 1  Seq 2  Seq 3(lost)  Seq 4  Seq 5
Sender: ─────►─────►─────X ────►─────►─────►
Receiver ACKs: ACK2  ACK3   ACK3(dup) ACK3(dup) ACK3(dup)
                                 │
                    3 dup ACKs → sender retransmits Seq 3 immediately
```

## MSS vs MTU

- **MTU (Maximum Transmission Unit)** — the largest frame size the underlying link (Ethernet, typically 1500 bytes) can carry.
- **MSS (Maximum Segment Size)** — the largest chunk of *application data* TCP will put in one segment, calculated as `MTU - IP header - TCP header` (commonly ~1460 bytes for a 1500-byte MTU Ethernet link). Negotiated during the handshake (as a TCP option) so both sides pick a size that avoids IP fragmentation.

Getting this wrong (e.g., across VPN tunnels or overlay networks like VXLAN in Kubernetes CNI) causes silent packet drops or fragmentation — a classic root cause of "works on localhost, breaks in cluster" networking bugs.

## Keepalive

TCP keepalive periodically sends empty/probe segments on an otherwise idle connection to detect if the peer has silently disappeared (crashed, network partition) without a proper FIN — important for long-lived connections like database pools or persistent gRPC/HTTP2 connections, so dead peers get cleaned up instead of held open forever.

## Connection States: TIME_WAIT, CLOSE_WAIT, RST vs FIN

```
              ESTABLISHED
                   │
        ┌──────────┴──────────┐
   (active closer)       (passive closer)
        │                       │
     FIN_WAIT_1              CLOSE_WAIT   ← waiting for local app to call close()
        │                       │
     FIN_WAIT_2               LAST_ACK
        │                       │
     TIME_WAIT ────────────► CLOSED
   (2×MSL wait, then closed)
```

- **TIME_WAIT** — the side that initiated the close waits (typically 2×MSL, ~60s default on Linux) before fully freeing the connection, to guarantee any delayed duplicate packets from the old connection don't get misinterpreted by a new connection reusing the same 4-tuple. High connection-churn services (e.g., short-lived HTTP/1.0-style connections at scale) can exhaust ephemeral ports/sockets from too many TIME_WAIT connections — a real production issue solved with connection reuse (keep-alive), `SO_REUSEADDR`, or wider ephemeral port ranges.
- **CLOSE_WAIT** — the passive side has received a FIN but the *local application* hasn't yet called `close()` on its socket. A pile-up of connections stuck in CLOSE_WAIT is almost always an **application bug** — the app isn't closing sockets after use (classic file-descriptor leak symptom).
- **RST (reset)** — an abrupt, ungraceful termination, sent when: connecting to a port with nothing listening, an app crashes and the OS cleans up mid-connection, or a firewall/security policy actively kills the connection. Unlike FIN, RST discards state immediately — no handshake, no guarantee of data delivery.
- **FIN vs RST**: FIN says "I'm done, but everything so far was valid, let's close politely." RST says "abort now, discard everything" — used for errors, not normal shutdown.

## Interview Q&A

**Why is TCP reliable?**
Because every byte is sequenced, every received byte range is acknowledged, unacknowledged data is retransmitted (via timeout or fast retransmit), and out-of-order segments are buffered and reassembled in the correct order before being handed to the application — none of which IP does on its own.

**How does the sender know a packet was lost?**
Either (a) an RTO timer expires with no ACK, or (b) it receives 3 duplicate ACKs for the same sequence number, both of which indicate the receiver never saw a specific segment.

**How does retransmission happen without the app knowing?**
Retransmission is entirely handled inside the OS kernel's TCP/IP stack — it's invisible to the application, which just sees a slightly slower, but complete and correctly ordered, byte stream.

**Flow control vs congestion control — what's the difference?**
Flow control (rwnd) protects the *receiver's buffer* from overflow. Congestion control (cwnd) protects the *network* from overload. TCP always sends `min(cwnd, rwnd)`.
