---
layout: default
title: Blog
permalink: /
---

{% for post in site.posts %}
# [{{ post.title }}]({{ post.url }})

{{ post.content }}

{% endfor %}
