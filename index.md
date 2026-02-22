---
title: Security Research Blog
description: Critical findings and security research
---

# Security Research Blog

Welcome to my security research blog. Here I document critical vulnerabilities and security findings.

## Latest Posts

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}

## About

This blog publishes security research, vulnerability disclosures, and CTF writeups.
