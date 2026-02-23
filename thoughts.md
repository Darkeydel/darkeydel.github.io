---
title: Thoughts
layout: page
permalink: /thoughts/
---

<div class="archive">
  <p>Security insights, opinions, and musings.</p>
  
  {% assign thoughts = site.posts | where: "categories", "thoughts" | sort: "date" | reverse %}
  {% for post in thoughts %}
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
