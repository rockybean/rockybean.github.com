---
layout: page
title: 文章列表
tagline: 
---
{% include JB/setup %}

<ul class="nav nav-list">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
