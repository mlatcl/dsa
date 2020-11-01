---
layout: home
title: Data Science in Africa
---

This is a site that is *UNDER CONSTRUCTION* containing my material for lecturing on data science in the African context.

## Lectures

{% assign lastone = site.lectures | last %}
{% for lecture in site.lectures %}
{% include listlecture.html %}
{% endfor %}

