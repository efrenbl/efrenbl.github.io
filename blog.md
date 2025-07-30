---
layout: page
title: Blog
permalink: /blog/
---

# Blog Posts

Welcome to my blog! Here you'll find my thoughts on technology, programming, and various topics that interest me.

<div class="posts">
  {% for post in site.posts %}
    <article class="post">
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      <div class="entry">
        {{ post.excerpt }}
      </div>
      <p class="post-meta">
        Posted on {{ post.date | date: "%B %d, %Y" }}
        {% if post.tags.size > 0 %}
          | Tags: 
          {% for tag in post.tags %}
            <span class="tag">{{ tag }}</span>{% unless forloop.last %}, {% endunless %}
          {% endfor %}
        {% endif %}
      </p>
      <a href="{{ post.url }}" class="read-more">Read More</a>
    </article>
    <hr>
  {% endfor %}
</div>

{% if site.posts.size == 0 %}
<p>No posts yet. Check back soon for updates!</p>
{% endif %}
