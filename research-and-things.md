---
title: Research & Things
layout: page
permalink: /research-and-things/
---

<div class="archive">
  <h1 class="page__title">Research & Things</h1>
  
  {% assign sorted_posts = site.posts | sort: "date" | reverse %}
  {% for post in sorted_posts %}
  <div class="list__item">
    <article class="archive__item" itemscope itemtype="http://schema.org/CreativeWork">
      <h2 class="archive__item-title" itemprop="headline">
        <a href="{{ post.url | relative_url }}" rel="permalink">{{ post.title }}</a>
      </h2>
      <p>Published in <i>{{ site.title }}</i>, {{ post.date | date: "%Y" }}</p>
      {% if post.description %}
      <p class="archive__item-excerpt" itemprop="description">{{ post.description }}</p>
      {% endif %}
    </article>
  </div>
  {% endfor %}
</div>
