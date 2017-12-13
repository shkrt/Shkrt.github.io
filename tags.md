---
layout: default
title: Tags
permalink: /tags/
---

{% for item in site.tags %}
###	{{ item[0] }}

  {% for post in site.posts %}
    {% if post.tags contains item[0] %}
[{{ post.title }}]({{ site.baseurl }}{{ post.url }})
    {% endif %}
  {% endfor %}
{% endfor %}

