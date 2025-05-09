---
layout: default
title: Notes
permalink: /notes/
---

#### [Home](/) | [Blogs](/blogs/) | [Notes](/notes/) 

# Notes

Some interesting guides that I will add here. _Keep an eye out for the new updates!_

{% assign categories = site.notes | group_by: "category" %}

{% for category in categories %}
## {{ category.name }}

{% for note in category.items %}
- [**{{ note.title }}**]({{ note.url }}) <br>
  <small>{{ note.date | date: "%B %d, %Y" }}</small>
{% endfor %}

{% endfor %}