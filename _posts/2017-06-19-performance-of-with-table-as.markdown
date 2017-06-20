---
layout: post
title:  "Performance Hit of Using Temporary Tables"
date:   2017-06-19 16:00:00 -0700
categories: Query Optimization
---

**tl;dr:** Be very careful when using temp tables, especially when filtering against results from them. Performance using temp tables seems very variable and can be up to 100x slower. If possible, compare your queries using temp tables with hardcoded queries that don't use them.

### Filtering on temporary tables performance

Pop quiz: We have a view `fact.user_tracking_view` and with a sort key of `time_occurred_pst`. Below are three queries that return how many events happened yesterday. Can you guess the relative performance of each?

{% highlight sql %}
-- First Query
with today as (select (current_timestamp at time zone 'PST8PDT')::date::timestamp as timestamp)
select count(*) from fact.user_tracking_view 
where time_occurred_pst > ((select timestamp from today) - interval '2 day')
and time_occurred_pst < ((select timestamp from today) - interval '1 day');

-- Second Query
with yesterday as (select (current_timestamp at time zone 'PST8PDT'::date - interval '2 day') as start_t, (current_timestamp at time zone 'PST8PDT'::date - interval '1 day') as end_t)
select count(*) from fact.user_tracking_view
where time_occurred_pst > (select start_t from yesterday)
and time_occurred_pst <  (select end_t from yesterday);

-- Third Query
select count(*) from fact.user_tracking_view 
where time_occurred_pst > (current_timestamp at time zone 'PST8PDT'::date - interval '2 day')
and time_occurred_pst < (current_timestamp at time zone 'PST8PDT'::date - interval '1 day');
{% endhighlight %}

These queries all look pretty similuar; however, the **first query is actually ~100x slower**. For my Redshift table, the first finishes in around 20 seconds, whereas the second and third takes around 200 milliseconds. **The performance hit is HUUUUUUUUUUUUUUUUUUUUUUUUGGGGGGGGGGGGGGGEEEEEEEEEEEEEEE.**

### What's the deal?

I'm not really sure. Doing an explain query on the three, the first two queries have actually very similar explain paths. You might think the filter condition `where time_occurred_pst > ((select timestamp from today) - interval '2 day')` violates the per row conversion rule I talk about [here]({% post_url 2017-04-26-timestamp-conversions-in-looker %}), but that doesn't seem to be the case as the third query is also quite fast.

So, the lesson is be careful when using temporary tables. **Try to avoid performing operations on values from temporary tables, and try to benchmark queries using temporary tables against queries using hard coded values.**

