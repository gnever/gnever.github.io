---
layout: page
title: Wiki
description: 
keywords:  Wiki
comments: false
menu: Wiki
permalink: /wiki/
---

> case 

<ul class="listing">
{% for wiki in site.wiki %}
{% if wiki.title != "Wiki Template" %}
<li class="listing-item"><a href="{{ wiki.url }}">{{ wiki.title }}</a></li>
{% endif %}
{% endfor %}
</ul>
