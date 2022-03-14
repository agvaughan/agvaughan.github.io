---
layout: archive
title: "Selected Publications"
permalink: /publications/
author_profile: true
---

<!-- {% if author.googlescholar %}
 -->  
<!-- {% endif %}
 -->
{% include base_path %}

<p>You can also find my articles on <u><a href="{{author.googlescholar}}">my Google Scholar profile</a>.</u></p>

{% for post in site.publications reversed %}
  {% include archive-single.html %}
{% endfor %}
