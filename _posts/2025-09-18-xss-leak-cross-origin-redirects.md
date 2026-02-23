---
title: "XSS-Leak: Leaking Cross-Origin Redirects"
date: 2025-09-18
categories:
  - Security Research
  - XS-Leaks
  - Chrome
---

Don't be fooled by the name - XSS-Leak has nothing to do with XSS. This beautiful attack uses Chrome's connection-pool prioritization algorithm as an oracle to leak redirect hostnames cross-domain.

## Introduction

In this post, I will introduce `XSS-Leak` ("Cross-Site-Subdomain Leak"), a technique for Chromium-based browsers that leaks cross-origin redirects, `fetch()` destinations, and more. I use the name `XSS-Leak` because the original goal was to leak subdomains of cross-origin requests.

## Understanding Chrome's Connection Pool

Chrome can run up to 256 concurrent requests across different origins, but only 6 in parallel to the same origin. Chrome schedules by priority first: the request with the highest priority gets a socket. But what if two requests have the same priority?

When two requests have the same priority, Chrome will sort by port, then scheme, then host. If two requests have the same priority, port and scheme, Chrome will prioritize the one whose host is lexicographically lower.

Example: if request A targets `http://google.com:80` and request B targets `http://example.com:80`, `http://example.com:80` wins and gets the socket first (same port and scheme but `example.com` < `google.com`).

This quirk gives us an oracle. We can "guess" the unknown subdomain with a binary search by sending a request to a test host and seeing which request "goes first".

## Challenge Overview

A minimal Express app was given with two routes:
- `/` - serves a page with an inline script
- `/report` - triggers a visit from a headless Chrome bot

The page reads `flag` from localStorage and hex-encodes it, then sends a request to `http://<hex(flag)>.${DOMAIN}:${PORT}`.

## Exploitation

The exploit works as follows:

1. **Exhaust the entire pool** - Use 255 blocking requests
2. **Trigger the challenge's cross-origin request** (priority: High)
3. **Send our request** to `<our-guess><char-to-brute>FFF.attacker.com` (same priority)
4. **Free exactly one socket** and immediately send a request to a host that is lexicographically smaller
5. **Binary search** on the unknown subdomain using this oracle

```javascript
async function test(leak, w, threshold){
    // Exhaust sockets
    await exhaust_sockets();
    
    // Trigger the cross-origin request
    w.location = `${TARGET}#b`;
    await sleep(100);

    // Our leak attempt
    let promise1 = fetch_leak(leak + mid, threshold);
    await sleep(100);
    res_blocker_controller.abort();
    
    // Race condition determines if our host is lower/higher
    let is_lower = !(await promise1);
    
    // Binary search on charset
}
```

## Leaking Redirect Hosts

This technique also leaks where a redirect goes. The idea is:
- Exhaust the connection pool
- Open the auth URL in a window
- Free and re-block the last socket
- Trigger a frame navigation to test host
- Use the same ordering oracle

If the redirect host sorts before our test host, our request is delayed.

## Use Cases

This technique is useful for:
- **Cross-Origin Request Counting**
- **Delay POST requests**
- **Leak the scheme of cross-origin requests**
- **Leak the port of cross-origin requests**
- **Leak redirect destinations**

## Limitations

Only works in Chromium-based browsers.

## References

- [Original Research - Salvatore Abello](https://blog.babelo.xyz/posts/cross-site-subdomain-leak/)
- [XS-Leaks Dev](https://xsleaks.dev)
- [Chrome Connection Pool](https://xsleaks.dev/docs/attacks/timing-attacks/connection-pool/)
