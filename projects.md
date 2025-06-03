---
layout: page
title: Projects
permalink: /projects/
---

Here are some of the projects I've worked on:

_{% for project in site.projects %}_
## [{{ project.title }}]({{ project.url }})
{{ project.excerpt }}
_{% endfor %}_

More details will be added soon!
