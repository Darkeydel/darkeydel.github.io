---
title: "Next.js, cache, and chains: the stale elixir"
date: 2025-01-01
description: "CVE-2024-46982 - Cache Poisoning to Stored XSS in Next.js"
categories:
  - research
---

## Introduction

Some time after publishing my previous research on Next.js, I was left with a feeling of unfinished business. That work had sparked my curiosity, and I sensed that this framework still had more secrets to unveil.

It turned out to be a good decision. The findings from this new research have had a significant impact on the ecosystem, and their application in the context of bug bounty programs has been remarkable. This led to numerous reports being submitted, cumulatively amounting to a beautiful six-figure sum in bounties.

This article will focus on the highly popular Next.js — an open-source JavaScript framework based on React, developed and maintained by Vercel.

## Data request

Before diving into the heart of the matter, it's necessary to take a brief detour to understand the role of two Next.js functions that are crucial for what's to come.

### getStaticProps - SSG (Static Site Generation)

If you export a function called `getStaticProps` from a page, Next.js will pre-render this page at build time using the props returned by getStaticProps.

It's quite straightforward: the function simply allows you to transmit data already available during the build process, which is, by its nature, meant to be publicly cached.

### getServerSideProps - SSR (Server-Side Rendering)

`getServerSideProps` is a Next.js function that can be used to fetch data and render the contents of a page at request time.

Unlike `getStaticProps`, `getServerSideProps` transmits data that is only available at the time of requests, based on factors such as the user's data who made the request: cookies, headers, URL parameters, etc.

### Data fetching

When using either of these functions (whether for SSG or SSR), Next.js employs specific routes for data fetching. These routes follow this pattern: `/_next/data/{buildID}/targeted-page.json`.

The response is a JSON object named `pageProps` containing the transmitted data.

## Internal URL parameter and pageProps

In my previous research, I noticed that some internal "components" of the framework were not hermetic to the outside world as was the case for the various headers.

As I revisited the Next.js source code, I focused specifically on its internal operations, with the goal of finding ways to influence its behavior. I came across this particularly interesting piece of code.

The name of the constant as well as the comments clearly indicate its purpose: its value is a boolean that determines whether or not the request is a data request. For the request to be classified as such:

1. Either the URL parameter `__nextDataReq` must be present in the request OR the request must contain the header `x-nextjs-data` along with a specific server configuration
2. It must be an SSG (Static Site Generation) request OR `hasServerProps` must return `true`

Thus, by sending a request to an endpoint that uses `getServerSideProps` and appending the `__nextDataReq` URL parameter, the server should return the expected JSON object instead of the HTML page.

### Exploitation - DoS via Cache Poisoning

Typically, to exploit cache poisoning, we leverage the presence of URL parameters in the cache-key. This not only prevents interference with the client's site during testing, but more importantly, allows us to reset the cache to check whether changes in the request affect the response.

In our case, to trigger a DoS attack via cache poisoning by altering the content of different endpoints on the target site through the JSON object `pageProps`, the target site must have a caching system, and the URL parameters must not be part of the cache-key.

### Real world exploitation (Bug Bounty)

Many sites were vulnerable, this was almost always the case when URL parameters were not included in the cache-key and the site was not hosted by Vercel.

To address this, we can check if the `Accept-Encoding` header is part of the cache-key. If it is, we can send the malicious request without the `Accept-Encoding` header, allowing us to check if the site is vulnerable without impacting it.

## CVE-2024-46982: The stale elixir

**Important note**: the cache affected in this section is the framework's internal caching mechanism. Consequently, the exploits presented below do not depend on the presence of an external caching layer, which significantly increases the severity of the issue.

It all starts with this particularly interesting conditional statement, which, when it returns `true` considers the request to be an SSG.

Since the data transmitted via `getServerSideProps` is dynamic, it is not intended to be cached. This differs from SSG requests, which handle static data. Therefore, it's no surprise how the framework manages `Cache-Control`:

**Default SSR Cache-Control**: `Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate`

**Default SSG Cache-Control**: `Cache-Control: s-maxage=31536000, stale-while-revalidate`

With the basics laid out, let's now go back to our piece of code. If we could have `true` in our first part of the conditional structure, the `isSSG` variable would be set to `true`, indicating to the framework that this is an SSG request, and as previously stated, SSG requests are cacheable. This would therefore allow a Server Side Rendering request to be passed off as a Server Static Generation request, forcing its caching.

The `if` block that interests us contains two `OR` operators, among the three conditions, one of them immediately catches the eye: `req.headers['x-now-route-matches']`

It would therefore be sufficient for the header to be present in the request to achieve our goal.

### Exploitation - DoS via Cache Poisoning

Now that we can cache a request containing SSR data, we need to exploit it and it's time to bring out the `__nextDataReq` card.

By combining in the request:
1. The internal URL parameter `__nextDataReq` to make it a data request
2. The header `x-now-route-matches` to make it pass for an SSG thereby changing the `Cache-Control`

It is possible to cache the JSON object `pageProps` on the target endpoint altering the content of any SSR page.

### Exploitation - Stored XSS via Cache Poisoning

Now, the sharpest among you are surely wondering: since we're now accessing the "normal" page without artificially modifying it, what about the `content-type`? It should no longer be a `application/json` response. Looking at the poisoned response gave me a good shot of dopamine, when we directly access the poisoned endpoint the `content-type` is `text/html`!

This means that any value of the request being reflected in the response is a vector for a SXSS. As explained before, developers very often use `getServerSideProps` to transmit information from the user request to the page: user-agent, CSRF token, cookies, headers, URL parameters, etc.

So for an SXSS to be possible, it only takes one element to be reflected.

### Exploitation - Another way

Sending a request to the data fetch route by adding the `x-now-route-matches` header leads to poisoning the target page endpoint.

## Security Advisory

To be potentially affected all of the following must apply:
- Next.js between 13.5.1 and 14.2.9
- Using pages router
- Using non-dynamic server-side rendered routes

The below configurations are unaffected:
- Deployments using only app router
- Deployments on Vercel are not affected

## Conclusion

The Next.js framework is currently downloaded nearly 6 million times weekly, making it one of the most popular JavaScript frameworks today. Many sensitive platforms rely on it, meaning that vulnerabilities of this nature can have devastating consequences.

On my side, I was able to send many reports relating to these vulnerabilities, whose severity is between High and Critical depending on the possibility or not to chain the CP to a SXSS.

Research conducted by zhero;

*Published in January 2025*
