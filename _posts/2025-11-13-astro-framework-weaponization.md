---
title: "Astro framework and standards weaponization"
date: 2025-11-13
description: "CVE-2025-64525 - Multiple vulnerabilities in Astro framework via header manipulation"
categories:
  - research
---

![](/images/astro-midd-1.png)

## Introduction

For this small research, we focus on Astro, a framework gaining momentum with over 800,000 downloads per week and more than 50,000 stars on GitHub at the time of writing. Its technical choices differ from common JS frameworks: while Astro renders components on the server (like Next.js), it prioritizes sending lightweight HTML to the browser with no JavaScript by default, loading JavaScript for interactive components only when needed.

These design choices proved popular, placing Astro third among GitHub's fastest‑growing languages this year, which prompted inzo_ and me to investigate it for exploitable gadgets and whatever may follow.

We will show here how simple, widely known standard request headers, combined with an opportunistic use of the URL parser, can lead to bypassing path‑based middleware protections and enable multiple exploits, ranging from simple SSRF to stored XSS, ending with a complete bypass of a previously disclosed CVE.

## Index

- [URL creation and unwanted guests](#url-creation-and-unwanted-guests)
- [Exploitation: Two vectors, numerous possibilities](#exploitation-two-vectors-numerous-possibilities)
  - [Bypassing path-based middleware protections](#bypassing-path-based-middleware-protections)
    - [WHATWG URL Standard and parser behavior](#whatwg-url-standard-and-parser-behavior)
    - [Final round and bypass](#final-round-and-bypass)
  - [SSRF](#ssrf)
  - [URLs pollution (to SXSS)](#urls-pollution-to-sxss)
  - [WAF bypass](#waf-bypass)
- [CVE-2025-61925 - Complete bypass](#cve-2025-61925---complete-bypass)
- [Security Advisory - CVE-2025-64525](#security-advisory---cve-2025-64525)
- [Conclusion](#conclusion)

## URL construction and unwanted guests

Our story unfolds within the Node adapter's function, `createRequest`, invoked for requests handled by Server-Side Rendering (SSR). When an incoming request arrives, the URL is structured as follows:

```
url = new URL(`${protocol}://${hostnamePort}${req.url}`);
```

A template literals, with three values: one for the **protocol**, one for the **hostname** + **port**, and the other for the **path**. Astro's security advisories indicate that a vulnerability (CVE-2025-61925) was discovered last month involving the `x-forwarded-host` request header, the latter being used without validation to construct the URL above, allowing the hostname to be spoofed and leading to multiple vulnerabilities. We were able to completely bypass the CVE patch in question, as we will see later in this paper.

The issue was addressed by implementing a validation check to ensure that the `x-forwarded-host` header contains only values from a predefined list of allowed domains (`allowedDomains`), a standard whitelisting approach in such cases.

Despite the previous fix, the problem is far from resolved and the worst is yet to come. Inspection of the URL construction process shows that two other standard headers are still employed without validation: `x-forwarded-proto` for the protocol and `x-forwarded-port` concatenated with the `host`:

![](/images/astro-midd-2.png)

The `x-forwarded-proto` header is of primary interest because it is consumed **at the start** of the *template* string. By injecting our payload at the protocol level during URL assembly, **we can effectively rewrite the entire URL**, including `scheme`, `host`, `port`, and `path`, and relocate the original hostname and path into the query component, thereby avoiding influence on routing logic.

Consider the following header and value, added to a request to **https://www.example.com/ssr**:

```
x-forwarded-proto: https://www.malicious-url.com/nope?tank=
```

The complete URL created will be:

```
https://www.malicious-url.com/nope?tank=://www.example.com/ssr
```

Our value is injected at the beginning of the string (`${protocol}`), and ends with a query `?tank=` whose value is the rest of the *template* string: `://${hostnamePort}${req.url}`. This way we have control over the routing without modifying the *real* path, and can manipulate the URL arbitrarily.

## Exploitation: Two vectors, numerous possibilities

Arbitrary modification of the request URL yields multiple attack vectors, from server-side request forgery (SSRF) and SXSS through cache poisoning, to, with a bit more ingenuity, circumvention of path-based middleware protections. The feasibility of some of these attack vectors commonly depends on environmental conditions, such as the presence of a CDN, developers' use of `Astro.url` for constructing links, or the application performing external requests.

## Bypassing path-based middleware protections

Let's start by reviving some good memories with the following exploit, which permits access to routes protected by middleware whose authorization logic relies, among other factors, **on the request path**.

We will base our middleware on the developers' source of truth, the official Astro documentation and use a snippet to protect our `/admin` route.

Unsurprisingly, when we try to access `/admin` *unauthenticated* we are redirected.

Trying to use x-forwarded-[proto/port] with a classic payload by spoofing the path will not work, since once the middleware is reached, the request has already been created, and access to the `url` or `pathname` property will already contain the final path taken into account by the router.

### WHATWG URL Standard and parser behavior

However, since `x-forwarded-proto` seeds the scheme for `new URL(...)`, it effectively grants control over the whole WHATWG URL instance (protocol/host/port/path/query), enabling parser manipulation and exploration of router-accepted edge cases, leading, after some research, to the following interesting payload:

![](/images/astro-midd-5.png)

Before explaining why this payload is relevant to our exploit, let's take a detour to understand what happened at the parser level, based on the WHATWG URL standard specification which operates like a state machine, updating its internal state based on the characters/inputs observed.

As expected, the segment preceding the colon (`:`) is interpreted as the URL scheme. So in our case -> `protocol: 'x:'`

The character after `:` determines whether the parser enters the **authority** or the **path state**; in our case, because the next character is not `/`, the parser goes directly into the **path state**.

However, when using the `http` scheme, it appears to use `admin` as the host and automatically adding `/` as a pathname. The reason is that `http` is recognized as a **special scheme**, which affects how the URL is parsed.

### Final round and bypass

Now that this is clear, let's see why the `x:admin?` payload is interesting in our case.

As we saw earlier in the middleware snippet provided by the Astro documentation, and as is often the case for protected paths in general, a check is performed verifying that the pathname is strictly equal to the protected path (*or start with it*), naturally expecting the path to begin with a slash (`/`):

```javascript
if (context.url.pathname === "/dashboard" && !isAuthed) (...)
```

This is where our payload becomes truly useful, allowing us **to specify a path without a slash** (`/`), while **still maintaining a valid URL for the Astro router and the WHATWG parser**. Being considered valid by the URL parser is obviously not enough, because the router's regex must match in order to land on the desired route, and the router expects it to start with a slash.

Fortunately for us, the pathname passes through the `prependForwardSlash` function before being passed to the router matcher, which adds a slash at the beginning of the string if there isn't one.

![](/images/astro-midd-15.png)

Our pathname was initially `admin` when it reached the middleware layer, but became `/admin` once it reached the router layer. By adding the following payload to our request:

```
x-forwarded-proto: x:admin?
```

We were able to bypass the check because `"admin" != "/admin"`. The question mark `?` marks the start of the query string and absorbs the current path.

## SSRF

As the request URL is built from untrusted input via the `x-forwarded-protocol` header, if it turns out that this URL is subsequently used to perform external network calls, for an API for example, this allows an attacker to supply a malicious URL that the server will fetch, resulting in server-side request forgery (SSRF).

## URLs pollution (to SXSS)

The exploitability of the following depends on the presence of a CDN and therefore corresponds to a cache-poisoning scenario. If the value of `Astro.url` is used to construct links within the page, an attacker can achieve stored XSS by manipulating the `x-forwarded-proto` header.

We need to craft a payload that allows the router to reach the correct path, namely `/links`, while also having valid JavaScript so that the payload executes:

```
x-forwarded-proto: javascript:/links#/;alert('XSS')//
```

This makes it possible to reliably achieve an SXSS when a CDN is present.

## CVE-2025-61925 - Complete bypass

As mentioned earlier, a security advisory was published last month regarding the `X-Forwarded-Host` header. After finishing the writing of this paper, we decided to take a closer look at the patch for the CVE-2025-61925. The patch was bypassed in five minutes flat.

By sending `x-forwarded-host` with an **empty value**, the `forwardedHostname` variable is assigned an empty string. Then, during the subsequent check, the condition fails because `forwardedHostname` returns `false`, its value being **an empty string**.

Consequently, the implemented check is bypassed and the value of `forwardedHostname` is used for URL construction. From this point on, since the request has no `host`, **the path value is retrieved by the URL parser to set it as the host**.

## Security Advisory - CVE-2025-64525

Affected versions: =< 2.16.0

Patched versions: >= 5.15.5

CVSS score: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:L` (Moderate - 6.5)

## Conclusion

The vulnerabilities described in this paper serve as a reminder of the potential impact of seemingly simple, standard headers. This apparently innocuous aspect may explain the historical lack of attention from researchers and maintainers.

**Timeline:**

- 2025-11-03: Report sent via the GitHub template
- 2025-11-03: Report acknowledged and accepted a few minutes later
- 2025-11-04: PR review requested by the Astro team
- 2025-11-04: Review carried out with some feedback
- 2025-11-04: Advisory update following the complete bypass of CVE-2025-61925
- 2025-11-07: Implementation of the recommended changes + bypass fix
- 2025-11-10: Implementation of the latest fixes and release of the patched version - `astro@5.15.5`
- 2025-11-13: Publication of the security advisory

Research conducted by zhero; & inzo_
