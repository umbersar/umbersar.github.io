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

I am always on the lookout for interesting SQL 'puzzles' and this was posted by [Brittany Bennett](https://twitter.com/thebmbennett). Instead of restating the problem, I will have you read the problem statement on her blog [here](https://invincible-failing-289.notion.site/SQL-Puzzle-Calculating-engagement-with-a-decay-function-661cda4a4e754cbaa45f42a5356138e7). 

## My solutions

Here I will present my solutions to the problem. Instead of using the toy dataset that was used to describe the problem, we will generate our own 'big data' so that we can compare the approaches to the problem.

So lets get to generating a table with 2000 records.

```sql
create database sqlpuzzle;
go

use sqlpuzzle;
go

if object_id(N'dbo.getnums', N'if') is not null drop function dbo.getnums;
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

drop table if exists basetable;
select dateadd(week,n,d) as [week], abs(checksum(newid())%9) as points_this_week,  CONVERT(decimal(10,3), null) as calcval
into basetable
from (values('2021-10-04')) as b(d)
cross apply getnums(1, 2000) as gn;
go
```

And there we have our very own big data (remember how big is 'big data' is very subjective!).

### Solution 1: Naive approach

The first solution is a naive iterative one in which we use two cursors, one nested inside another, to calculate the decay function for each row of data. Ran this query twice on my system and for both cases it took 19 seconds to get the results.

```sql
--Iterative algorithm. 2 cursors needed:
--1. iterate over rows in the table without (in any order) using a cursor. Lets call it a outer row.
--2. for each outer row, find the rows which have a [week] value less than or equal to that of outer row. Lets call them inner rows. Order the rows thus found in desc order of [week]. Now open a cursor to
--   iterate over the inner rows adding them using the exponential formula.

declare @startrow as datetime;
declare outerloop cursor 
	forward_only
	--read_only 
for 
	select [week] 
	from basetable;

open outerloop;
fetch next from outerloop into @startrow; 

while @@fetch_status = 0   
begin  
	declare @exponential as int = 0;
	declare @points_this_week as decimal(10,4) = 0.;
	declare @sumofpoints as decimal(10,4) = 0.;
	declare innerloop cursor
		forward_only
		read_only 
	for
		select points_this_week from basetable
		where [week] <= @startrow--starting from the current row, get all rows in desc order of [week]
		order by [week] desc;

		open innerloop
		fetch next from innerloop into @points_this_week
		while @@fetch_status = 0   
			begin
				set @sumofpoints = @sumofpoints + @points_this_week * power(cast(.9 as float), @exponential);
				set @exponential = @exponential + 1;
			 	fetch next from innerloop into @points_this_week;
			end
		close innerloop;
		deallocate innerloop;
		update basetable
			set calcval = @sumofpoints
		--where [week] = @startrow
		  where CURRENT OF outerloop;
 	fetch next from outerloop into @startrow;
end 

close outerloop;
deallocate outerloop;

select *
from basetable;
```

### Solution 2: Hybrid Cursor-Window_function approach

In this hybrid approach, we still use the outer loop cursor but inside of it, instead of using another cursor, we use window functions. Ran this query twice on my system and for both cases it took 4 seconds to get the results.

```sql
--Hybrid approach. 1 outer cursor and window function:
--1. iterate over rows in the table without (in any order) using a cursor. Lets call it a outer row.
--2. for each outer row, use a window function to do a exponential decay aggregation.

declare @startrow as datetime;
declare outerloop cursor 
	forward_only
	--read_only 
for 
	select [week] 
	from basetable;

open outerloop;
fetch next from outerloop into @startrow; 

WHILE @@FETCH_STATUS = 0   
BEGIN  
	declare @base as float = .9;
	declare @points_this_week as decimal(10,4) = 0.;
	declare @sumofPoints as decimal(10,4) = 0.;
	
	;with cte as
	(
		select points_this_week, row_number()over(order by [week] desc)-1 as exponent
		from basetable
		where [week] <= @startRow--starting from the current row, get all rows with desc orderd till the first row
	)	
	select @sumofPoints=sum(points_this_week * power(@base, exponent) )
	from cte;

	update basetable
		set calcVal = @sumofPoints
	--where [week] = @startRow
	WHERE current OF outerloop;
 	
	FETCH NEXT FROM outerLoop INTO @startRow;
END 

close outerloop;
deallocate outerloop;

select *
from basetable;
```

### Solution 3: Set based through and through

In this attempt, we used a set based approach doing away with cursors completely. Ran this query twice on my system and for both cases it took 1 second to get the results.

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

update bt
set calcval = expdecayaggregate
from basetable as bt
cross apply (select dbo.exponentialdecayaggregation(bt.week)) as tbl(expdecayaggregate);

select * 
from basetable;
```

## Clean up

```sql
use master;
go
alter database sqlpuzzle set single_user with rollback immediate;
drop database sqlpuzzle;
```

## Conclusion

We see that iterative approaches to computation took a lot more time to calculate the decay function. Set based queries outperform the iterative approaches.
