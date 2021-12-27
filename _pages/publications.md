---
layout: archive
title: "Publications and preprints"
permalink: /publications/
author_profile: true
---

<!--
{% if author.googlescholar %}
  You can also find my articles on <u><a href="{{author.googlescholar}}">my Google Scholar profile</a>.</u>
{% endif %}
-->

{% include base_path %}

<!--
Thanks to https://github.com/academicpages/academicpages.github.io/issues/48#issuecomment-418460161

#{% for post in site.publications reversed %}
#  {% include archive-single.html %}
#{% endfor %}
-->

<h2>Preprints</h2>
{% for post in site.publications reversed %}
  {% if post.pubtype == 'preprint' %}
      {% include archive-single.html %}
  {% endif %}
{% endfor %}

<h2>Non-math publications and preprints</h2>
{% for post in site.publications reversed %}
  {% if post.pubtype == 'non-math' %}
      {% include archive-single.html %}
  {% endif %}
{% endfor %}
