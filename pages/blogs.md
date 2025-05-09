---
layout: default
title: Blogs
permalink: /blogs/
---

#### [Home](/) | [Blogs](/blogs/) | [Notes](/notes/) 

# Blogs
Some interesting findings that I will add here. _Keep an eye out for the new updates!_

{% for post in site.posts %}
- [**{{ post.title }}**]({{ post.url }}) <br>
  <small>{{ post.date | date: "%B %d, %Y" }}</small>
{% endfor %}