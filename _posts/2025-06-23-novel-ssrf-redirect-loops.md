---
title: "Novel SSRF Technique: HTTP Redirect Loops"
date: 2025-06-23
categories:
  - Security Research
  - SSRF
  - HTTP
---

Blind Server-Side Request Forgery bugs are tricky to exploit. This technique makes blind SSRF visible by leveraging HTTP redirect loops and incremental status codes.

## The Problem

Blind SSRF vulnerabilities are difficult to exploit because:
- The application doesn't return the full HTTP response
- Parsing errors occur for unexpected response formats
- Can't see cloud metadata responses (169.254.169.254)

## The Discovery

While testing enterprise software using libcurl under the hood, researchers discovered unexpected behavior:
- Following redirects resulted in JSON parsing errors
- Following too many redirects (above 30) resulted in NetworkException
- But certain 3xx status codes triggered different error handling!

## The Technique

### Step 1: Test Redirect Following

First, determine if the application follows redirects and how many it can follow.

### Step 2: Test Different Status Codes

Test different 3xx status codes to find ones that behave differently.

### Step 3: Create Redirect Loop

Create a Flask server that:
1. Increments status code on each redirect (301, 302, 303...)
2. After reaching a threshold, redirects to final destination
3. Application returns full redirect chain including final response

```python
@app.route('/redir', methods=['GET', 'POST'])
def redir():
    redirect_count = int(request.args.get('count', 0))
    redirect_count += 1
    status_code = 301 + redirect_count
    
    if redirect_count >= 10:
        return redirect("http://example.com", code=302)
    
    return redirect(f"/redir?count={redirect_count}", code=status_code)
```

### The Magic

When redirect status codes exceed a certain threshold (like 305+), the application returns the **full HTTP response chain**, including the final 200 OK response!

```
HTTP/1.1 305 USE PROXY
Location: /redir?count=4

HTTP/1.1 306 SWITCH PROXY
...

HTTP/1.1 302 FOUND
Location: https://example.com

HTTP/1.1 200 OK
<full response body>
```

## Why It Works

The theory:
- Application follows a few redirects, fails on JSON parsing
- Application doesn't want to follow too many (max redirects)
- There's an error state between these limits not handled properly
- The error handler leaks the full redirect chain

## Impact

This technique allows:
- Leaking AWS metadata credentials
- Reading internal services
- Bypassing blind SSRF limitations

## References

- [Original Research - Shubham Shah](https://slcyber.io/research-center/novel-ssrf-technique-involving-http-redirect-loops/)
- [PortSwigger Top 10 2025](https://portswigger.net/research/top-10-web-hacking-techniques-of-2025)
