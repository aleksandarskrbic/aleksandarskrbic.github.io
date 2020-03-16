---
title: "About"
layout: single
permalink: /about/
header:
    image: "/images/header.png"
---

Software Engineer with primarily focuses on distributed and data-intensive systems who started career as a Data Scientist,
but realized that a combination of a backend and data engineering suits me better.
By day I'm working in a data engineering team utilizing technologies such as Java, Spring Boot, Apache Kafka, Elasticsearch, Apache Ignite, and Apache Cassandra.
By night I'm either hakking with Akka or exploring functional programming techniques in Scala with ZIO, Monix, or Cats.

{% include base_path %}
{% include group-by-array collection=site.posts field="tags" %}

{% for tag in group_names %}
  {% assign posts = group_items[forloop.index0] %}
  <h2 id="{{ tag | slugify }}" class="archive__subtitle">{{ tag }}</h2>
  {% for post in posts %}
    {% include archive-single.html %}
  {% endfor %}
{% endfor %}