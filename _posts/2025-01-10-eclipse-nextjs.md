---
title: "Eclipse on Next.js: Race Condition Exploitation"
date: 2025-01-10
categories:
  - Security Research
  - Next.js
  - Race Condition
  - Cache Poisoning
---

Racing Next.js's response-cache batcher by forcing disparate failing requests to collide on a shared error cache-key, leaking a transient pageProps HTML variant.

## The Vulnerability

Building on previous Next.js cache poisoning research, this technique exploits race conditions in the response-cache batcher.

## Exploitation

1. Force multiple failing requests to collide on error cache-key
2. Leak transient pageProps HTML variant
3. Externally cache for poisoning-to-SXSS despite prior fixes

## References

- [Original Research](https://zhero-web-sec.github.io/research-and-things/eclipse-on-nextjs-conditioned-exploitation-of-an-intended-race-condition)
