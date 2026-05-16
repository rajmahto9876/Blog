---
title: Projects
layout: page
permalink: /projects/
icon: fas fa-tags
order: 4
---

{% assign project_posts = site.posts | where_exp: "item", "item.tags contains 'projects'" %}

{% if project_posts.size > 0 %}
{% for post in project_posts %}
- [{{ post.title }}]({{ site.baseurl }}{{ post.url }}) — {{ post.excerpt | strip_html | strip_newlines | truncate: 140 }}
{% endfor %}
{% else %}
No projects yet. Add `tags: [projects]` to your project posts.
{% endif %}
