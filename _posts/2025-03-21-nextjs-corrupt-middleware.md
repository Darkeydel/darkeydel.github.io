---
title: "Next.js and the corrupt middleware: the authorizing artifact"
date: 2025-03-21
description: "CVE-2025-29927 - Critical middleware bypass vulnerability in Next.js"
categories:
  - research
---

## Introduction

Recently, Yasser Allam, known by the pseudonym inzo_, and I decided to team up for some research. We discussed potential targets and chose to begin by focusing on Next.js — a framework with 130K stars on GitHub, currently downloaded +9.4 million times per week.

Next.js is a comprehensive JavaScript framework based on React, packed with numerous features — the perfect playground for diving into the intricacies of research.

It didn't take long before we uncovered a great discovery in the middleware. The impact is considerable, with all versions affected, and no preconditions for exploitability.

## The Next.js middleware

Middleware allows you to run code before a request is completed. Then, based on the incoming request, you can modify the response by rewriting, redirecting, modifying the request or response headers, or responding directly.

Next.js, being a complete framework, has its own middleware — an important and widely used feature. Its use cases are numerous, but the most important ones include:

- Path rewriting
- Server-side redirects
- Adding elements such as headers (CSP, etc.) to the response
- And most importantly: Authentication and Authorization

## The authorizing artifact: old code, 0ld treasure

While browsing an older version of the framework (v12.0.7), we came across this piece of code:

When a Next.js application uses a middleware, the `runMiddleware` function is used, the latter retrieves the value of the `x-middleware-subrequest` header and uses it **to know if the middleware should be applied or not**. The header value is split to create a list using the column character (`:`) as a separator and then checks if this list contains the `middlewareInfo.name` value.

This means that if we add the `x-middleware-subrequest` header with the correct value to our request, the middleware — whatever its purpose — **will be completely ignored**, and the request will be forwarded via `NextResponse.next()` and will complete its journey to its original destination **without the middleware having any impact/influence on it**.

For our "universal key" to work, its value must contain `middlewareInfo.name`, which is simply the path in which the middleware is located.

### Execution order and middlewareInfo.name

Versions prior to 12.2 allowed nested routes to place one or more `_middleware` files anywhere in the tree (starting from the `pages` folder). The payload for older versions:

```
x-middleware-subrequest: pages/_middleware
```

### Modern versions

Starting with version 12.2, the file no longer contains underscores and must simply be named `middleware.ts`. The payload:

```
x-middleware-subrequest: middleware
```

Or if using `/src` directory:

```
x-middleware-subrequest: src/middleware
```

### Max recursion depth

On more recent versions, the logic has changed. The value of the constant depth **must be greater or equal than the value of the constant** `MAX_RECURSION_DEPTH` (which is `5`):

```
x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware
```

or

```
x-middleware-subrequest: src/middleware:src/middleware:src/middleware:src/middleware:src/middleware
```

## Exploits

### Authorization/Rewrite bypass

With our "authorizing artifact", we can **completely bypass the middleware**, and therefore any protection system based on it, starting with authorization.

### CSP bypass

We can also bypass middleware that sets CSP headers and cookies.

### DoS via Cache-Poisoning

A cache-poisoning DoS is also possible via this vulnerability. If a site rewrites its users' paths based on their geographic location, and we bypass the middleware, we consequently **avoid the rewrite** and we end up on the root page. If the site has a cache/CDN system, it may be possible to **force the caching** of a `404` response, rendering its pages unusable.

## Security Advisory - CVE-2025-29927

**Patches**

- For Next.js 15.x, this issue is fixed in 15.2.3
- For Next.js 14.x, this issue is fixed in 14.2.25
- For Next.js versions 11.1.4 thru 13.5.6 we recommend consulting the below workaround.

**Workaround**

If patching to a safe version is infeasible, we recommend that you prevent external user requests which contain the `x-middleware-subrequest` header from reaching your Next.js application.

**Severity**

`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N` (Critical 9.1/10)

## Conclusion

As highlighted in this paper, this vulnerability has been present for several years in the Next.js source code, evolving with the middleware and its changes over the versions.

**Timeline:**

- 02/27/2025: Vulnerability reported to the maintainers
- 03/01/2025: Second email sent explaining that all versions were ultimately vulnerable
- 03/18/2025: Report accepted, patch implemented
- 03/21/2025: Publication of the security advisory

Research conducted by zhero; & inzo_
