---
layout: page
title: Jeroen Serpieters
---
{% include JB/setup %}

{{ site.tagline }}

<div class="blog-index">
  {% assign post = site.posts.first %}
  {% assign content = post.content %}
  {% include post_detail.html %}
</div>
