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

## What is the problem?

I was working on a use case where I wanted to generate some synthetic data. In such scenarios I always find Itzik Ben-Gan's [GetNums](https://tsql.solidq.com/SourceCodes/GetNums.txt) to be very useful. Now for each row of data that I was generating, I also wanted a associated random number. Thinking that it is easy to do as T-SQL provides a random number generator function, [RAND()](https://docs.microsoft.com/en-us/sql/t-sql/functions/rand-transact-sql?view=sql-server-ver15), I cross applied the call to RAND. But, guess what, it gave me the same  random number for every row of synthetic data I generated. That was perplexing. I assumed that for RAND will be called for each row and, thus, I would get a new random number per row. This expectaion was not out of the ordniary as this is the behaviour exhibited by some other functions. For example, if I cross applied [NEWID](https://docs.microsoft.com/en-us/sql/t-sql/functions/newid-transact-sql?view=sql-server-ver15), then I would get a different Id for each row. The following example code and output shows the results.

```sql
select *
from TSQLV3.dbo.GetNums(1,3) 
cross apply (select RAND()) as rnd(random_num)
cross apply (select NEWID()) as Id(id)

n                    random_num             id
-------------------- ---------------------- ------------------------------------
1                    0.0254042467507439     4DBC8945-0D80-4DC7-9941-B1F7984B111E
2                    0.0254042467507439     063CABCA-53E3-4B53-8E0C-E4B70186E6D4
3                    0.0254042467507439     4A544748-D5FC-437E-B825-314CE8BA0468
```

## Why is this happening?

So that brings us to the question of WHY. It seems what is happening is that SQL Server query optimizer is executing the call to RAND once and then using that result for each row. with the calls (https://en.wikipedia.org/wiki/Constant_folding)
