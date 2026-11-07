---
layout: default
title: Home
---
<section class="hero">
  <img class="avatar" src="{{ '/assets/images/profile.jpg' | relative_url }}" alt="Photo of {{ site.author }}" width="120" height="120">
  <div>
    <h1>Hi, I'm {{ site.author }}.</h1>
    <p class="lead">{{ site.tagline }}</p>
    <p class="hero-links"><a href="{{ '/about/' | relative_url }}">About me</a> &middot; <a href="{{ '/assets/lily_resume.pdf' | relative_url }}">Resume (PDF)</a></p>
  </div>
</section>

<section>
  <div class="section-head">
    <h2>Recent posts</h2>
    <a href="{{ '/blog/' | relative_url }}">All posts &rarr;</a>
  </div>
  {% include post-list.html posts=site.posts limit=5 %}
</section>
