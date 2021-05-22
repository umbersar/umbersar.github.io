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

I was working on a use case where I wanted to generate some synthetic data. In such scenarios I always find Itzik Ben-Gan's [GetNums](https://tsql.solidq.com/SourceCodes/GetNums.txt) to be very useful. Now for each row of data that I was generating, I also wanted a associated random number. Thinking that it is easy to do as T-SQL provides a random number generator function, [RAND()](https://docs.microsoft.com/en-us/sql/t-sql/functions/rand-transact-sql?view=sql-server-ver15), I cross applied the call to RAND. But, guess what, it gave me the same  random number for every row of synthetic data I generated. That was perplexing. I assumed that RAND will be called for each row and, thus, I would get a new random number per row. This expectation was not out of the ordinary as this is the behaviour exhibited by some other functions. For example, if I cross applied [NEWID](https://docs.microsoft.com/en-us/sql/t-sql/functions/newid-transact-sql?view=sql-server-ver15), then I would get a different Id for each row. The following example code and output shows the results.

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

So that brings us to the question of WHY. It looks like that SQL Server query optimizer is executing the call to RAND once and then using that result for each row. It seems that the optimzer decides that since the calls to RAND are independent of data in each row, it can apply [Constant Folding](https://en.wikipedia.org/wiki/Constant_folding). Constant Folding evaluates an expression once to get a result and then replaces the call to that expression with the result at compile time. But it did not apply Constant Folding for calls to NEWID so to me it is not consistent behaviour on part of optimizer.

## How to prevent constant folding?

Well, one way to stop the compiler from doing Constant Folding is to have the call to RAND depend on a changing column value in the row on left had side of cross apply by wirtting a corelated query. That way we make sure that compiler won't treat the call to RAND as an constant expression and as the argument to RAND changes, a new random number is generated as expected. That all works but how to prevent Constant Folding without using seed value for RAND through corelated subquery?

```sql
select *
from TSQLV3.dbo.GetNums(1,3) 
cross apply (select RAND(n)) as rnd(random_num)
cross apply (select NEWID()) as Id(id)

n                    random_num             id
-------------------- ---------------------- ------------------------------------
1                    0.713591993212924      EEB1911D-DB14-44AC-8961-CEFD91902A54
2                    0.713610626184182      58423EC3-4DC9-43CC-9C3A-F8E8F0E9EC03
3                    0.71362925915544       9DAFACD2-7B69-4299-86D1-300147F0CF96
```

## How to prevent constant folding without providing a seed value for RAND ?

One trick that could work is to abstract away the call to RAND behind another layer of indirection such that optimizer cannot decide if it is a appropriate case to do Constant Folding. We can try to put the RAND function call in a User Defined Function (UDF) or behind a View. Lets try using a view first. But it does not prevent Constant Folding and we get the same random value for all the rows.

```sql
create or alter view getrandvalue
as
	select rand() as randvalue;
go

select *
from TSQLV3.dbo.GetNums(1,3) 
cross apply (select randvalue from getrandvalue) as v

n                    randvalue
-------------------- ----------------------
1                    0.39118403703769
2                    0.39118403703769
3                    0.39118403703769

```

Second, we try adding a UDF to encapsulate the call to RAND. Here is the code for that and it does not work as we cannot call RAND from inside a user-defined function. The UDF creation fails with the message:
> Invalid use of a side-effecting operator 'rand' within a function.

```sql
create or alter function dbo.randomValueGenerator() 
returns int
as
begin
  return (select rand())
end;
go
```

But if we abstract away the call to RAND in the UDF behind a view it all works and optimizer is not able to do Constant Folding.

```sql
create or alter view getrandvalue
as
	select rand() as randvalue;
go

create or alter function dbo.randomValueGenerator() 
returns float
as
begin
  return (select randvalue from getrandvalue)
end;
go

select *
from TSQLV3.dbo.GetNums(1,3) 
cross apply (select dbo.randomValueGenerator()) as udf(random_num) 
go

n                    random_num
-------------------- ----------------------
1                    0.333414481261585
2                    0.332958889887198
3                    0.650838151152864
```

## Conclusion

In the end, adding another layer of abstraction causes the optimizer to stop Constant Folding but the overall behaviour of optimizer is not consistent. NEWID works as expected without us having to add extra layers of indirection but for RAND we have to. Strange. It is not only RAND but [GETDATE](https://docs.microsoft.com/en-us/sql/t-sql/functions/getdate-transact-sql?view=sql-server-ver15) also gets Constant Folded away at compile time.
