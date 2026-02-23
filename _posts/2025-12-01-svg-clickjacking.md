---
title: "SVG Filters: Clickjacking 2.0"
date: 2025-12-01
categories:
  - Security Research
  - Clickjacking
  - SVG
  - CSS
---

## Overview

Abusing SVG filter pipelines on cross-origin iframes to read selected pixels and implement logic-gated, multi-step interactive clickjacking with exfiltration via user-scanned QR codes generated entirely inside the filter.

## Introduction

Traditional clickjacking has evolved. New techniques use SVG filters to:
- Read pixel data from iframes
- Implement complex logic gates
- Exfiltrate data via QR codes

## The Attack Vector

### SVG Filter Basics

SVG filters can process pixel data:

```xml
<svg>
  <filter id="pixelRead">
    <feImage href="targetiframe" result="img"/>
    <feColorMatrix type="matrix" in="img" values="..."/>
  </filter>
</svg>
```

### Reading Cross-Origin Data

By applying filters to cross-origin iframes, attackers can:
1. Render iframe content through SVG filter
2. Extract pixel information
3. Reconstruct data visually

## Advanced Techniques

### 1. QR Code Exfiltration

Generate QR codes inside SVG filters to encode stolen data:

```xml
<filter id="exfiltrate">
  <feImage href="sensitive-data"/>
  <feQRCode encode="auto" version="L"/>
</filter>
```

### 2. Logic Gating

Multi-step attacks based on pixel values:
- If pixel(X,Y) = red → Click here
- If pixel(X,Y) = blue → Click there

### 3. Progressive Exfiltration

Multiple clicks to extract entire datasets:
- Click 1: Character 1-4
- Click 2: Character 5-8
- ...

## Countermeasures

1. **X-Frame-Options**: DENY or SAMEORIGIN
2. **CSP**: frame-ancestors directive
3. **Content-Type**: Prevent SVG content loading as HTML

## References

- [Original Research](https://lyra.horse/blog/2025/12/svg-clickjacking/)
