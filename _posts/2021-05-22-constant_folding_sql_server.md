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

I was working on a use case where I wanted to generate some synthetic data. In such scenarios I always find Itzik Ben-Gan's [GetNums](https://tsql.solidq.com/SourceCodes/GetNums.txt) to be very useful. Now for each row of data that I was generating, I also wanted a associated random number. Thinking that it is easy to do as T-SQL provides a random number generator function, [RAND()](https://docs.microsoft.com/en-us/sql/t-sql/functions/rand-transact-sql?view=sql-server-ver15), I cross applied the call to RAND. But, guess what, it gave me the same  random number for every row of synthetic data I generated. That was perplexing. 

```sql
select *
from TSQLV3.dbo.GetNums(1,3) 
cross apply (select RAND()) as rnd(num)

n                    num
-------------------- ----------------------
1                    0.197917491935496
2                    0.197917491935496
3                    0.197917491935496
```
