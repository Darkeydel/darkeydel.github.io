---
title: How to Exploit Parser Differentials
date: 2024-01-15
categories:
  - Security Research
  - Parser Differential
---

Parser differentials are a fascinating class of vulnerabilities that arise when different components interpret the same input in different ways. This post explores real-world examples of how attackers exploit these differences to achieve remote code execution or bypass security controls.

![Parser Differential Overview](/images/parser-differential-overview.png)

---

## What Are Parser Differentials?

> Different interpretation of messages or data streams by components breaks any assumptions that components adhere to a shared specification and so introduces inconsistent state and unanticipated computation.

![LangSec Approach](/images/langsec-approach.png)

The term originates from the **Language-theoretic Security (LangSec)** approach, which argues that input handling is the root cause of many security vulnerabilities. When multiple parsers process the same data, they often reach different conclusions—and attackers can exploit these differences.

---

## GitLab File Upload Vulnerability (CVE-2020-)

### The Attack Surface

Modern web applications often use layered architectures. In GitLab's case:

| Component | Role |
|-----------|------|
| **GitLab Workhorse** | Reverse proxy that handles file uploads |
| **GitLab Rails** | Ruby on Rails backend that processes requests |

![GitLab Architecture](/images/gitlab-architecture.png)

### How the Vulnerability Worked

GitLab Workhorse receives file uploads and rewrites requests to pass file path information to GitLab Rails:

```http
PUT /api/v4/packages/conan/v1/files/Hello/0.1/root+xxxxx/beta/0/export/conanfile.py
```

Workhorse modifies the request to include metadata:

```json
{
  "key": "file.path",
  "value": "/var/opt/gitlab/gitlab-rails/shared/packages/tmp/uploads/582573467"
}
```

### The Parser Differential

| Component | What It Sees |
|-----------|--------------|
| **GitLab Workhorse** | `POST` or `PUT` request |
| **GitLab Rails** | Any HTTP method (via `Rack::MethodOverride`) |

![HTTP Method Override Bypass](/images/method-override-bypass.png)

By using the `X-HTTP-Method-Override` header or `_method` parameter, attackers could bypass Workhorse routing entirely:

```http
POST /api/v4/packages/conan/v1/files/Hello/0.1/lol+wat/beta/0/export/conanmanifest.txt?file.size=4&file.path=/tmp/test1234
X-HTTP-Method-Override: PUT
```

This allowed **arbitrary file read** from the server's filesystem.

### The Fix

GitLab addressed this by implementing **request signing**:

- Workhorse now signs all requests passing through it
- Rails verifies the signature before processing
- Unauthorized requests are rejected

