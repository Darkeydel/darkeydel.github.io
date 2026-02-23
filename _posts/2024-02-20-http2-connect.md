---
title: Playing with HTTP/2 Connect
date: 2024-02-20
categories:
  - Research
  - HTTP/2
  - Security
---

In HTTP/1, the CONNECT method instructs a proxy to establish a TCP tunnel to a requested target. Once the tunnel is up, the proxy blindly forwards raw traffic in both directions. This mechanism is most commonly used to tunnel TLS traffic through forwarding proxies.

While digging through the HTTP/2 specification (RFC 9113), I noticed it also features the CONNECT method but with a slight twist. Unlike its predecessor, HTTP/2 CONNECT doesn't hijack the entire TCP connection - it operates on a single stream. This subtle difference piqued my curiosity.

![HTTP/1 CONNECT Tunnel](/images/http2-connect-v1.png)

---

## HTTP/1 CONNECT: The Classic Approach

In HTTP/1, CONNECT transforms the proxy into a blind TCP forwarder:

```
┌──────────┐         ┌──────────┐         ┌──────────┐
│  Client  │────────▶│  Proxy   │────────▶│  Target  │
│          │         │          │         │   :443   │
└──────────┘         └──────────┘         └──────────┘
    │                   │                    │
    │ CONNECT google.de:443                  │
    │─────────────────▶│                    │
    │               200 OK                  │
    │◀────────────────│                    │
    │                   │                    │
    │    TLS Handshake │                    │
    │─────────────────▶│─────────────────▶│
    │                   │                    │
    │      Encrypted Traffic (RAW BYTES)     │
    │═══════════════════════════════════════│
```

### A typical HTTP/1 CONNECT request

```http
CONNECT google.de:443 HTTP/1.1
Host: google.de:443
User-Agent: curl/8.5.0
```

**Key characteristics:**
- The entire TCP connection becomes a tunnel
- The proxy has **no visibility** into encrypted traffic
- Blocks all other requests while tunnel is active
- Used for HTTPS through corporate proxies

> "The most common form of HTTP tunneling is the standardized HTTP CONNECT method. In this mechanism, the client asks an HTTP proxy server to forward the TCP connection to the desired destination." — [Wikipedia](https://en.wikipedia.org/wiki/HTTP_tunnel)

---

## HTTP/2 CONNECT: The Stream-Based Approach

The key difference in HTTP/2 is that CONNECT creates a **stream** rather than a full connection, allowing for more granular control over tunneled connections.

![HTTP/2 CONNECT Stream](/images/http2-connect-v2.png)

### HTTP/2 CONNECT Request

```http
:method CONNECT
:authority google.de:443
:scheme https
:path /connect
```

### Key Differences

| Feature | HTTP/1 CONNECT | HTTP/2 CONNECT |
|---------|----------------|----------------|
| **Scope** | Full TCP connection | Single stream |
| **Multiplexing** | No (blocks other requests) | Yes (multiple tunnels) |
| **Control** | Binary all-or-nothing | Per-stream control |
| **Frame Type** | Raw bytes | HTTP/2 frames |

---

## How HTTP/2 CONNECT Works

### Stream States

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Idle    │────▶│  Open    │────▶│  Half    │
│          │     │          │     │  Closed   │
└──────────┘     └──────────┘     └──────────┘
                      │                   │
                      ▼                   ▼
                ┌──────────┐       ┌──────────┐
                │  Closed  │◀──────│  Half    │
                │          │       │  Closed   │
                └──────────┘       └──────────┘
```

### CONNECT Frame Types

| Frame | Purpose |
|-------|---------|
| **HEADERS** | Initial CONNECT request |
| **DATA** | Tunnel payload |
| **RST_STREAM** | Close tunnel |
| **WINDOW_UPDATE** | Flow control |

---

## Security Implications

### The Good ✅

1. **Better Isolation**: Each CONNECT tunnel is a separate stream, improving containment
2. **No Connection Hijacking**: Other streams remain unaffected
3. **Frame-Level Control**: Can apply flow control per tunnel
4. **Multiplexed Tunnels**: Multiple tunnels on single connection

### The Attack Surface ⚠️

1. **Stream Confusion**: Different interpretations of stream boundaries
2. **Protocol Downgrade**: Exploiting CONNECT to bypass security controls
3. **Proxy Confusion**: HTTP/2 proxies may handle CONNECT differently than HTTP/1
4. **Tunnel Escaping**: Breaking out of tunnel isolation

---

## Practical Use Cases

### 1. HTTPS Proxying Through Forward Proxy

```http
CONNECT internal-api.company.com:443 HTTP/2
Host: internal-api.company.com:443
Authorization: Basic dXNlcjpwYXNz
```

### 2. WebSocket Over HTTP/2

HTTP/2 CONNECT can establish WebSocket-like tunnels:

```http
CONNECT wss://chat.company.com/socket HTTP/2
Protocol: websocket
```

### 3. Kubernetes Service Mesh

HTTP/2 CONNECT is used in service meshes (like Istio Ambient mode) for transparent traffic interception:

> "The HTTP/2 CONNECT method is a core technology for creating tunnels that enable transparent traffic interception and forwarding." — [Tetrate Blog](https://tetrate.io/blog/implementing-http-2-connect-tunnels-with-envoy-concepts-and-practical-guide)

---

## Interesting Behaviors

### Header Compression Differences

HTTP/2 uses HPACK for header compression. The CONNECT method headers are compressed differently:

- `:method`: CONNECT
- `:authority`: target:port
- `:scheme`: always `https` for tunnels

### Push Promises Don't Apply

Unlike regular streams, CONNECT streams **cannot have push promises** — they're exclusively bidirectional.

### No Stream Priority

CONNECT streams have different priority handling than regular request streams.

---

## Debugging HTTP/2 CONNECT

### Wireshark Filters

```bash
# Filter HTTP/2 CONNECT
http2.stream == 1 && http2.type == 0xa

# Show all CONNECT tunnels
http2.headers.method == "CONNECT"
```

### nghttp2 Commands

```bash
# Start HTTP/2 server
nghttpd -v 8080

# Make CONNECT request
curl -v --http2-prior-knowledge https://example.com
```

---

## HTTP/2 CONNECT vs Modern Extensions

| Protocol | Description |
|----------|-------------|
| **CONNECT** | Original TCP tunneling |
| **CONNECT-UDP** | UDP datagram proxying (RFC 9298) |
| **CONNECT-IP** | IP-level tunneling (RFC 9483) |
| **MASQUE** | UDP/IP encapsulation over HTTP |

---

## References

- [RFC 9113 - HTTP/2](https://www.rfc-editor.org/rfc/rfc9113)
- [RFC 7231 - HTTP/1.1 CONNECT](https://www.rfc-editor.org/rfc/rfc7231)
- [HTTP tunnel - Wikipedia](https://en.wikipedia.org/wiki/HTTP_tunnel)
- [HTTP CONNECT (IANA)](https://www.iana.org/assignments/http2-parameters/http2-parameters.xhtml)
- [Implementing HTTP/2 CONNECT Tunnels - Tetrate](https://tetrate.io/blog/implementing-http-2-connect-tunnels-with-envoy-concepts-and-practical-guide)
- [A Primer on Proxies - Cloudflare](https://blog.cloudflare.com/a-primer-on-proxies/)
- [HTTP/2 Stream Visualizer](https://toolkit.whysonil.dev/tools/simulators/http2-streams)
