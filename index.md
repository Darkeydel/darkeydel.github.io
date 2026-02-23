---
title: Home
layout: page
description: Security research blog
---

<header>
  <a href="/" class="logo">darkeydel</a>
  <nav>
    <a href="/">Home</a>
  </nav>
</header>

<main>
  <section class="about">
    <h1>Security Research</h1>
    <p>Research and writeups on web security, bug bounty and CTF challenges.</p>
  </section>

  <section class="posts">
    <h2 style="color: #fff; margin-bottom: 25px; font-size: 20px;">Blog posts</h2>
    <ul class="posts-list">
      {% assign sorted_posts = site.posts | sort: "date" | reverse %}
      {% for post in sorted_posts %}
      <li class="post-item">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <p class="post-meta">{{ post.date | date: "%B %d, %Y" }}</p>
        {% if post.description %}
        <p class="post-excerpt">{{ post.description }}</p>
        {% endif %}
      </li>
      {% endfor %}
    </ul>
  </section>
</main>
