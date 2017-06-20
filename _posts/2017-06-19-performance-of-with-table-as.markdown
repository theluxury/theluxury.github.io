---
layout: post
title:  "Perform Hit of Using Temporary Tables"
date:   2017-06-19 16:00:00 -0700
categories: Query Optimization
---

**tl;dr:** Be very careful when using temp tables, especially when filtering against results from them. Performance using it seems very variable and can have up to 100x performance hits. If possible, compare your queries using temporary tables with hardcoded queries that don't use them.

### Filtering on with table as ... Performance

Pop quiz: We have a view `fact.user_tracking_view` and with a sort key of `time_occurred_pst`. Below are some queries that return how many events happened yesterday. Can you guess the relative performance of each?

{% highlight sql %}
-- First Query
with yesterday as (select (current_timestamp at time zone 'PST8PDT' - interval '1 day')::date::timestamp)
select count(*)::decimal from fact.user_tracking_view 
	where time_occurred_pst > (select timestamp from yesterday)
    AND time_occurred_pst < ((select timestamp from yesterday) + interval '1 day');

-- Second Query
select count(*)::decimal from fact.user_tracking_view 
	where time_occurred_pst > (current_timestamp at time zone 'PST8PDT' - interval '1 day')::date::timestamp
	AND time_occurred_pst < (current_timestamp at time zone 'PST8PDT' )::date::timestamp;
{% endhighlight %}

If you're familiar with Postgres/Redshift, you might think the first is slightly slower but justified with improved readability, like this [blog post][with-as-blog] suggests. However, the **second query is actually ~100x faster**. For my Redshift table, the first finishes in around 20 seconds, whereas the second query takes around 200 milliseconds. **The performance hit is HUUUUUUUUUUUUUUUUUUUUUUUUGGGGGGGGGGGGGGGEEEEEEEEEEEEEEE.**

### What's the deal?

Doing an explain query on both, the differences is the slower query has to scan through more rows. (`user_tracking_2017_05` is an example of a table that makes up `fact.user_tracking_view`.)

{% highlight sql %}           
-- Slower query explain path example
->  XN Seq Scan on user_tracking_2017_05  (cost=0.00..4351900.16 rows=1087976 width=8)
    Filter: ((time_occurred_pst < ($0 - '6 days'::interval)) AND (time_occurred_pst > ($1 - '7 days'::interval)))

-- Faster query explain path example
->  XN Subquery Scan "*SELECT* 4"  (cost=0.01..0.02 rows=1 width=0)
    ->  XN Aggregate  (cost=0.01..0.01 rows=1 width=0)
        ->  XN Seq Scan on user_tracking_2017_05  (cost=0.00..0.00 rows=1 width=0)
            Filter: ((time_occurred_pst > '2017-06-11 00:00:00'::timestamp without time zone) AND (time_occurred_pst < '2017-06-12 00:00:00'::timestamp without time zone))
{% endhighlight %}

The differences in cost between `(cost=0.00..4351900.16 rows=1087976 width=8)` and `(cost=0.01..0.02 rows=1 width=0)` suggest that in the faster query is able to completely skip `user_tracking_2017_05` since it has no records that satisfy the filter condition, whereas the slower faster still needs to examine 1087976 rows. Note that 1087976 is much less than the total count of `user_tracking_2017_05` which is 217595067, so it's probably scanning every block of presorted analyzed data. 

Looking carefully at the query, you can see the slower one's filter condition is `((time_occurred_pst < ($0 - '6 days'::interval)) AND (time_occurred_pst > ($1 - '7 days'::interval)))` whereas the faster query's filter condition is `((time_occurred_pst > '2017-06-11 00:00:00'::timestamp without time zone) AND (time_occurred_pst < '2017-06-12 00:00:00'::timestamp without time zone))` Note the slower has $0 and $1. Therefore, it seems like the **faster query can optimize using the existing timestamp, whereas the slower query has to adhoc select from from the temporary table to compare to each data block.** 

So, the lesson may be more general than just being careful using `with table as ...`, it may be **be careful filtering by selects from other tables when you can filter by hard coded values.**

[with-as-blog]: http://www.craigkerstiens.com/2013/11/18/best-postgres-feature-youre-not-using/