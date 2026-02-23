---
title: Playing with HTTP/2 Connect
date: 2024-02-20
categories:
  - Research
  - HTTP/2
  - Security
---

In HTTP/1, the [`CONNECT`](https://datatracker.ietf.org/doc/html/rfc9110#name-connect) method instructs a proxy to establish a TCP tunnel to a requested target. Once the tunnel is up, the proxy blindly forwards raw traffic in both directions. This mechanism is most commonly used to tunnel TLS traffic through forwarding proxies.

While digging through the HTTP/2 specification ([RFC 9113](https://datatracker.ietf.org/doc/html/rfc9113)), I noticed it also features the `CONNECT` method but with a slight twist. Unlike its predecessor, HTTP/2 `CONNECT` doesn't hijack the entire TCP connection — it operates on a single stream. This subtle difference piqued my curiosity.

---

## HTTP/1 CONNECT: The Classic Approach

A typical HTTP/1 `CONNECT` request is straightforward:

```http
CONNECT google.de:443 HTTP/1.1 
Host: google.de:443 
User-Agent: curl/8.5.0 
```

Once the server responds with `HTTP/1.1 200 OK`, the tunnel is ready. In essence, HTTP/1 `CONNECT` upgrades the entire TCP connection. Raw data is forwarded from that point on, making it seem as if the client is connected directly to the target.

---

## HTTP/2 CONNECT: Stream-Based Approach

HTTP/2 is a binary protocol where the fundamental protocol unit is the frame. Every frame carries a stream identifier:

![HTTP/2 Frame Structure](/images/http2-connect-v1.png)

This stream identifier is used to simulate the HTTP/1 request and response pattern by associating each interaction with a unique stream. **Streams allow multiplexing**, which means a single HTTP/2 connection can host multiple, simultaneous `CONNECT` tunnels. The raw data for each tunnel is then encapsulated within `DATA` frames on its respective stream.

---

## Building a Scanner in Go

To experiment with this, let's build a scanner for misconfigured proxies that allow connections to internal targets, leveraging multiplexing to efficiently scan a range of internal ports over a single TCP connection.

### Establishing the Connection

First, we establish a raw TCP or TLS connection to the proxy. For TLS, we must negotiate HTTP/2 using ALPN:

```go
conn, _ := tls.Dial("tcp", url.Host, &tls.Config{
    InsecureSkipVerify: true, 
    NextProtos:         []string{"h2"},
})
```

### HTTP/2 Connection Preface

To initiate the HTTP/2 connection, the client must send the connection preface (`PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`) followed by a `SETTINGS` frame:

```go
conn.Write([]byte(http2.ClientPreface))
framer := http2.NewFramer(conn, conn)
framer.WriteSettings(http2.Setting{})
```

### Processing Frames

The `http2.Framer` handles low-level frame parsing. Our main logic reads and processes incoming frames:

```go
for {
    f, _ := framer.ReadFrame()
    switch f := f.(type) {
    case *http2.DataFrame:
        // Handle incoming data for a tunnel
    case *http2.MetaHeadersFrame:
        // Handle response headers for CONNECT request
    case *http2.GoAwayFrame:
        return // Connection is closing
    case *http2.SettingsFrame:
        // Received SETTINGS frame
    case *http2.RSTStreamFrame:
        // Stream error - port closed
    }
}
```

### Creating a CONNECT Tunnel

To create a tunnel, we send a `HEADERS` frame with the `CONNECT` method:

```go
hpackEncoder := hpack.NewEncoder(&hpackBuf)
streamID := uint32(1) // Odd-numbered for client-initiated

connectHeaders := []hpack.HeaderField{
    {Name: ":method", Value: "CONNECT"},
    {Name: ":authority", Value: targetAddress},
}

for _, hf := range connectHeaders {
    _ = hpackEncoder.WriteField(hf)
}

framer.WriteHeaders(http2.HeadersFrameParam{
    StreamID:      streamID,
    EndHeaders:    true,
    EndStream:     false,
    BlockFragment: hpackBuf.Bytes(),
})
```

If successful, the proxy responds with `HEADERS` containing `:status 200`. From then on, `DATA` frames on that stream are forwarded to the target.

### Wrapping as net.Conn

We can wrap the tunnel in a `net.Conn` interface for easy use:

```go
type tunnelConn struct {
    proxyConn *proxyConn 
    streamID  uint32
    rxChan    chan []byte
}

func (t *tunnelConn) Write(b []byte) (n int, err error) {
    err = t.proxyConn.framer.WriteData(t.streamID, false, b)
    return len(b), err
}

func (t *tunnelConn) Read(b []byte) (n int, err error) {
    // Read from channel
}
```

Now we can layer other protocols on top:

```go
tlsClient := tls.Client(tunnel, &tls.Config{
    ServerName:         "example.com",
    InsecureSkipVerify: true,
})
fmt.Fprintf(tlsClient, "GET / HTTP/1.1\r\nHost: %s\r\n\r\n", "example.com")
```

![Go Implementation Flow](/images/http2-connect-v3.png)

---

## Application: Efficient Port Scanning

Building a port scanner is straightforward. Send `CONNECT` requests for various `ip:port` combinations, each on a new stream ID. We don't even need to send `DATA` frames — just monitor response headers:

![Port Scanning via HTTP/2 CONNECT](/images/http2-connect-v2.png)

| Response | Meaning |
|----------|---------|
| `HEADERS` with `:status 200` | Port is open |
| `HEADERS` with `:status 503` | Connection failed |
| `RST_STREAM` with `CONNECT_ERROR` | Port is closed |

By tracking responses for each stream ID, we can map open ports on an internal network — all **multiplexed over a single HTTP/2 connection**. This technique may bypass network security monitoring that isn't equipped to inspect multiplexed HTTP/2 traffic.

---

## Setting Up a Test Environment

Support for HTTP/2 `CONNECT` tunneling is not yet widespread. Reliable implementations exist in **Envoy** and **Apache httpd**.

### Envoy Configuration

```bash
docker run --rm -v ./envoy.yaml:/envoy.yaml:ro envoyproxy/envoy:distroless-v1.35-latest -c /envoy.yaml
```

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10001
    http_filters:
    - name: envoy.filters.http.dynamic_forward_proxy
    - name: envoy.filters.http.router
  http2_protocol_options:
    allow_connect: true
```

### Apache httpd Configuration

```bash
docker run --rm -p 8080:8080 -v ./httpd.conf:/usr/local/apache2/conf/httpd.conf httpd:2.4.65
```

```apache
LoadModule http2_module modules/mod_http2.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_connect_module modules/mod_proxy_connect.so

<VirtualHost *:8080>
    ProxyRequests on
    Protocols h2c
    AllowCONNECT 443 80 8888
</VirtualHost>
```

---

## Extended CONNECT: Beyond TCP

While this post focused on TCP tunneling, the `CONNECT` mechanism is being extended:

### WebSockets (RFC 8441)

```http
:method: CONNECT
:authority: 172.17.0.1:8888
:protocol: websocket
```

### UDP Proxying (RFC 9298)

```http
:method: CONNECT
:authority: dns.example.com:53
:scheme: dns
```

### IP Proxying (RFC 9484)

Raw IP packet tunneling.

---

## Conclusion

HTTP/2's `CONNECT` method transforms a single connection into a conduit for multiple, independent tunnels. This offers a highly efficient way to multiplex connections. Because this traffic is encapsulated within a standard HTTP/2 stream, it may also bypass traditional network inspection tools.

---

## References

- [RFC 9113 - HTTP/2](https://datatracker.ietf.org/doc/html/rfc9113)
- [RFC 9110 - CONNECT Method](https://datatracker.ietf.org/doc/html/rfc9110#name-connect)
- [RFC 8441 - Extended CONNECT for WebSockets](https://datatracker.ietf.org/doc/html/rfc8441)
- [RFC 9298 - CONNECT for UDP](https://datatracker.ietf.org/doc/html/rfc9298)
- [RFC 9484 - CONNECT for IP](https://datatracker.ietf.org/doc/html/rfc9484)
- [Source Code](https://github.com/fl0mb/HTTP2-CONNECT-Tunnel)