> See the fix: [gitlab_workhorse commit 3a34323b](https://gitlab.com/gitlab-org/gitlab-workhorse/-/commit/3a34323b104be89e92db49828268f0bfd831e75a)

---

## CouchDB Remote Code Execution

### The Root Cause

CouchDB uses **Erlang** (jiffy) for document storage but allows **JavaScript** for validation functions. These two parsers handle duplicate keys differently:

![CouchDB Architecture](/images/couchdb-architecture.png)

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

### Exploiting the Differential

By sending a user document with duplicate keys:

```json
{
  "type": "user",
  "name": "oops",
  "roles": ["admin"],
  "roles": [],
  "password": "password"
}
```

| Parser | Interpretation |
|--------|----------------|
| **Erlang** | `roles: ["admin"]` |
| **JavaScript** | `roles: []` |

The validation passes (JavaScript sees empty roles), but Erlang stores the admin role—granting the attacker administrator access.

![CouchDB Exploit](/images/couchdb-exploit.png)

---

## HTTP Desync Attacks

Another form of parser differential occurs when front-end and back-end proxies parse HTTP requests differently. The [HTTP desync attack technique](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn) exploits this to:

- Bypass authentication
- Harvest user credentials
- Perform cache poisoning
- Hijack user sessions

![HTTP Desync Attack](/images/http-desync-attack.png)

### How It Works

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Attacker │───▶│   Front-end │───▶│   Back-end  │
│   (smuggler)    │   (Proxy)   │    │   (Server)  │
└─────────────┘    └─────────────┘    └─────────────┘
```

The front-end and back-end disagree on where the request ends, allowing attackers to inject malicious requests.

---

## Template Injection: Jinja2 vs Python

Server-side template injection (SSTI) often involves parser differentials between the template engine and the underlying programming language.

![Template Injection](/images/template-injection.png)

### Flask/Jinja2 Example

In Flask applications, attackers can sometimes escape the template sandbox:

```python
{{ request.__class__.__mro__[2].__subclasses__() }}
```

This traverses Python's object hierarchy to find classes that can execute system commands.

### The Differential

| Layer | What It Does |
|-------|--------------|
| **Jinja2** | Renders templates, has sandbox restrictions |
| **Python** | Executes arbitrary code if accessible |

---

## XML External Entity (XXE) Injection

Different XML parsers handle external entities differently:

![XXE Attack](/images/xxe-attack.png)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<data>&xxe;</data>
```

| Parser | Behavior |
|--------|----------|
| **libxml2** | May disable external entities by default |
| **Java XMLStreamReader** | Often allows by default |
| **PHP SimpleXML** | Varies by configuration |

---

## JWT Parser Differential

JSON Web Tokens can be parsed differently across libraries:

![JWT Structure](/images/jwt-structure.png)

### The "none" Algorithm Attack

```javascript
// Token header
{"alg": "none", "typ": "JWT"}

// Attacker removes signature, server may accept "none"
```

| Library | Vulnerable? |
|---------|-------------|
| pyjwt | Yes (older versions) |
|-json nodewebtoken | Yes |
| java-jwt | No (rejects "none") |

### Key Confusion Attack

```javascript
// RS256 token (asymmetric)
// Attacker changes to HS256 and signs with public key
{"alg": "HS256", "typ": "JWT"}
```

Some libraries don't properly validate the algorithm, allowing attackers to switch from asymmetric to symmetric algorithms.

---

## URL Parser Differential

Different URL parsing libraries can interpret URLs differently:

![URL Parser](/images/url-parser.png)

### Example: Scheme Confusion

```python
# What does this URL actually do?
"http://evil.com;@google.com"
```

| Parser | Result |
|--------|--------|
| **Python urlparse** | Host: `google.com` |
| **PHP parse_url** | Host: `evil.com` |

### AWS S3 Bucket Takeover

This differential has been used to takeover AWS S3 buckets:

```
http://target.s3.amazonaws.com/vulnerable-param
```

---

## Unicode Normalization Vulnerabilities

Different systems normalize Unicode differently, leading to security issues:

![Unicode Normalization](/images/unicode-normalization.png)

### IDN Homograph Attacks

```python
# Cyrillic 'a' vs Latin 'a'
"аpple.com"  # Cyrillic
"apple.com"  # Latin
```

| Component | Normalization |
|-----------|---------------|
| **Punycode** | Converts to ASCII |
| **Browsers** | Display Unicode |
| **IDNA** | Varies by version |

---

## PHP Type Juggling

PHP's type coercion creates parser differentials:

![PHP Type Juggling](/images/php-type-juggling.png)

### Loose Comparison Attack

```php
"0e12345" == "0e54321"  // True! Both treated as scientific notation
"1" == "01"              // True!
"abc" == 0               // True!
```

| Comparison | Result |
|------------|--------|
| `"0e12345" == "0e99999"` | `true` (both = 0) |
| `"1" == "1e0"` | `true` |
| `"1" == "01"` | `true` |

### Password Hash Comparison

```php
if (md5($password) == $stored_hash)  // Vulnerable!
```

---

## LDAP Injection

Different LDAP implementations parse queries differently:

![LDAP Injection](/images/ldap-injection.png)

```python
# Injection payload
*)(uid=*))(|(uid=*

# Results in:
(&(uid=*))(|(uid=*))
```

| Parser | Behavior |
|--------|----------|
| **OpenLDAP** | Strict filtering |
| **Active Directory** | May allow bypassing |

---

## ReDoS (Regular Expression Denial of Service)

The same regex can be parsed differently by different engines:

![ReDoS Attack](/images/redos-attack.png)

```python
# Vulnerable regex
^(a+)+$

# Input: aaaaaaaaaaaX (catastrophic backtracking)
```

| Engine | Behavior |
|--------|----------|
| **Python re** | Backtracking, vulnerable |
| **Rust regex** | Linear time, safe |
| **RE2** | Deterministic, safe |

---

## SQL Injection: Different DBMS

Different database systems parse SQL differently:

![SQL Injection](/images/sql-injection.png)

```sql
-- MySQL comment
admin' --

-- Different comment syntax
admin' #
```

| Database | Comment Syntax |
|----------|----------------|
| MySQL | `#`, `-- ` |
| PostgreSQL | `--` |
| SQL Server | `--`, `/* */` |

---

## Path Traversal Differential

Filesystem parsers handle paths differently:

![Path Traversal](/images/path-traversal.png)

```python
# Both may work
../../../etc/passwd
..%2F..%2F..%2Fetc%2Fpasswd
..\..\..\windows\system32\config\sam
```

| Parser | Interpretation |
|--------|----------------|
| **Windows** | `\`, `/` both work |
| **URL decode** | `%2F` = `/` |
| **Double decode** | `%252F` = `%2F` = `/` |

---

## Why This Matters

The rise of **microservices** creates complex environments where a single HTTP request may be interpreted by multiple services differently—often with serious security implications.

![Microservices Complexity](/images/microservices-complexity.png)

---

## How to Defend Against Parser Differentials

### 1. Input Validation at All Layers

```
┌─────────────────────────────────────┐
│         Defense in Depth            │
├─────────────────────────────────────┤
│  1. Front-end validation            │
│  2. API gateway validation          │
│  3. Backend validation              │
│  4. Database schema constraints     │
└─────────────────────────────────────┘
```

### 2. Use Consistent Parsers

- Standardize on one JSON library
- Use consistent URL parsing
- Validate at every boundary

### 3. Request Signing

Like GitLab's fix, sign requests between components:

```python
signature = hmac_sha256(secret, request_data)
```

### 4. Security Testing

- Fuzz parsers with varied inputs
- Test edge cases
- Use automated scanners

---

## References

- [How to exploit parser differentials - GitLab Blog](https://about.gitlab.com/blog/how-to-exploit-parser-differentials/)
- [CouchDB RCE - Max Justicz](https://justi.cz/security/2017/11/14/couchdb-rce-npm.html)
- [HTTP Desync Attacks - PortSwigger Research](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)
- [HTTP Desync Attacks Paper - PortSwigger](https://portswigger.net/kb/papers/z7ow0oy8/http-desync-attacks.pdf)
- [LangSec - Language-theoretic Security](http://langsec.org)
- [HTTP Garden - Differential Fuzzing](https://arxiv.org/abs/2405.17737)
- [JWT Algorithm Confusion - PortSwigger](https://portswigger.net/web-security/jwt/algorithm-confusion)
- [JWT Algorithm Confusion - Pentest UK](https://pentest.co.uk/insights/json-web-token-algorithm-confusion-attack/)
- [JWT Structure - Auth0](https://auth0.com/docs/tokens/json-web-tokens/json-web-token-structure)
- [XXE Vulnerability - Bright Security](https://www.brightsec.com/blog/xxe-vulnerability/)
- [XXE Prevention - OWASP](https://owasp.org/www-community/v/XML_External_Entity_Prevention_Cheat_Sheet)
- [SSTI Jinja2 - PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md)
- [SQL Injection Infographic - Veracode](https://www.veracode.com/blog/sql-injection-attacks-and-how-prevent-them-infographic/)
- [Understanding ReDoS - OWASP](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS)
- [Parser Differential Survey - IEEE](https://ieeexplore.ieee.org/document/10188627/)
- [Iterasec - Parser Differential Explained](https://iterasec.com/blog/understanding-parser-differential-vulnerabilities/)
