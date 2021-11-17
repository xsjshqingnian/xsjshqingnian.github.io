---
layout: page
title: Wiki
description: 一些分享
keywords: Wiki
comments: false
menu: 维基
permalink: /wiki/
---

> 三千弱水，我都要

<ul class="listing">
{% for wiki in site.wiki %}
{% if wiki.title != "Wiki Template" and wiki.topmost == true %}
<li class="listing-item"><a href="{{ site.url }}{{ wiki.url }}"><span class="top-most-flag">[置顶]</span>{{ wiki.title }}</a></li>
{% endif %}
{% endfor %}
{% for wiki in site.wiki %}
{% if wiki.title != "Wiki Template" and wiki.topmost != true %}
<li class="listing-item"><a href="{{ site.url }}{{ wiki.url }}">{{ wiki.title }}<span style="font-size:12px;color:red;font-style:italic;">{%if wiki.layout == 'mindmap' %}  mindmap{% endif %}</span></a></li>
{% endif %}
{% endfor %}
</ul>
