# No Loops solution using SQL

Recently came across a problem posed at [this site](https://ryxcommar.com/2019/08/05/a-cool-sql-problem-avoiding-for-loops/) . Basically, it is a test for a programmer to see if he can come up with a solution using a set-based, non-procedural approach to the problem. Two set-based solutions are already presented at the link. Here I present another solution. 

## What is the problem?
For the sake of completeness, here is a copy of text of the problem, verbatim, from the above link.

> You have a table of trading days (with no gaps) and close prices for a stock.
```TSQL
CREATE TABLE stock (
      trading_date DATE UNIQUE,
      price FLOAT
 );
```
> Find the highest and lowest profits (or losses) you could have made if you bought the stock at one close price and sold it at another close price, i.e. a total of exactly two transactions.

> You cannot sell a stock before it has been purchased. Your solution can allow buying and selling on the same trading_date (i.e. profit or loss of $0 is always, by definition, an available option); however, for some bonus points, you may write a more general solution for this problem that requires you to hold the stock for at least N days.

The problem also provides some synthetic data which I am copying here again for reproducibility purposes. 
```TSQL
INSERT INTO stock VALUES
('2015-06-01', 41),
('2015-06-02', 43),
('2015-06-03', 47),
('2015-06-04', 42),
('2015-06-05', 45),
('2015-06-08', 39),
('2015-06-09', 38),
('2015-06-10', 41);
```
Given this dataset, the biggest profit you can make is $6, and the smallest profit is -$9 (i.e. a loss of $9).

## My no loop solution

As stated before, the source link provides you with 2 solutions. Here is my solution to the problem:
```TSQL
;with CTE1(price, max_following_price, min_following_price) as
(
	select price
		, max(price) over(order by trading_date asc rows BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) 
		, min(price) over(order by trading_date asc rows BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) 
	from stock
),
CTE2 as
(
select max_following_price-price as greatest_potential_profit
	, min_following_price - price as smallest_potential_profit
from CTE1 
)
select MAX(greatest_potential_profit) as highest_profit
	, MIN(smallest_potential_profit) as lowest_profit
from CTE2
```

My [github profile](https://github.com/umbersar/). Tweet me [@umbersar](https://twitter.com/umbersar).
