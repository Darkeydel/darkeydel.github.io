---
title: "Lost in Translation: Exploiting Unicode Normalization"
date: 2025-06-15
categories:
  - Security Research
  - Unicode
  - Normalization
  - WAF
---

Unicode normalization attacks have lurked on the edge of testing methodologies for years. This research tackles this vast topic with practical exploitation techniques and tool updates.

## Introduction

Unicode normalization is the process of converting Unicode strings to a canonical form. Different systems normalize Unicode differently, creating security vulnerabilities.

## The Problem

When user input containing Unicode is processed by different components:
- Web applications
- Databases
- WAFs
- Operating systems

Each may normalize differently, leading to bypasses.

## Normalization Forms

| Form | Description |
|------|-------------|
| NFD | Canonical Decomposition |
| NFC | Canonical Decomposition + Composition |
| NFKD | Compatibility Decomposition |
| NFKC | Compatibility Decomposition + Composition |

## Attack Techniques

### 1. IDN Homograph Attacks

```python
# Cyrillic 'a' vs Latin 'a'
"аpple.com"  # Cyrillic (U+0430)
"apple.com"  # Latin (U+0061)
```

Visual spoofing for phishing or bypass.

### 2. SQL Injection Bypass

```sql
' OR 1=1--
' OR 1=1\u2014'
```

Different Unicode representations of common SQLi payloads.

### 3. WAF Bypass

```html
<img src=x onerror=alert(1)>
<img src=x onerror=\u0061lert(1)>
```

### 4. Path Traversal

```python
../../../etc/passwd
..\u002F..\u002F..\u002Fetc\u002Fpasswd
```

## Database-Specific Issues

- **MySQL**: Various collations handle Unicode differently
- **PostgreSQL**: UTF-8 normalization
- **SQL Server**: Windows collation quirks

## Tool Updates

This research includes updates to:
- ActiveScan++
- Custom fuzzing tools

## References

- [Original Research - Ryan Barnett & Isabella](https://www.youtube.com/watch?v=ETB2w-f3pM4)
- [PortSwigger Top 10 2025](https://portswigger.net/research/top-10-web-hacking-techniques-of-2025)
