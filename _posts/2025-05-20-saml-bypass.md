---
title: "The Fragile Lock: SAML Authentication Bypass"
date: 2025-05-20
categories:
  - Security Research
  - SAML
  - Authentication
---

## Overview

Void Canonicalization + Parser namespace/attribute inconsistencies to make signature verification and assertion processing diverge for full SAML authentication bypass.

## Introduction

SAML (Security Assertion Markup Language) is widely used for SSO. This research reveals new bypass techniques.

## The Vulnerabilities

### 1. Void Canonicalization

Forcing canonicalization to error so signature-digest code treats the signed data as empty:

```
<ds:Signature>
  <ds:SignedInfo>
    <ds:CanonicalizationMethod Algorithm="http://www.w3.org/TR/xml-c14n11"/>
    <ds:Reference URI="#assertion">
      <ds:DigestValue></ds:DigestValue>  <!-- Empty! -->
    </ds:Reference>
  </ds:SignedInfo>
  <ds:SignatureValue></ds:SignatureValue>
</ds:Signature>
```

### 2. Namespace Inconsistencies

Parser handles namespaces differently:
- Signature verification sees: `namespace:element`
- Assertion processing sees: `element`

### 3. Attribute Ordering

Different attribute orders lead to different interpretations.

## Exploitation

1. **Create Malformed Assertion**: Craft SAML with above issues
2. **Bypass Signature Check**: Error handling makes signature valid
3. **Authentication Bypass**: Process assertion as legitimate user

## Impact

- Full account takeover
- Privilege escalation
- Access to sensitive resources

## Mitigation

1. **Strict Validation**: Validate all elements before processing
2. **Canonicalization**: Use standard canonicalization consistently
3. **Library Updates**: Keep SAML libraries updated

## References

- [Original Research](https://portswigger.net/research/the-fragile-lock)
