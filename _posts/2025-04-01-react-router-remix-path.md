---
title: "React Router and the Remix'ed path"
date: 2025-04-01
description: "CVE-2025-31137 - URL manipulation vulnerability in React Router Express adapter"
categories:
  - research
---

![](/images/remix-url-0.png)

## Introduction

Continuing the momentum, inzo_ and I collaborated again to take a look at Remix, a popular full-stack web framework. During our research, we discovered some interesting behaviors, including a vulnerability in React Router, a library used to manage multi-strategy routing in React applications. React Router is downloaded over 13.2 million times per week and is developed and maintained by Remix Run.

The vulnerability allows URL manipulation through the `Host`/`X-Forwarded-Host` header and affects all users of Remix 2, as well as, more generally, React Router 7, who use the Express adapter. This could potentially lead to several exploits.

## Index

- [Remix and data request, again](#remix-and-data-request-again)
- [_data parameter](#_data-parameter)
  - [Exploitation - CPDoS](#exploitation---cpdos)
- [React Router, Express adapter and a talkative port](#react-router-express-adapter-and-a-talkative-port)
  - [Exploitation - chain for a less picky CPDoS](#exploitation---chain-for-a-less-picky-cpdos)
  - [Exploitation - WAF bypass and escalations](#exploitation---waf-bypass-and-escalations)
- [Security Advisory - CVE-2025-31137](#security-advisory---cve-2025-31137)
- [Conclusion](#conclusion)

## Remix and data request, again

Although our research initially focused on Remix, it led to the discovery of an issue in React Router. During our investigation, we identified several interesting behaviors in Remix, including a URL parameter that, if present, allows the request to be treated as a data request. This request returns a JSON object containing the data transmitted from the server-side.

To transmit data to a route, Remix uses what they call a "loader": a server-side function that allows —during the initial server rendering— to feed the HTML document.

## _data parameter

While analyzing the source code, we quickly came across this interesting piece of code:

If the URL parameter `_data` is present, its value is retrieved and passed to the variable `routeId`. Then, the corresponding data request is retrieved and assigned to the variable `response`, the latter then being returned.

The server naturally behaves differently depending on where and how the URL parameter is used; If it's used on a page that doesn't use a loader the server returns a `400` response JSON object containing an error message.

If this is a page using a loader but no valid value is specified, we receive a `403` response, which is a JSON object containing an error message.

The value of the `_data` parameter was assigned to the `routeId` variable, which is simply the name of the page prefixed by `routes/`. So the correct value for the `/ssr` page (using a loader) is `routes/ssr` allowing us to get a `200` response.

### Exploitation - CPDoS

Now that we know we can tamper with any response from a Remix application and obtain different types of status codes without any cache-control constraints, we have what we need for a potential DoS attack via cache poisoning.

Most cache/CDN systems rely -among other things- on the path during content-negotiation. Being able to force the JSON object to be rendered on any route by adding the URL parameter allows to cache the response containing the data object instead of the normal page, completely impacting its availability.

This behavior is similar to what was previously found on Nuxt and Next.js.

## React Router, Express adapter and a talkative port

While React Router is at the heart of Remix, it can be used in various environments, and there are several adapters to ensure it works correctly and consistently across different contexts. The one affected by the vulnerability in our case is the Express adapter.

It is possible to call any path directly from the `host` or `x-forwarded-host` header due to the lack of port sanitization. Although it is possible to take advantage of both headers, we will focus on `x-forwarded-host` because manipulating the `Host` value will result in an error with most reverse proxies/CDNs.

The code retrieves the two values, splits them to retrieve the part after the colon character (`:`), so traditionally, the "port". The value of `hostnamePort` (`x-forwarded-host`) has priority over `hostPort` (since the values are evaluated from left to right, the first truthy value is chosen) and is assigned to the `port` variable.

The `port` value is directly concatenated - without sanitization/verification - to `req.hostname` then assigned to the variable `resolvedHost`. The `resolvedHost` variable is concatenated - again, without sanitization/verification - with the protocol and the current path to create the URL object.

So this allows us to call a path this way, since the part before the column character is not used, we omit it and place the desired path directly.

### Exploitation - chain for a less picky CPDoS

If the target - in addition to using React Router and its Express adapter - uses Remix, we can chain this with the previously mentioned behavior to bypass the "URL parameter not in the cache-key" condition making DoS via cache-poisoning much easier to exploit.

We specify a path using a loader directly after the column character `:` (to get a `200` response), `/ssr` in our case, then we add the URL parameter `?_data` as well as the corresponding `routeId`.

### Exploitation - WAF bypass and escalations

It is also possible to bypass any firewall, whether for a reflected XSS, an SQLi (whose payload would be in the path or URL parameter), or simply to access paths normally prohibited.

We bypass it by splitting the payload — in this case, for an SQLi — into two parts, taking advantage of the fact that the part in the URL is the last to be concatenated (as explained earlier) to form the final URL, and therefore has no impact on routing.

Additionally, a Reflected XSS can be escalated to a Stored XSS if a caching system is present.

## Security Advisory - CVE-2025-31137

@react-router/express (npm) -> Affected versions: 7.0.0-7.4.0 -> Patched versions: 7.4.1
@remix-run/express (npm) -> Affected versions: >=2.11.1 -> Patched versions: 2.16.3

CVSS Score: `CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H` (High/7.5)

## Conclusion

As explained in this brief paper, this vulnerability can be exploited in several ways, either directly or indirectly if chained with other exploits. React Router is no exception; downloaded over 13 million times per week, we found several impacted sites as part of bug bounty programs.

All of these sites were very popular and widely used globally, and the CPDoS aspect alone could render them completely unusable.

**Timeline:**

- 2025/03/26: Report sent by email
- 2025/03/26: Fix implemented
- 2025/03/28: Release of a new version (v2.16.3) containing the fix
- 2025/04/01: Security advisory/CVE-2025-31137

Research conducted by zhero; & inzo_
