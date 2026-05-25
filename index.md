---
layout: default
title: Blog
permalink: /
---

[RSS](/feed.xml)

{% for post in site.posts %}
# [{{ post.title }}]({{ post.url }})

{{ post.content }}

{% endfor %}
