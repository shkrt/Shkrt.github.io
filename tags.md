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

### Workflow

[Dry-matcher usage examples]({{ site.baseurl }}{% post_url 2017-09-25-dry-matcher-examples %})

### Dry-rb

[Dependency inversion with dry-cintainer]({{ site.baseurl }}{% post_url 2017-10-09-dependency-inversion-with-dry-container %})

[How to describe issues in a clear way]({{ site.baseurl }}{% post_url 2017-10-02-howto-describe-issues %})

### Ruby

[Different approaches to file attaching]({{ site.baseurl }}{% post_url 2017-10-16-different-approaches-to-file-attaching %})

[Simplest implementation of interactor]({{ site.baseurl }}{% post_url 2017-10-23-simplest-implementation-of-interactor %})

[Building API with Roda and Sequel]({{ site.baseurl }}{% post_url 2017-10-30-building-api-with-roda-and-sequel %})

[Comparing Sequel and ActiveRecord]({{ site.baseurl }}{% post_url 2017-11-06-comparing-sequel-and-activerecord %})

