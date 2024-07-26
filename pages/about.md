---
layout: page
title: About
description: 量身而行
keywords: bender
comments: true
menu: 关于
permalink: /about/
---

Hi, is bender



## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
