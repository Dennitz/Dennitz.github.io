---
layout: post
title: "Projects"
permalink: /projects/
---

<div class="projects">
{% for project in site.data.projects %}
  <div class="project">
    <div class="project-image">
      <a href="{{project.link}}">
        <img src="{{project.image}}" alt="{{project.alt}}" />
      </a>
    </div>
    <div class="project-description">
      {{project.description}}
    </div>
  </div>
{% endfor %}
</div>
