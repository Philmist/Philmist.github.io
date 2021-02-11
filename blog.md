---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: Blog
---

{% for post in site.posts %}
# [{{ post.title }}]({{ post.url | relative_url }})
{{ post.excerpt }}

[続きを読む]({{ post.url | relative_url }})

--------

{% endfor %}
