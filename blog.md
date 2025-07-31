---
layout: page
title: Blog
permalink: /blog/
---

# Technical Blog

Welcome to my technical blog! I share insights on **machine learning engineering**, **cloud architecture**, **fintech innovation**, and **engineering leadership**. Whether you're building ML platforms, scaling infrastructure, or leading engineering teams, you'll find practical insights and lessons learned from real-world implementations.

---

## 🎯 What I Write About

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin: 20px 0;">

<div style="padding: 15px; border-left: 4px solid #2196F3; background-color: #f8f9fa;">
<h3>🤖 MLOps & Platform Engineering</h3>
<p>Best practices for building scalable ML platforms, model lifecycle management, and production ML systems.</p>
</div>

<div style="padding: 15px; border-left: 4px solid #4CAF50; background-color: #f8f9fa;">
<h3>☁️ Cloud Architecture</h3>
<p>AWS/Azure infrastructure patterns, security protocols, DevOps practices, and scaling strategies.</p>
</div>

<div style="padding: 15px; border-left: 4px solid #FF9800; background-color: #f8f9fa;">
<h3>👥 Engineering Leadership</h3>
<p>Team management insights, mentoring strategies, and career development advice for engineers.</p>
</div>

<div style="padding: 15px; border-left: 4px solid #9C27B0; background-color: #f8f9fa;">
<h3>💳 Fintech Innovation</h3>
<p>AI in financial services, scaling challenges, and building ML systems that serve millions of users.</p>
</div>

</div>

---

## 📚 Latest Posts

<div class="posts">
  {% for post in site.posts %}
    <article class="post" style="margin-bottom: 30px; padding: 20px; border: 1px solid #e1e1e1; border-radius: 8px;">
      <h2 style="margin-top: 0;"><a href="{{ post.url }}" style="text-decoration: none; color: #333;">{{ post.title }}</a></h2>
      <p class="post-meta" style="color: #666; font-size: 0.9em; margin-bottom: 15px;">
        📅 {{ post.date | date: "%B %d, %Y" }}
        {% if post.tags.size > 0 %}
          <br>🏷️ Tags: 
          {% for tag in post.tags %}
            <span style="background-color: #e3f2fd; padding: 2px 8px; border-radius: 12px; font-size: 0.8em; color: #1976d2;">{{ tag }}</span>{% unless forloop.last %} {% endunless %}
          {% endfor %}
        {% endif %}
      </p>
      <div class="entry" style="line-height: 1.6;">
        {{ post.excerpt }}
      </div>
      <a href="{{ post.url }}" style="display: inline-block; margin-top: 15px; padding: 8px 16px; background-color: #2196F3; color: white; text-decoration: none; border-radius: 4px; font-size: 0.9em;">Read Full Post →</a>
    </article>
  {% endfor %}
</div>

{% if site.posts.size == 0 %}
<div style="text-align: center; padding: 40px; background-color: #f8f9fa; border-radius: 8px; margin: 20px 0;">
  <h3>🚀 Coming Soon!</h3>
  <p>I'm preparing in-depth technical content covering MLOps, cloud architecture, and engineering leadership. Check back soon for insights from building ML platforms at scale!</p>
</div>
{% endif %}

---

## 💬 Let's Discuss

I love engaging with the technical community! If you have questions about any of my posts, want to discuss ML engineering challenges, or have ideas for future content, feel free to reach out.

**Connect with me:**
- 📧 [Email me](mailto:efrenbl@proton.me) for detailed technical discussions
- 💼 [LinkedIn](https://linkedin.com/in/efrenbl) for professional networking
- 💻 [GitHub](https://github.com/efrenbl) to see my code and contributions

---

## 📈 Subscribe for Updates

Want to stay updated on new posts? Connect with me on [LinkedIn](https://linkedin.com/in/efrenbl) where I share updates about new blog posts, interesting projects, and insights from the ML engineering world.

**Topics you can expect:**
- Building ML platforms at fintech scale
- Cloud security for ML applications  
- Engineering leadership lessons learned
- Career advice for ML engineers
- IEEE community insights and opportunities
