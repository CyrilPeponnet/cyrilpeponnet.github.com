---
layout: page
title: Hard
---
{% for post in site.posts %}
  {% if post.category == "Hard" %}
* {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
  {% endif %}
{% endfor %}