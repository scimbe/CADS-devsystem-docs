---
title: Explanation
description: Understand why something works the way it does.
permalink: /explanation/
---

# Explanation

Understanding-oriented — the design decisions, trade-offs, and real gaps behind how this pipeline
actually works, including where it's still honestly incomplete.

<div class="index-list">
{% for p in site.explanation %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
