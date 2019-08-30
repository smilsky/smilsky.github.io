---
layout: archive
permalink: /projects/
title: "My Projects"
author_profile: true
header:
  image: "/images/about.jpg"
---
{% include base_path %}
  {% for tag in group_names %}
    {% assign posts = group_items[forloop.index0] %}
    <h2 id="{{ tag | slugify }}" class="archive_subtitle">{{ tag }}</h2>
    {% for post in posts %}
      {% includes archive-single.html %}
    {% endfor %}
  {% endfor %}
