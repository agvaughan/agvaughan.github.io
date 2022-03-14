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

<p>You can also find my work on my <a href="https://scholar.google.com/citations?user=4K8JDZEAAAAJ&hl=en">Google Scholar profile</a>.</p>

{% for post in site.publications reversed %}
  {% include archive-single.html %}
{% endfor %}
