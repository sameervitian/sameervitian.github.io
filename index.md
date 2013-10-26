---
layout: page
title: CacheIt
tagline: stuffs that matter
---

<nav>
<ul class="posts">
        {% for post in site.posts limit:10 %}
        <li class="article-link">
        <a href="{{ post.url }}" class="h3">{{ post.title }}</a>
        <small>
                <date>{{ post.date | date_to_string }}</date>
        </small>
        </li>
        {% endfor %}
</ul>
</nav>
