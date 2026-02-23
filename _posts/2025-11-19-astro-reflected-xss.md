---
title: "Unlocking Reflected XSS in the Astro framework"
date: 2025-11-19
description: "CVE-2025-64764 - Reflected XSS vulnerability in Astro framework server islands"
categories:
  - research
---

![](/images/astro-rxss-0.png)

## Introduction

Back to the Astro framework. Following our initial, rather fruitful research on the framework, which we encourage you to read if you haven't already, this paper details a small discovery concerning a Reflected XSS vulnerability that led to CVE-2025-64764.

As I mentioned in a previous article, I've decided to be more selective about which vulnerabilities deserve a dedicated paper. This helps avoid redundancy, prevent a drop in quality, and, of course, maintain exclusivity on a few exploits that support our side activities as a bounty hunters.

We hesitated to write a full article for this one, but an RXSS affecting a framework downloaded hundreds of thousands of times per week deserved it, even if it's shorter than usual.

## Index

- [Partial SSR with Server Islands](#partial-ssr-with-server-islands)
- [Exploit lock in a conditional block](#exploit-lock-in-a-conditional-block)
- [Reflected XSS](#reflected-xss)
  - [Anti-WAF Helper](#anti-waf-helper)
- [Security Advisory - CVE-2025-64764](#security-advisory---cve-2025-64764)
- [Conclusion](#conclusion)

## Partial SSR with Server Islands

Astro includes a feature called Server islands, which allows specific components to be isolated from the rest of the page and rendered dynamically on the server. This approach keeps the page itself static and fully cacheable, while each server island can generate dynamic content on the fly, enhancing overall performance.

Server islands run in their own isolated context outside of the page request and use the following pattern path to hydrate the page: `/_server-islands/[name]`. These paths are called via `GET` or `POST`, depending on the size of the data transmitted, and use three parameters (`GET` verb):

![](/images/astro-rxss-1.png)

`e`: Contains the name of the component export, which Astro uses to look up the correct export in the island module

`p`: Contains the props of the island component, encrypted server-side (AES‑256‑GCM) before being sent to the browser. The encryption key is generated or imported once, stored in `manifest.key`, and reused when the server islands component request arrives so the server can decrypt `p` and rebuild the props. This ensures that props are not sent in plain text between the browser and the server, and prevents an attacker from spoofing their content.

`s`: Contains the slot values in JSON format, where each key is the name of the slot. A slot is a location in a component where you can inject content, these act as placeholders for external HTML and, by default, allow code injection **if the component template supports it**. Nothing exceptional in principle, just an optional feature.

## Exploit lock in a conditional block

This is where it becomes interesting/problematic: it is, in fact, possible, **regardless of the component template being used** and whether it initially provides slots or not, to inject a slot containing an XSS payload. This **enables reflected XSS on any application**, as long as a server island is used at least once.

The key lies in the following conditional block:

![](/images/astro-rxss-3.png)

This code handles the case where `Component` is not a function (Astro/React/.. component). In this case, Astro generates a complete template and returns the built HTML, independently of any template originally provided by the developer.

- the `Component` value, being a string here, is sanitized and then injected as the tag name
- `childSlots`, the value provided to the `s` parameter, is injected as a child of the tag without any sanitization

Access to this block would unlock the exploit, and for that, the value of the `Component` must be a `string`, which is not originally the case.

`Component` is the export of the island module selected via `data.componentExport`, itself defined by the parameter `e`. The value of `e` is used as a key to access the corresponding property in `componentModule`.

By default, its value is `default`, whose type is a `function`.

## Reflected XSS

As explained earlier, `Component` is defined by the value passed to `e`, which acts as the key for the `componentModule` object. From there, a simple console.log is enough to reveal all existing exports:

![](/images/astro-rxss-5.png)

A very pleasant sight appears in the output: in addition to `default`, two other exports are present by default: `url`, whose value is `undefined`, and… `file`, **a string** whose value is the absolute path of the island file.

Since the type is a `string`, we can fulfill the condition of the block above, unlocking the exploit by setting `file` as the value of the parameter `e` and injecting an XSS payload as a slot (`s` parameter):

![](/images/astro-rxss-6.png)

There's no need to set anything in the value of `p`, since the returned template is the one generated by Astro, **completely ignoring the template intended by the developer**. For the same reason, the JSON keys (`s` value) are irrelevant here, since we are not pointing to any existing slots.

### Anti-WAF Helper

Another interesting fact is that all values (from the key/value pairs) provided in the JSON passed to the `s` parameter are concatenated together before being injected as a child:

```javascript
const childSlots = Object.values(children).join('');
```

This allows us to divide our payload into as many pieces as we want, offering an almost guaranteed bypass of all WAFs:

```
?e=file&p=&s={"zhero;":"<img+src=x+one","inzo":"rror=a","dz":"lert(0)>"}
```

![](/images/astro-rxss-8.png)

## Security Advisory - CVE-2025-64764

Affected versions: =< 5.15.6

Patched versions: 5.15.8

CVSS score: `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:L/I:H/A:N` (High - 7.1)

## Conclusion

As we have seen, arbitrary JavaScript code execution is possible, requiring a user click, and the sole condition for its exploitability is that the targeted application uses at least one server island. The consequences can obviously be severe, especially considering the popularity of the framework (+800,000 downloads per week). The impact aligns with the typical range of what is achievable through classic XSS attacks: from the exfiltration of sensitive data to potential full account takeover.

Fortunately, the team was once again very responsive and the fixes were implemented and released within 48 hours of the vulnerability being reported.

**Timeline:**

- 2025-11-13: Report sent via the GitHub template
- 2025-11-14: Report acknowledged and accepted
- 2025-11-14: PR review requested by the Astro team
- 2025-11-15: PR review completed
- 2025-11-15: Implementation of the latest fixes and release of the patched version - `astro@5.15.8`
- 2025-11-19: Publication of the security advisory

Research conducted by zhero; & inzo_
