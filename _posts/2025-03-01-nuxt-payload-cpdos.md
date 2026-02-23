---
title: "Nuxt, show me your payload - a basic CP DoS"
date: 2025-03-01
description: "CVE-2025-27415 - Cache Poisoning DoS vulnerability in Nuxt"
categories:
  - research
---

## Introduction

During a research, it is relatively important not to lock ourselves too much into a paradigm when we particularly appreciate a type of vulnerability, our reading is no longer objective and this naturally reduces the range of our radar. On the other hand, these biases can sometimes be advantageous, allowing us to perceive potential threats in places that may initially seem harmless for most people.

One such bias helped me identify a small vulnerability during my research on Nuxt, an open-source JavaScript framework used for building full-stack web applications with Vue.js:

The ability to trick the server into rendering the payload on main routes, which, if a caching system is in place, could force the caching of the response on these routes. This could severely impact the application's availability, as the page content would be entirely altered, rendering the site unusable.

## Rendering payload

On Nuxt, payloads are used in order to transmit data from the server-side to the client-side. These can be in JSON (default) or JS format depending on the configuration.

The data is passed to the frontend without requiring additional API requests after the initial page load and injected into the HTML of the page, between the script tags containing the attribute id `__NUXT_DATA__`.

Nuxt also externalizes this data on a specific route that matches the following pattern: `/endpoint/_payload.json` (or `.js`).

## Lax check

Checking the source code a little more closely, it turns out that Nuxt uses a regular expression to check if it is a rendering payload route:

Nothing weird at first glance, if the "path" ends with `/_payload.json` or `/_payload.js` with a possible beginning of query string then the constant `isRenderingPayload` becomes `true`. I use the word "path" but is it really the case?

If it is really the full URL that is being tested instead of the path, then it is possible to trick the check by calling a normal route by adding a query string like: `?poc=/_payload.json`

This will respect the constraints of the regex, and set the value of `isRenderingPayload` to `true`, consequently retrieving the payload.

## Exploit: CP-DoS - Query

Most cache/CDN systems rely -among other things- on the path during content-negotiation. Being able to force the payload to be rendered on a "normal" route by passing the rest of the "path" (`/_payload.json`) as a URL parameter allows to cache the response containing the payload instead of the normal page, completely impacting its availability.

This behavior is similar to what was previously found on Next.js.

### Scope

Interestingly, all pages of a Nuxt application are vulnerable, even the ones not transmitting data from the server to the client, the object will be empty, but still returned, increasing the attack surface for our CP DoS.

## Exploit: CP-DoS - Hash

The requirement that URL parameters not be part of the cache key bothered me, and I pondered how to circumvent this constraint until I came up with the idea of exploiting the hash.

In a URL, the hash portion (`#`) is exclusively processed by the browser and is not transmitted to the server. When a request is sent, the hash portion is therefore not part of the journey.

With the entire URL tested, it was almost certain that sending a request to `#/_payload.json` through the proxy would achieve the same result.

## Security advisory - CVE-2025-27415

**Affected versions**: 3.0.0 < 3.16.0

**Patched versions**: 3.16.0

**Severity**: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H` (High 7.5)

## Conclusion

A simple request can make a site unusable, which can have a significant financial impact depending on the nature of the application (e-commerce, web3, etc.).

Research conducted by zhero;

*Published in March 2025*
