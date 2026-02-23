---
title: Writeups
layout: page
permalink: /writeups/
---

<div class="archive">
  <h1 class="page__title">Writeups</h1>
  <p>Bug bounty findings and vulnerability disclosures.</p>
  
  {% assign writeups = site.posts | where: "categories", "writeup" | sort: "date" | reverse %}
  {% for post in writeups %}
  <div class="list__item">
    <article class="archive__item" itemscope itemtype="http://schema.org/CreativeWork">
      <h2 class="archive__item-title" itemprop="headline">
        <a href="{{ post.url | relative_url }}" rel="permalink">{{ post.title }}</a>
      </h2>
      <p class="archive__item-excerpt" itemprop="description">{{ post.description }}</p>
    </article>
  </div>
  {% endfor %}
</div>
