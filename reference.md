---
title: Reference
description: Look up a fact.
permalink: /reference/
---

# Reference

Dry, precise, and structured to be scanned rather than read start to finish — the real facts
(routes, fields, behavior), not a narrative.

<div class="index-list">
{% for p in site.reference %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
