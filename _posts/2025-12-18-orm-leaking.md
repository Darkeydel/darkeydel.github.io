---
title: "ORM Leaking: More Than You Joined For"
date: 2025-12-18
categories:
  - Security Research
  - ORM
  - SQL Injection
---

Like XS-leaks? ORM leaks are their chunky server-side cousin. This research evolves ORM leaks from niche framework-specific vulnerabilities into a generic methodology.

## Introduction

ORM Leak vulnerabilities allow attackers to filter on sensitive fields that should not be accessible, leaking data through the application's own search/filter functionality.

## Beego ORM - CVE-2025-30086

Beego is a popular Golang web framework (32k GitHub stars). Its filtering syntax is heavily inspired by Django:

```go
qs.Filter("title__contains", search)
```

### The Vulnerability in Harbor

Harbor (container registry, 27k GitHub stars) used Beego ORM with vulnerable code:

```go
qs.Filter(key, value)  // User controls both key and value!
```

This allowed filtering on ANY field including `password` and `salt`.

### Bypassing Patches

**First Patch:** Added `filter:"false"` annotation to sensitive fields
- **Bypass:** Use relational field parsing bug - `email__password` overwrites `email` with `password`

**Second Patch:** Limited `__` separator to one occurrence
- **Bypass:** Use fuzzy match format - `email__password=~value`

**Final Patch:** Allow-listed operators only

## Prisma ORM - Authentication Bypass

When input type validation is missing with Prisma, the JSON request body can inject Prisma operators:

```json
{"resetToken": {"not": ""}}
```

This injects the `not` condition, bypassing reset token validation!

### Attack Vectors
- JSON request bodies
- URL-encoded with extended parser: `resetToken[not]=E`
- JSON Cookies: `Cookie: resetToken=j:{"not": "E"}`

## Non-Susceptible ORMs

ORM leaks don't require susceptible ORMs. Any application with robust filtering can be vulnerable:

### Entity Framework + OData

```csharp
// Microsoft demo code - vulnerable!
modelBuilder.EntitySet<Article>("Articles");
```

The `User` model is automatically included due to associations, even if not explicitly added. Sensitive fields can be leaked via `$expand`.

### Directus CVE-2025-64748

The application included `token` and `tfa_secret` fields when searching, leaking authentication credentials.

## Database Collation Impact

Collation affects ORM leak exploitation:
- **Case sensitivity:** MySQL/MariaDB/SQLite default to case-insensitive
- **Character ordering:** MSSQL `SQL_Latin1_General_CP1_CI_AS` has non-byte-ordered sorting
- **Logical operators:** Can be used for binary search even without case-sensitive operators

## Key Takeaways

1. **ORM Leaks are common** - More frequent than SQL injection in bug bounties
2. **Any ORM could be leaked** - Even "secure" ones with proper filtering
3. **Logical operators are overlooked** - Greater than/less than can be powerful
4. **Allow-listing is key** - Don't rely on deny-lists

## References

- [Original Research - Alex Brown](https://www.elttam.com/blog/leaking-more-than-you-joined-for/)
- [plORMber Tool](https://github.com/elttam/plormber)
- [Semgrep Rules](https://github.com/elttam/semgrep-rules)
- [PortSwigger Top 10 2025](https://portswigger.net/research/top-10-web-hacking-techniques-of-2025)
