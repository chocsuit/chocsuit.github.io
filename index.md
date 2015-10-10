---
layout: page
title: Tech Summary!
tagline: 一个代码痴汉的自白
---
{% include JB/setup %}
    
# Recent Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


