---
title: "Next.js Cache Poisoning: The Stale Elixir (CVE-2024-46982)"
date: 2025-01-15
categories:
  - Security Research
  - Cache Poisoning
  - Next.js
  - CVE
---

While standalone web cache poisoning is well-understood, internal cache poisoning remains an overlooked and distinctly scary variant. This research discovered critical vulnerabilities in Next.js that allow attackers to poison the framework's internal cache.

## Introduction

Next.js is an open-source JavaScript framework based on React, developed and maintained by Vercel. With nearly 6 million weekly downloads, it's one of the most popular JavaScript frameworks.

This research discovered vulnerabilities that led to:
- DoS via Cache Poisoning
- Stored XSS via Cache Poisoning
- Multiple bug bounty rewards totaling six figures

## Data Request Methods

### getStaticProps - SSG (Static Site Generation)

If you export a function called `getStaticProps` from a page, Next.js will pre-render this page at build time using the props returned.

### getServerSideProps - SSR (Server-Side Rendering)

`getServerSideProps` fetches data and renders a page at request time, based on user-specific data like cookies, headers, and URL parameters.

### Data Fetching Routes

Next.js uses specific routes for data fetching: `/_next/data/{buildID}/targeted-page.json`

The response is a JSON object named `pageProps` containing the transmitted data.

## Internal URL Parameter and pageProps

By sending a request to an endpoint that uses `getServerSideProps` and appending the `__nextDataReq` URL parameter, the server returns the expected JSON object instead of the HTML page.

This can be exploited for DoS via cache poisoning when:
- The target site has a caching system
- URL parameters are NOT part of the cache-key

## CVE-2024-46982: The Stale Elixir

The cache affected is **the framework's internal caching mechanism**. This means exploits don't depend on an external caching layer.

### The Vulnerability

The issue stems from how Next.js handles the `x-now-route-matches` header. By adding this header to a request, an attacker can make the framework treat an SSR (Server-Side Rendering) request as SSG (Static Site Generation).

**Default SSR Cache-Control:** `private, no-cache, no-store, max-age=0, must-revalidate`

**Default SSG Cache-Control:** `s-maxage=31536000, stale-while-revalidate`

By sending the `x-now-route-matches` header, the `Cache-Control` changes to `s-maxage=1, stale-while-revalidate`, enabling caching!

### Exploitation - DoS via Cache Poisoning

By combining:
1. The internal URL parameter `__nextDataReq` to make it a **data request**
2. The header `x-now-route-matches` to make it pass for an **SSG**

Attackers can cache the JSON object `pageProps` on the target endpoint, altering the content of any SSR page.

### Exploitation - Stored XSS via Cache Poisoning

When accessing the poisoned endpoint directly, the `content-type` is `text/html`! This means any value reflected in the response becomes a vector for Stored XSS.

Common reflected elements:
- Cookie/header value about language preferences
- Session cookie/uid/anonymousId
- User-agent
- CSRF header
- Theme/color preference

## Affected Versions

- Next.js between 13.5.1 and 14.2.9
- Using Pages Router
- Using non-dynamic server-side rendered routes

### Unaffected:
- Deployments using only App Router
- Deployments on Vercel

## Security Advisory

- [GitHub Advisory](https://github.com/advisories/GHSA-gp8f-8m3g-qvj9)
- [Vercel Advisory](https://github.com/vercel/next.js/security/advisories/GHSA-gp8f-8m3g-qvj9)

## Conclusion

This research demonstrates that internal cache poisoning in frameworks like Next.js can have devastating consequences. The vulnerabilities affected multiple sectors including:
- Money transfer platforms
- Cryptocurrency platforms
- E-commerce platforms

Most reports resulted in 5-digit bounties.

## References

- [Original Research - Rachid Allam](https://zhero-web-sec.github.io/research-and-things/nextjs-cache-and-chains-the-stale-elixir)
- [PortSwigger Top 10 2025](https://portswigger.net/research/top-10-web-hacking-techniques-of-2025)
