---
layout: page
title: Misc
permalink: Misc/
---
{% for post in site.posts %}
  {% if post.category == "Misc" %}
* {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
  {% endif %}
{% endfor %}