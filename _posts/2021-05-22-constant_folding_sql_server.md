---
title: "Constant folding in SQL Server and workaround"
date: 2021-05-22T09:09:57+00:00
toc: true
toc_label: "On this page"
toc_icon: "file-alt"
# classes: wide #moved this setting to /_layouts/single.html page
categories:
  - Blog
tags:
  - T-SQL
---

I was working on a use case where i wanted to generate some synthetic data. For each row of data that i was generating, I also wanted a associated random number. 
