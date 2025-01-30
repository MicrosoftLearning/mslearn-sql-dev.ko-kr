---
title: 온라인 호스팅 지침
permalink: index.html
layout: home
---

# Azure SQL 개발 연습

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %} [{{ activity.lab.title }}{% if activity.lab.type %} - {{ activity.lab.type }}{% endif %}]({{ site.github.url }}{{ activity.url }}) {% endfor %}


