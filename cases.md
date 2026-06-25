---
layout: default
title: Кейсы
permalink: /cases/
---

<h1>Кейсы</h1>

<p>Технические разборы из реальной практики. Без NDA и без раскрытия деталей конкретных коммерческих проектов.</p>

<ul class="post-list">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">
        {{ post.title }}
        <div class="post-meta">{{ post.date | date: "%d.%m.%Y" }}</div>
      </a>
    </li>
  {% endfor %}
</ul>
