---
layout: page
title: About
description: 只要我写的够快，bug就追不上我
keywords: Zhuang Ma, 马壮
comments: true
menu: 关于
permalink: /about/
---

Oft expectation fails, and most oft there

Where most it promises; and oft it hits

Where hope is coldest, and despair most fits.


**简单的编程爱好者**

![]({{ site.url }}/assets/img/wzx_head.jpg)

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
{% if site.url contains 'masterwangzx.com' %}
<li>
</li>
{% endif %}
</ul>
