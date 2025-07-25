{% assign exclude = include.exclude | default: "" %}
{% assign inc = include.include | default: "" %}

{% for post in site.posts %}
{% if post.categories contains inc or inc == "" %}
{% unless post.categories contains exclude or exclude == "" %}
- [{{ post.title }}]({{ post.url }})
{% endunless %}
{% endif %}
{% endfor %}
