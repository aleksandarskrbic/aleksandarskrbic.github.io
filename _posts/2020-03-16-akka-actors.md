---
title: "Demo post"
date: 2020-03-16
tags: [scala, akka, jvm, big data]
excerpt: "Test one, two"
---

# H1 Title

Random text about programming.

{% include base_path %}
{% include group-by-array collection=site.posts field="tags" %}

{% for tag in group_names %}
  {% assign posts = group_items[forloop.index0] %}
  <h2 id="{{ tag | slugify }}" class="archive__subtitle">{{ tag }}</h2>
  {% for post in posts %}
    {% include archive-single.html %}
  {% endfor %}
{% endfor %}