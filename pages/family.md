---
layout: simple
title: 云无心以出岫，鸟倦飞而知还。
description: 
keywords:
comments: false
menu: 家庭
permalink: /family/
---

> 聊乘化以归尽，乐夫天命复奚疑。

<section class="container content">
    <div class="columns">
        <div class="column" >
            <ol class="repo-list">
              {% for post in site.family %}
              {% if post.title != "Family Template"%}
              <li class="repo-list-item">
                <h3 class="repo-list-name">
                  <a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
                </h3>
                <p class="repo-list-description">
                {{ post.excerpt }}
                </p>
                <p class="repo-list-meta">
                <span class="meta-info">
                  <span class="octicon octicon-calendar"></span> {{ post.date | date: "%Y/%m/%d" }}
                </span>
                </p>
              </li>
              {% endif %}
              {% endfor %}
            </ol>
        </div>
     
    </div>

</section>
