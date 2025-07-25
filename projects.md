---
layout: page
title: Projects
published: true
permalink: /projects/
---
## Opensource

{% for project in site.projects reversed -%}
    {% if project.opensource -%}
      {%- if project.page -%}
- [{{project.path | remove: "_projects/" | remove: ".md"}}]({{project.page}})
     {% elsif project.github -%}
- [{{project.path | remove: "_projects/" | remove: ".md"}}]({{project.github}})
      {%- else -%}
- **{{project.path | remove: "_projects/" | remove: ".md"}}**
      {%- endif %} - {{project.title}}
    {% endif %}
{% endfor %}

## Private

{% for project in site.projects %}
    {% unless project.opensource %}
- {{project.path | remove: "_projects/" | remove: ".md"}} - {{project.title}}
    {% endunless %}
{% endfor %}
