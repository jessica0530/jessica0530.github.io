---
layout: page
title: About
description: 
keywords: 
comments: true
menu: 关于
permalink: /about/
---



## 关于我

现就职于字节上海,

原 上海著名电商公司 Flink团队核心成员

上海 大数据领域 7年左右

2020年中发现自己记性衰退,开始梳理沉淀和总结归纳 行业知识



## 联系

telephone number : 15821661776 

Mail : wjxdtc10530@gmail.com




## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
