---
layout: default
title: Home
---
<section class="hero">
  <h1>Hi, I'm {{ site.author }}.</h1>
  <p class="lead">{{ site.tagline }}</p>
</section>

<section>
  <div class="section-head">
    <h2>Recent posts</h2>
    <a href="{{ '/blog/' | relative_url }}">All posts &rarr;</a>
  </div>
  {% include post-list.html posts=site.posts limit=5 %}
</section>
