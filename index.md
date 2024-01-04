---
title: Azure OpenAI 演習
permalink: index.html
layout: home
---

# SQL Server 移行の演習

次の演習は、Microsoft Learn の「[SQL Server ワークロードを Azure SQL に移行する](https://learn.microsoft.com/training/paths/migrate-sql-workloads-azure/)」ラーニング パスのモジュールをサポートするように設計されています。

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %}
- [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) {% endfor %}