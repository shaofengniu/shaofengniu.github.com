---
layout: page
title: Home
---
{% include JB/setup %}
<img class='inset left' title='Shaofeng Niu' src='/images/me.png' alt='Photo of Me' />
<div class="section">
<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
</div>
