---
layout: page
title: Jeroen Serpieters
---
{% include JB/setup %}

{{ site.tagline }}

{% for post in site.posts limit: 5 %}
<div class="blog-index">
  {% assign content = post.content %}
  {% include post_detail.html %}
</div>
{% endfor %}
