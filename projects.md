---
title: "Projects"
permalink: /projects
layout: default
---

# Projects

Here are some of the projects I've worked on:

{% for project in site.projects %}
## [{{ project.title }}]({{ project.url }})

{{ project.description }}

{% endfor %}
