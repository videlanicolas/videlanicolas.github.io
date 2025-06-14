---
layout: default
title: Blog
permalink: /blog/
---

<h1>{{ page.title }}</h1>

<ul>
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <p class="post-meta">__{{ post.date | date: site.minima.date_format | default: "%b %-d, %Y" }}__</p>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
