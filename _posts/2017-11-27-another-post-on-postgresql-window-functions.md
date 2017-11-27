---
layout: default
title:  "Another post on PostgreSQL window functions"
date:   2017-11-27 17:25:13 +0300
tags: postgresql
---

# Another post on PostgreSQL window functions

There are plenty of posts on web, describing PostgreSQL window functions, and I decided to make another one, to share my
view and my approach to understanding them.

Queries using window functions usually look like this one:

{% highlight sql %}
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;

  depname  | empno | salary |          avg
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
(10 rows)
{% endhighlight %}

copied from [official documentation](https://www.PostgreSQL.org/docs/9.6/static/tutorial-window.html)

To follow along, you can use the following setup:

{% highlight sql %}
createdb window_test --host=localhost --user=postgres

create table empsalary(
depname character varying,
empno integer,
salary integer);

insert into empsalary(depname, empno, salary) values
('develop', 11, 5200),
('develop', 7, 4200),
('develop', 9, 4500),
('develop', 8, 6000),
('develop', 10, 5200),
('personnel', 5, 3500),
('personnel', 2, 3900),
('sales', 3, 4800),
('sales', 1, 5000),
('sales', 4, 4800);
{% endhighlight %}

Window function part is starting from `avg(salary)`. The window functions introduce the concepts of `window frames`
In the above example, `avg(salary)` is the function, that is being applied to every row in the output.
The frame is representing the relation that is available to `avg(salary)` on the current row. In the example, the frame is
defined by `PARTITION BY depname` part. The whole relation available to window function initially, after filtering clauses,
is being partitioned by `depname` column's value. It results in

`(5200 + 4200 + 4500 + 6000 + 5200)/5 = 5020.0`

And we get the value of `5020.0` in every row related to `develop` depname. The same applies to `personnel` and `sales` depnames.

The frame is defined either by `partition by` or `order by` clauses, otherwise, the defaults are applied.

* when `partition by` is used - the frame consists of the whole partition

* when `order by` is used - the frame includes all records up to current row. Further records are omitted

{% highlight sql %}
SELECT depname, empno, salary, avg(salary) OVER (ORDER BY empno) FROM empsalary;

  depname  │ empno │ salary │          avg
═══════════╪═══════╪════════╪═══════════════════════
 sales     │     1 │   5000 │ 5000.0000000000000000
 personnel │     2 │   3900 │ 4450.0000000000000000
 sales     │     3 │   4800 │ 4566.6666666666666667
 sales     │     4 │   4800 │ 4625.0000000000000000
 personnel │     5 │   3500 │ 4400.0000000000000000
 develop   │     7 │   4200 │ 4366.6666666666666667
 develop   │     8 │   6000 │ 4600.0000000000000000
 develop   │     9 │   4500 │ 4587.5000000000000000
 develop   │    10 │   5200 │ 4655.5555555555555556
 develop   │    11 │   5200 │ 4710.0000000000000000
(10 rows)
{% endhighlight %}

Here we see the illustration of frame scoping to all the rows up to the current row. We have ordered the relation by `empno`
column value. On the first row, `avg(salary)` takes into account only first row, on the second row - first row + second row, and so on.

* when both clauses are omitted, the frame includes the whole relation

{% highlight sql %}
SELECT depname, empno, salary, avg(salary) OVER () FROM empsalary;
  depname  │ empno │ salary │          avg
═══════════╪═══════╪════════╪═══════════════════════
 develop   │    11 │   5200 │ 4710.0000000000000000
 develop   │     7 │   4200 │ 4710.0000000000000000
 develop   │     9 │   4500 │ 4710.0000000000000000
 develop   │     8 │   6000 │ 4710.0000000000000000
 develop   │    10 │   5200 │ 4710.0000000000000000
 personnel │     5 │   3500 │ 4710.0000000000000000
 personnel │     2 │   3900 │ 4710.0000000000000000
 sales     │     3 │   4800 │ 4710.0000000000000000
 sales     │     1 │   5000 │ 4710.0000000000000000
 sales     │     4 │   4800 │ 4710.0000000000000000
{% endhighlight %}

`avg` is now applied to the whole table.

The `avg` function used in the previous example is an aggregation function.
Window functions can be used with aggregate functions, or with so-called built-in window functions.
List of built-in window functions can be viewed [here](https://www.PostgreSQL.org/docs/9.6/static/functions-window.html)

{% highlight sql %}
postgres@window_test=#  SELECT depname, empno, salary, rank() OVER (partition by depname order by salary desc) FROM empsalary;
  depname  │ empno │ salary │ rank
═══════════╪═══════╪════════╪══════
 develop   │     8 │   6000 │    1
 develop   │    10 │   5200 │    2
 develop   │    11 │   5200 │    2
 develop   │     9 │   4500 │    4
 develop   │     7 │   4200 │    5
 personnel │     2 │   3900 │    1
 personnel │     5 │   3500 │    2
 sales     │     1 │   5000 │    1
 sales     │     3 │   4800 │    2
 sales     │     4 │   4800 │    2
(10 rows)
{% endhighlight %}

Another thing to note is that window functions are evaluated after filtering and grouping clauses, therefore we cannot
use grouping and filtering over windowing results.

Suggested reading:

[List of window functions](https://www.PostgreSQL.org/docs/9.6/static/functions-window.html)

[Window functions](https://www.PostgreSQL.org/docs/9.6/static/tutorial-window.html)

[Syntax](https://www.PostgreSQL.org/docs/9.6/static/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)

[Window function processing](https://www.PostgreSQL.org/docs/9.6/static/queries-table-expressions.html#QUERIES-WINDOW)
