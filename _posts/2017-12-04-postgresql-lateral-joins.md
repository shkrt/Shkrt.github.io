---
layout: default
title:  "Postgresql lateral joins"
date:   2017-12-04 22:57:21 +0300
tags: postgresql
---

# PostgreSQL lateral joins

Lateral is a relatively new concept for PostgreSQL, introduced in 9.3 version. At the time of writing of this post, there is already
10.1 version is out, but many of us had not even heard about the `lateral` feature. Basically, it should provide nicer syntax for some
occasions where we had to deal with PL/PGSQL procedures in versions prior to 9.3. But it is not only syntactic sugar and is
being interpreted by query planner in its own way.

The definition of `lateral` given in the official documentation can seem confusing to many of us, so basically `lateral`
keyword provides the ability to access columns of the `from` expression appearing in the same statement before `lateral` keyword.

{% highlight sql %}
SELECT field_1, field_2
FROM (subquery_1) a1
LEFT JOIN LATERAL (subquery_2) a2 ON condition
{% endhighlight %}

In the above pseudo-code, `subquery_2` can access each row of `subquery_1` and thus use them in the computation of each row.

Let's look at the simplest example implemented with both subquery and lateral join:

Given that we have the tables `articles` and `users` with following structure:

{% highlight sql %}
articles      users
--------     -------
id             id
recipient_id   email
title
status
{% endhighlight %}

Assuming that we want to select articles, assigned to given user and having given status code, but allowed to use only user's email, not id:

#### Subquery approach:

{% highlight sql %}
SELECT title, status, recipient_id FROM articles WHERE articles.status = 0 AND recipient_id =
(SELECT id FROM users WHERE users.email='email@example.com') LIMIT 10;

                                           QUERY PLAN
════════════════════════════════════════════════════════════════════════════════════════════════
 Limit  (cost=1.56..14.16 rows=1 width=141)
   InitPlan 1 (returns $0)
     ->  Seq Scan on users  (cost=0.00..1.14 rows=1 width=4)
           Filter: ((email)::text = 'email@example.com'::text)
   ->  Index Scan using index_articles_on_status on articles  (cost=0.42..13.02 rows=1 width=141)
         Index Cond: (status = 0)
         Filter: (recipient_id = $0)
{% endhighlight %}

#### Lateral approach:

{% highlight sql %}
SELECT title, status, u.id FROM articles m INNER JOIN LATERAL
(SELECT id FROM users WHERE users.email='email@example.com' and m.status=0 and users.id=m.recipient_id) u ON true LIMIT 10;

                                               QUERY PLAN
═════════════════════════════════════════════════════════════════════════════════════════════════════════
 Limit  (cost=1.57..14.21 rows=1 width=137)
   ->  Hash Join  (cost=1.57..14.21 rows=1 width=137)
         Hash Cond: (m.recipient_id = users.id)
         ->  Index Scan using index_articles_on_status on articles m  (cost=0.42..12.97 rows=23 width=141)
               Index Cond: (status = 0)
         ->  Hash  (cost=1.14..1.14 rows=1 width=4)
               ->  Seq Scan on users  (cost=0.00..1.14 rows=1 width=4)
                     Filter: ((email)::text = 'email@example.com'::text)
{% endhighlight %}

Explain output shows that in this very simple example lateral approach does not have any advantage over the subquery approach,
neither from point of readability nor from point of performance.

Let's tackle another, more complex example.

The example dataset represents a table of articles, each containing publication date,
its author's id. We will use the lateral joins to find the last article that was published by each of the authors.

Our articles table looks like this:

{% highlight sql %}
 title │ recipient_id │    published_at
═══════╪══════════════╪═════════════════════
 title │            2 │ 2010-07-20 10:12:43
 title │            3 │ 2013-09-12 12:40:54
 title │            4 │ 2003-11-25 09:04:14
 title │            5 │ 2005-04-18 06:17:01
 title │            1 │ 2005-08-25 07:39:42
 title │            1 │ 2009-11-11 14:42:41
 title │            1 │ 2010-04-23 11:33:37
 title │            2 │ 2015-11-23 10:36:00
 title │            3 │ 2003-05-21 10:47:24
 title │            4 │ 2008-03-18 14:04:49
(10 rows)
{% endhighlight %}

The result should look like this:

{% highlight sql %}
 title │   id   │      author_email
═══════╪════════╪═════════════════════════
 title │  17699 │ email1@example.com
 title │ 176013 │ email2@example.com
 title │ 176180 │ email3@example.com
 title │ 178341 │ email4@example.com
{% endhighlight %}

Of course, there are could be much more convenient ways to tackle this problem, but for demonstration purposes, we chose this example.
If we want to solve this task using lateral joins, we can approach the problem in the following way:

* FROM clause will select all existing users for us.

* Then we join articles table, order records by publication date and select first record

* Finally, we select title, id, and author_email from resulting dataset

Let's implement this steps, one-by-one:

Selecting all users:

{% highlight sql %}
SELECT id, email FROM users
{% endhighlight %}

Selecting corresponding articles:

{% highlight sql %}
SELECT title, id, recipient_id, published_at
FROM articles
WHERE recipient_id=u.id ORDER BY published_at asc limit 1
{% endhighlight %}

Join:

{% highlight sql %}
(SELECT id, email FROM users ) AS u
INNER JOIN LATERAL
(SELECT title, id, recipient_id, published_at FROM articles
  WHERE recipient_id=u.id ORDER BY published_at desc limit 1) AS m
ON true;
{% endhighlight %}

And finally selecting needed fields:

{% highlight sql %}
SELECT 'title' AS title, m.id, u.email AS author_email FROM
(SELECT id, email FROM users ) AS u
INNER JOIN LATERAL
(SELECT title, id, recipient_id, published_at FROM articles
  WHERE recipient_id=u.id ORDER BY published_at desc limit 1) AS m
ON true;
{% endhighlight %}

Lateral is not only meant to be used with joins because its core concept is re-using computed values of the
from a clause in subqueries. Thus it finds its application also in aggregates and custom functions.

Suggested reading:

[Reuse Calculations in the Same Query with Lateral Joins](https://www.periscopedata.com/blog/reuse-calculations-in-the-same-query-with-lateral-joins.html)
