# UDP — When Speed Matters More Than Guarantees

## What UDP is (and isn't)

UDP (User Datagram Protocol) is TCP's opposite philosophy: **connectionless, unordered, unacknowledged, best-effort delivery**. There's no handshake, no sequence numbers for reordering, no retransmission, and no congestion control built into the protocol itself.

```
TCP:  Handshake → Ordered, reliable stream → Graceful close  (heavyweight, guaranteed)
UDP:  Just send the datagram.                                (lightweight, no guarantee)
```

UDP header is tiny (just 8 bytes: source port, dest port, length, checksum) versus TCP's minimum 20 bytes plus all the state the OS must track (windows, timers, retransmit queues).

```
UDP Header (8 bytes)
┌──────────────┬──────────────┬────────┬──────────┐
│ Source Port  │  Dest Port   │ Length │ Checksum │
└──────────────┴──────────────┴────────┴──────────┘
        followed immediately by the payload
```

## Why and when UDP is better than TCP

The core tradeoff: **TCP guarantees correctness at the cost of latency (retransmission waits); UDP guarantees speed at the cost of correctness (packets can be lost or arrive out of order and nothing fixes that automatically).**

UDP wins when:
- **A late packet is worse than a lost packet.** In a live video call, a retransmitted 200ms-old audio frame is useless — you'd rather skip it than pause the whole call waiting for it (TCP would pause/buffer to preserve order).
- **The application can tolerate loss or handle reliability itself at a higher layer.** QUIC (HTTP/3) is built on UDP specifically so it can implement its *own* smarter reliability/congestion logic without being bound by the OS kernel's rigid TCP implementation.
- **Overhead matters more than guarantees**, e.g., tiny, frequent messages where the 3-way handshake cost would dominate.
- **One-to-many delivery** — UDP supports multicast/broadcast patterns that a connection-oriented protocol like TCP fundamentally cannot.

## Classic real-world examples

| Use case | Why UDP fits |
|---|---|
| **DNS** | A query/response is one tiny round trip; if it's lost, just retry the whole query — cheaper than TCP's handshake overhead for such a small exchange. (DNS does fall back to TCP for large responses, e.g., zone transfers or responses > 512 bytes/with EDNS0 limits, or when TC bit is set.) |
| **VoIP (SIP/RTP)** | Voice needs to be delivered *live*. A dropped audio packet causes a tiny click; waiting for its retransmission would cause a much worse stutter/delay. This directly matches the SIP/RTP-over-UDP/WebRTC work you've done with LiveKit + Stunner — NAT traversal (STUN/TURN) exists precisely because UDP has no built-in connection state to help with that. |
| **Video streaming (live)** | Similar to VoIP — a skipped frame is preferable to a frozen stream waiting for retransmission. (Note: on-demand streaming like Netflix/YouTube VOD actually often uses TCP-based HTTP for reliable chunked delivery since it's buffered, not live — UDP shines specifically for *real-time* streams.) |
| **Gaming** | Player position updates are sent many times per second; an old, retransmitted position update is worse than useless — it's actively wrong. Games send fresh state repeatedly and simply ignore/interpolate over any gaps. |
| **Metrics/Telemetry** (e.g., StatsD, some Prometheus pushgateway patterns, syslog) | Losing one metric sample out of thousands doesn't matter, and the overhead of a reliable connection per sample would be wasteful at high volume/frequency. |

## What UDP does NOT give you (and who compensates)

| Missing guarantee | Who fixes it, if anyone |
|---|---|
| Ordering | Application layer (e.g., RTP sequence numbers let VoIP/video reorder or detect gaps, but don't force retransmission) |
| Reliability/retransmission | Either nobody (metrics, live video — loss is accepted), or the app layer re-implements it (QUIC/HTTP3 has its own retransmission logic) |
| Congestion control | Modern UDP-based protocols like QUIC implement their own (BBR-style) congestion control in userspace, since the kernel won't do it for raw UDP |
| Flow control | Same — must be application-implemented if needed |

## Interview one-liners

- **"Why does DNS use UDP?"** — Queries are small and a single round trip; retry-on-loss is cheaper than paying for a TCP handshake for every lookup. Falls back to TCP only when the response won't fit in a single UDP datagram or truncation occurs.
- **"Why is UDP used for live video/voice but not for downloading a file?"** — Live media favors *timeliness* over *completeness* (a late packet is worthless); a file download favors *completeness* over speed (a missing byte corrupts the file), so TCP's guarantees are worth the latency cost there.
- **"Is UDP 'unreliable' a bad thing?"** — Not inherently — it's a deliberate design choice that hands reliability decisions to the application instead of forcing one-size-fits-all guarantees, which is exactly why QUIC/HTTP3 chose to build on top of UDP rather than TCP.
