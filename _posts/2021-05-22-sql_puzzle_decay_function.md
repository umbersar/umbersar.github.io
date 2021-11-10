---
title: "SQL Puzzle: Calculating engagement with a decay function"
date: 2021-11-10T21:06:00+00:00
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

I am always on the lookout for interesting SQL 'puzzles' and this was posted by at [Brittany Bennett](https://twitter.com/thebmbennett). Instead of restating the problem, I will have you read the problem statement on her blog [read here](https://invincible-failing-289.notion.site/SQL-Puzzle-Calculating-engagement-with-a-decay-function-661cda4a4e754cbaa45f42a5356138e7). 

## My solutions

Here I will present my solutions to the problem. Instead of using the toy dataset that was used to describe the problem, we will generate our own 'big data' so that we can compare the approaches to the problem.

So lets get to generating a table with 1000 records.

```sql
create database sqlpuzzle;
go

if object_id(n'dbo.getnums', n'if') is not null drop function dbo.getnums;
go
create function dbo.getnums(@low as bigint, @high as bigint) returns table
as
return
  with
    l0   as (select c from (select 1 union all select 1) as d(c)),
    l1   as (select 1 as c from l0 as a cross join l0 as b),
    l2   as (select 1 as c from l1 as a cross join l1 as b),
    l3   as (select 1 as c from l2 as a cross join l2 as b),
    l4   as (select 1 as c from l3 as a cross join l3 as b),
    l5   as (select 1 as c from l4 as a cross join l4 as b),
    nums as (select row_number() over(order by (select null)) as rownum
             from l5)
  select top(@high - @low + 1) @low + rownum - 1 as n
  from nums
  order by rownum;
go

drop table if exists basetable
select dateadd(week,n,d) as [week], abs(checksum(newid())%9) as points_this_week
into basetable
from (values('2021-10-04')) as b(d)
cross apply getnums(1, 1000) as gn
go
```

And there we have our very own big data (remember how big is 'big data' is very subjective!).

### Solution 1: Naive approach

The first solution is a naive iterative one in which we use two cursors, one nested inside another, to calculate the decay function for each row of data.

```sql
drop table if exists #ttorder
create table #ttorder
(
[week]  varchar(20),
points_this_week  decimal(10,4),
ascnum  int,
dscnum  int,
calcval  decimal(10,4)
)
insert into #ttorder
select *, row_number() over(order by [week]) as ascnum, row_number() over(order by [week] desc) as dscnum, null as calcval
from #tt;

--select *
--from #ttorder;

--iterative algorithm:
--1. order the rows of data in order of date (asc): cursor needed
--2. for each row thus ordered, starting from lowest in order, iterate over that row and all the previous rows adding them using the exponential formula: cursor needed


declare @startrow as int;
declare outerloop cursor 
	forward_only
	read_only 
for 
	select [ascnum] from #ttorder
	order by [ascnum]

open outerloop
fetch next from outerloop into @startrow  

while @@fetch_status = 0   
begin  
	declare @exponential as int = 0;
	declare @points_this_week as decimal(10,4) = 0.;
	declare @sumofpoints as decimal(10,4) = 0.;
	declare innerloop cursor
		forward_only
		read_only 
	for
		select points_this_week from #ttorder
		where [ascnum] <= @startrow--starting from the current row, get all rows with desc orderd till the first row
		order by [ascnum] desc

		open innerloop
		fetch next from innerloop into @points_this_week
		while @@fetch_status = 0   
			begin
				set @sumofpoints = @sumofpoints + @points_this_week * power(cast(.9 as float), @exponential);
				set @exponential = @exponential + 1
			 	fetch next from innerloop into @points_this_week
			end
		close innerloop 
		deallocate innerloop 
		update #ttorder
			set calcval = @sumofpoints
		where [ascnum] = @startrow
 	fetch next from outerloop into @startrow  
end 

close outerloop
deallocate outerloop 

select *
from #ttorder;
```

### Solution 2: Hybrid Cursor-Window_function approach

Well, to test the hypothesis that optimizer decides to do Constant Folding since the calls to RAND are independent of data in each row, we make the call to RAND depend on a changing column value in the row on the left had side of cross apply (by writing a corelated query). That way we make sure that compiler won't treat the call to RAND as an constant expression and as the argument to RAND changes, a new random number is generated as expected. That all works but how to prevent Constant Folding without using seed value for RAND through corelated subquery?

```sql

```

### Solution 3: Set based through and through

One trick that could work is to abstract away the call to RAND behind another layer of indirection such that optimizer cannot decide if it is a appropriate case to do Constant Folding. We can try to put the RAND function call in a User Defined Function (UDF) or behind a View. Lets try using a view first. But it does not prevent Constant Folding and we get the same random value for all the rows.

```sql
create or alter function dbo.exponentialdecayaggregation(@currentweek varchar(20))  
returns decimal(10,3)   
as   
begin  
	declare @base as float = .9;
	declare @sumofpoints as decimal(10,3) = 0.;
	;with cte as
	(
		select points_this_week, row_number() over(order by [week] desc)-1 as exponent
		from basetable
		where [week] <= @currentweek
	)	
	select @sumofpoints=sum(points_this_week * power(@base, exponent) )
	from cte

	return @sumofpoints;  
end; 
go

select * 
from basetable as bt
cross apply (select dbo.exponentialdecayaggregation(bt.week)) as tbl(expdecayaggregate)

```

## Clean up

But if we abstract away the call to RAND in the UDF behind a view it all works and optimizer is not able to do Constant Folding. 

```sql
drop database sqlpuzzle
```

## Conclusion

In the end, .
