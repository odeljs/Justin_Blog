---
layout: page
title: Post-Mortems
permalink: /post_moretums/
---

Here is an organized archive of all incident post-mortems:

<ul class="posts-list">
  {% assign sorted_docs = site.post_moretum | sort: 'date' | reverse %}
  {% for doc in sorted_docs %}
    <li class="post-preview">
      <a href="{{ doc.url | relative_url }}">
        <h2 class="post-title">{{ doc.title }}</h2>
      </a>
      <p class="post-meta">Posted on {{ doc.date | date: "%B %d, %Y" }}</p>
      {% if doc.description %}
        <div class="post-entry">{{ doc.description }}</div>
      {% endif %}
    </li>
  {% endfor %}
</ul>
