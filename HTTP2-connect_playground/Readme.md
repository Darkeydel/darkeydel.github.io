::::::Playing with HTTP/2 Connect:::::::

n HTTP/1, the CONNECT method instructs a proxy to establish a TCP tunnel to a requested target. Once the tunnel is up,
the proxy blindly forwards raw traffic in both directions.
This mechanism is most commonly used to tunnel TLS traffic through forwarding proxies.
While digging through the HTTP/2 specification (RFC 9113),
I noticed it also features the CONNECT method but with a slight twist. Unlike its predecessor,
HTTP/2 CONNECT doesn’t hijack the entire TCP connection, it operates on a single stream. This subtle
difference piqued my curiosity.

##
A typical HTTP/1 CONNECT request is straightforward:


#
CONNECT google.de:443 HTTP/1.1
Host: google.de:443
User-Agent: curl/8.5.0
#

