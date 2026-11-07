---
layout: default
title: Home
---
<section class="hero">
  <img class="avatar" src="{{ '/assets/images/profile.jpg' | relative_url }}" alt="Photo of {{ site.author }}" width="120" height="120">
  <div>
    <h1>Hi, I'm {{ site.author }}.</h1>
    <p class="bio">Hey! I'm a computer engineering student at University of Waterloo, interested in robot learning. Previously, I worked on humanoid grasp manipulation at Apptronik. My current fixation is the application of world models in manipulation. In my free time, I like eating, reading, rock climbing, and snowboarding.</p>
    {% include social-links.html %}
  </div>
</section>

<section>
  <div class="section-head">
    <h2>Recent posts</h2>
    <a href="{{ '/blog/' | relative_url }}">All posts &rarr;</a>
  </div>
  {% include post-list.html posts=site.posts limit=5 %}
</section>
