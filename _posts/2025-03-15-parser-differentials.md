---
title: "Parser Differentials: When Interpretation Becomes a Vulnerability"
date: 2025-03-15
categories:
  - Security Research
  - Parser Differential
  - Web Security
---

## Overview

Parser differentials emerge when two (or more) parsers interpret the same input in different ways. This talk explores case studies affecting a broad range of languages, frameworks and technologies.

## Introduction

The term originates from the **Language-theoretic Security (LangSec)** approach, which argues that input handling is the root cause of many security vulnerabilities. When multiple parsers process the same data, they often reach different conclusions—and attackers can exploit these differences.

## Key Concepts

### What Are Parser Differentials?

> Different interpretation of messages or data streams by components breaks any assumptions that components adhere to a shared specification and so introduces inconsistent state and unanticipated computation.

### Why They Matter

- Modern applications use multiple parsers (JSON, XML, SQL, templates)
- Each parser may interpret input differently
- Attackers exploit these gaps for:
  - Authentication bypass
  - Data exfiltration
  - Remote code execution

## Case Studies

### GitLab File Upload Vulnerability

**CVE-2020-**

| Component | What It Sees |
|-----------|--------------|
| **GitLab Workhorse** | `POST` or `PUT` request |
| **GitLab Rails** | Any HTTP method (via `Rack::MethodOverride`) |

By using `X-HTTP-Method-Override: PUT` header, attackers could bypass Workhorse routing!

### CouchDB RCE

**Erlang (jiffy):**
```erlang
jiffy:decode("{\"foo\":\"bar\", \"foo\":\"baz\"}")
% Returns: {[{<<"foo">>,<<"bar">>},{<<"foo">>,<<"baz">>}]}
```

**JavaScript:**
```javascript
JSON.parse("{\"foo\":\"bar\", \"foo\": \"baz\"}")
// Returns: {foo: "baz"}
```

By sending duplicate keys, Erlang stores `["admin"]` while JavaScript validation sees `[]`.

### HTTP Desync Attacks

Front-end and back-end proxies parse HTTP requests differently:
- Different interpretations of Content-Length
- Transfer-Encoding handling discrepancies
- Request smuggling for cache poisoning

## Languages & Technologies Affected

| Category | Examples |
|----------|----------|
| **Web** | HTTP, HTML, URLs, JSON |
| **Data Formats** | XML, YAML, CSV |
| **Template Engines** | Jinja2, ERB, Freemarker |
| **Databases** | SQL, NoSQL |
| **Protocols** | WebSocket, HTTP/2 |

## Detection Techniques

1. **Fuzzing** - Send varied inputs to multiple parsers
2. **Error Differences** - Compare error messages
3. **Output Comparison** - Check for different behaviors

## Mitigation

- Use consistent parsers across components
- Validate input at every boundary
- Implement defense in depth

## References

- [PortSwigger Top 10 2025](https://portswigger.net/research/top-10-web-hacking-techniques-of-2025)
- [LangSec](http://langsec.org)
