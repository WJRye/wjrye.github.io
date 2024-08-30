---
layout: simple
title: 云无心以出岫，鸟倦飞而知还。
description: 
keywords:
comments: false
menu: 家庭
permalink: /family/
---
<blockquote>
        聊乘化以归尽，乐夫天命复奚疑。
        <cite>– 陶渊明</cite>
</blockquote>



<section>
{% assign sorted = site.family | sort: 'date' | reverse  %}
{% for post in sorted %}
{% if post.title != "Family Template"%}
<h3>
    <a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
</h3>
<p>{{ post.excerpt }}</p>
<p><span><span class="octicon octicon-calendar"></span> {{ post.date | date: "%Y/%m/%d" }}</span></p>
<hr style="height:1px">
{% endif %}
{% endfor %}

</section>
