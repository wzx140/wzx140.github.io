---
layout: page
title: About
description: 只要我写的够快，bug就追不上我
comments: true
menu: 关于
permalink: /about/
---

Oft expectation fails, and most oft there

Where most it promises; and oft it hits

Where hope is coldest, and despair most fits.

<br>
**简单的编程爱好者**，只要我写的够快，bug就追不上我。

![]({{ site.url }}/assets/img/wzx_head.jpg)

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
</ul>
