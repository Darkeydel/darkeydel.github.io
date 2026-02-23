---
title: "Successful Errors: New Code Injection and SSTI Techniques"
date: 2025-01-03
categories:
  - Security Research
  - SSTI
  - Code Injection
  - Web Security
---

This research introduces new error-based techniques for exploiting blind server-side template injection, including novel polyglot-based detection techniques.

## Introduction

Server-Side Template Injection (SSTI) is a critical vulnerability that can lead to remote code execution. However, detecting and exploiting blind SSTI remains challenging. This research introduces new techniques that adapt old-school SQL injection methods to SSTI.

## The Problem

Clear and obvious names create false familiarity. "Error-based" techniques for SSTI were never thoroughly researched - payloads were limited to specific examples.

This research focuses on two techniques:
1. Error-based Code Injection
2. Error-based SSTI

## Error-Based Techniques

### Traditional SQL Injection vs SSTI

| SQL Injection | SSTI |
|--------------|------|
| Error-based Oracle | Error messages reveal info |
| UNION-based | Template debugging |
| Blind boolean | Time-based |

### New SSTI Approach

Error messages in templates can reveal:
- Template engine version
- Internal paths
- Available objects
- Even execute code through error handling

## Novel Polyglot Detection

### SSTI Polyglots

Polyglots that work across multiple template engines:

```python
{{7*7}}
${7*7}
<%= 7*7 %>
{{config}}
```

### Detection Technique

1. Inject error-causing payloads
2. Analyze error messages
3. Identify template engine
4. Choose appropriate exploitation

## Tools

### SSTImap

Automatic SSTI detection tool with interactive interface:
```bash
python3 ./sstimap.py -u 'https://example.com/?name=John' -s
```

### TInjA

Efficient SSTI + CSTI scanner with novel polyglots:
```bash
tinja url -u "http://example.com/?name=Kirlia"
```

## Exploitation Techniques

### 1. Error-Based Enumeration

Force error messages to reveal:
- Available classes
- File paths
- Configuration
- Version information

### 2. Blind RCE via Error Handling

Some applications display detailed errors that can be leveraged for code execution even without direct output.

### 3. Template-Specific Payloads

Different engines require different payloads:
- Jinja2: `{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}`
- Twig: `{{_self.env.display('{{7*7}}')}}`
- FreeMarker: `${"freemarker.template.utility.Execute"?new()("id")}`

## Impact

By adapting SQL injection techniques to SSTI, this research opens new exploitation paths for what was previously considered "blind" or unexploitable.

## References

- [Original Research - Vladislav Korchagin](https://github.com/vladko312/Research_Successful_Errors)
- [SSTImap](https://github.com/vladko312/SSTImap)
- [TInjA](https://github.com/Hackmanit/TInjA)
- [PortSwigger Top 10 2025](https://portswigger.net/research/top-10-web-hacking-techniques-of-2025)
