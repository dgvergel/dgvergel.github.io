---
layout: page
title: All posts
permalink: /all-posts/
---

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) – {{ post.date | date: "%Y/%m/%d }}
{% endfor %}
