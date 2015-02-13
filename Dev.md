---
layout: page
title: Dev
permalink: dev/
---
{% for post in site.posts %}
  {% if post.category == "Dev" %}
* {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
  {% endif %}
{% endfor %}