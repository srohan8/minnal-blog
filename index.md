---
layout: default
title: Blog
---
<main class="blog-index">
  <div class="container">
    <div class="blog-header">
      <h1>Minnal Blog</h1>
      <p>Insights on AI-native marketing, GEO, and building brand visibility in the age of AI search.</p>
    </div>
    <div class="post-list">
      {% for post in site.posts %}
      <a href="{{ post.url }}" class="post-card">
        <div class="post-card-meta">{{ post.date | date: "%B %-d, %Y" }}</div>
        <h2>{{ post.title }}</h2>
        {% if post.description %}<p>{{ post.description }}</p>{% endif %}
        <span class="read-more">Read more →</span>
      </a>
      {% endfor %}
    </div>
  </div>
</main>
