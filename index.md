---
layout: default
title: "Sam's Site"
---

{% for post in site.posts %}
            
{{ post.date }}: [{{ post.title }}]({{ post.url }})

{% endfor %}
