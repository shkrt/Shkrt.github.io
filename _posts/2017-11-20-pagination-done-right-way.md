---
layout: default
title:  "Pagination done right way"
date:   2017-11-20 08:37:59 +0300
tags: postgresql activerecord
---

# Pagination done right way

Most of the times, when implementing pagination feature in applications, we use the most trivial approach - limit and offset
combination. Of course, in some cases this type of pagination is enough - when we do not have to care about
performance issues, for example, for small applications, blogs, lightweight admin interfaces and so on.

Now we will look at example application. It has `articles` table with about 700 thousand records. The table is not sharded/partitioned,
and has the following set of indexes:

{% highlight sql %}
CREATE INDEX index_articles_on_published_at_and_id ON articles USING btree (published_at, id);
CREATE INDEX index_articles_on_preview ON articles USING btree (preview);
CREATE INDEX index_articles_on_status ON articles USING btree (status);
CREATE INDEX index_articles_on_rubric ON articles USING btree (rubric);
{% endhighlight %}

ActiveRecord example:

{% highlight ruby %}
# current page number
page_number = 1

# articles count per each page
PER_PAGE = 10

Article.where(status: "published")
       .where(rubric: "society")
       .where.not(preview: true)
       .offset(page_number * PER_PAGE - PER_PAGE)
       .order(published_at: :desc, id: :desc)
       .limit(PER_PAGE)
       .select(:id, :title, :published_at)
{% endhighlight %}

This produces the following SQL query:

{% highlight sql %}
SELECT articles.id, articles.title, articles.published_at
FROM articles
WHERE articles.status = 'published'
  AND articles.rubric = 'society'
  AND articles.preview <> true
ORDER BY published_at desc, id desc
LIMIT 10 OFFSET 0
{% endhighlight %}

Let's look at the `explain` output for this.

```
Limit  (cost=0.42..15.47 rows=10 width=128)
  ->  Index Scan Backward using index_articles_on_published_at_and_id on articles  (cost=0.42..626361.59 rows=416338 width=128)
        Filter: ((NOT preview) AND (status = 'published') AND (rubric = 'society'))
```

This query's cost will increase with offset increasing. Let's check this
assumption by increasing the offset up to 2000:

```
Limit  (cost=3009.33..3024.38 rows=10 width=128)
  ->  Index Scan Backward using index_articles_on_published_at_and_id on articles  (cost=0.42..626361.59 rows=416338 width=128)
        Filter: ((NOT preview) AND (status = 'published') AND (rubric = 'society'))
```

The cost is already about 3000. What if we set offset to 20000? The table is big enough and our pagination theoretically
can reach this far:

```
Limit  (cost=30089.50..30104.54 rows=10 width=128)
  ->  Index Scan Backward using index_articles_on_published_at_and_id on articles  (cost=0.42..626361.59 rows=416338 width=128)
        Filter: ((NOT preview) AND (status = 'published') AND (rubric = 'society'))
```

The cost numbers are terrifying now. And they will grow even bigger. The trick has been explained in [this](http://use-the-index-luke.com/no-offset)
blog post and numerous conference talks. The problem is, that on every page, we have to scan through `+ offset` amount of records.
Given that limit parameter is 10 when the offset is 0, we have to scan through 10 records. When the offset is 200, we have to go through 200 records,
when 2000 - 2000 records, and so on. So, the bigger the page is, the bigger the transaction cost becomes. To solve this problem,
we can implement keyset pagination. The keyset pagination does not rely on offset and simply uses a combination
of the values of the last fetched rows. Not all the values, but the ones that we use for ordering. In our example, we order by
`published_at` and `id` columns. To proceed to the next page, we should memorize `published_at` and `id` values for the
last fetched row, and take `limit` amount of rows, that have `published_at` and `id` values bigger (or smaller, depending on
ordering direction) than memorized value. For example, the last row from the first page can be this:

```
   id    |                                    title                                    |        published_at
---------+-----------------------------------------------------------------------------+----------------------------
 1030984 | A-Bank Card Holders Are No Longer In Charge                                 | 2017-11-19 14:50:17.18685
```

So our pagination SQL query will now look like this without using offset:

{% highlight sql %}
SELECT articles.id, articles.title, articles.published_at
FROM articles
WHERE articles.status = 'published'
  AND articles.rubric = 'society'
  AND articles.preview <> true
  AND published_at < '2017-11-19 14:50:17.18685'
  AND id < 1030984
ORDER BY published_at desc, id desc
LIMIT 10 OFFSET 0
{% endhighlight %}

The output of explain for the first page now looks like this:

```
Limit  (cost=0.42..14.29 rows=10 width=128)
  ->  Index Scan Backward using index_articles_on_published_at_and_id on articles  (cost=0.42..627756.86 rows=412252 width=128)
        Index Cond: ((published_at < '2017-11-19 14:50:17.18685'::timestamp without time zone) AND (id < 1030984))
        Filter: ((NOT preview) AND (status = 'published') AND (rubric = 'society'))
```

And for the record, that was created about two weeks ago and roughly matches the offset of 2000:

```
Limit  (cost=0.42..14.29 rows=10 width=128)
   ->  Index Scan Backward using index_articles_on_published_at_and_id on articles  (cost=0.43..627755.37 rows=412212 width=128)
         Index Cond: ((published_at < '2017-11-04 08:52:29.889916'::timestamp without time zone) AND (id < 1022805))
         Filter: ((NOT preview) AND (status = 'published') AND (rubric = 'society'))
```

As you can see, with this approach, increasing page number has no performance impact at all. Of course, this kind of pagination
requires the presence of a respective index, in this example, it is `published_at, id` index.
Also, we can use PostgreSQL's row() function for the same purpose, when we do not order by autoincrementing column (id in our example)

{% highlight sql %}
SELECT articles.id, articles.title, articles.published_at
FROM articles
WHERE articles.rubric = 'society'
  AND articles.preview <> true
  AND row(published_at, status) < ('2017-11-19 14:50:17.18685', 5)
ORDER BY published_at desc, id status
LIMIT 10 OFFSET 0
{% endhighlight %}

Of course, all of these queries can easily be translated to your favorite ORM's DSL. As a conclusion, I can advise not to use
offset pagination on the relatively big datasets. Even when the records count exceeds only tens of thousands, `offset's` impact
on performance is becoming noticeable. But the use of offset is ok for applications operating on the relatively small
datasets.

Suggested reading:

[Markus Winand blogpost](http://use-the-index-luke.com/no-offset)

[Five ways to paginate in Postgres, from the basic to the exotic](https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/)
