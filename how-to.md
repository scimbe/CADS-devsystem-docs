---
title: How-to guides
description: Accomplish one specific task.
permalink: /how-to/
---

# How-to guides

Goal-oriented, one task at a time — you already know roughly what you're doing and want the exact
steps, not an explanation of why.

<div class="index-list">
{% for p in site.how-to %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
