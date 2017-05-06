---
layout: post
title:  "Timestamp Conversions in Looker"
date:   2017-04-26 16:00:00 -0700
categories: SQL Looker
---

**tl;dr:** If you're using Looker and going to be filtering by timestamp/datetimes, definitely use timestamp instead of timestamptz. If you're going to group by timestamp (ie, group by DATE(timestamp)), then it's highly recommended the timestamp is stored in the same timezone as the user querying the data. To make sure there is no confusion, add an extra timestamptz column with the time as the source of time truth.


### Timestamp Basics


If you're like most developers I know, chances are you kind of know how timestamp/timestamptz in postgres works but not really. There are already some really good [blog posts][timestamp-blog] on that subject. I did want to address one particular piece of advice: [always use timestamptz][always-use-timestamp-tz]. While I agree with it in general, I've found it's bad advice if [Looker][looker] is your BI tool.


To explain, first a pop quiz: Assuming you have two identical tables, except one has an indexed [(1)](#note) timestamp column and one has an indexed timestamptz column. Which of the following queries will be faster? 

{% highlight sql %}
SELECT count(*) 
FROM table_1 
WHERE timestamp > '2017-04-01'::timestamp;
{% endhighlight %}

or 
{% highlight sql %}
SELECT count(*) 
FROM table_2 
WHERE timestamptz > '2017-04-01'::timestamp;
{% endhighlight %}

Run against our view of 1.3 billion rows with 145 million rows that fit the where clause, the first query takes **~0.5 seconds** whereas the second query takes **~16 seconds**. The reason is because the index will be used in the first query, whereas in the second it won't be because each timestamptz has to be converted to a timestamp first, leading to a sequential scan.


### Great, what does that have to do with Looker?


**Looker has no concept of a timestamptz column.** Amazing, I know! We were so surprised we even opened a ticket to confirm, and that's what they said. Basically, all time related filter queries are ran against timestamp (and not timestamptz). And as we've already shown, filtering by timestamp when your column is timestamptz leads to a slow query. Therefore, **only expose timestamp columns in Looker for filtering.**


### Thanks for that. Any other tips?


You bet! Looker has a [feature][looker-tz] that automatically converts timestamps in a table to another timestamp in a different timezone. This _usually_ works fine, as the conversion _usually_ only happens once per query instead for every row. IE, you get

{% highlight sql %}
--This is good: only 1 CONVERT_TIMEZONE done per query.
SELECT count(*)
FROM table_1 
WHERE timestamp > CONVERT_TIMEZONE('America/Los_Angeles', 'UTC', '2017-04-01'::timestamp);
{% endhighlight %}

instead of 

{% highlight sql %}
--This is bad: CONVERT_TIMEZONE done for every row.
SELECT count(*) 
FROM table_1 
WHERE CONVERT_TIMEZONE('America/Los_Angeles', 'UTC', timestamp) > '2017-04-01'::timestamp;
{% endhighlight %}

However, as my italicized _usually_ forshadows, **there is an exception to this, which is when you do group by in your timestamp.** For example, if you want to get a count of the rows by date, the Looker generated query becomes 

{% highlight sql %}
--BAD!!!! CONVERT_TIMEZONE done on every row.
SELECT
	DATE(CONVERT_TIMEZONE('America/Los_Angeles', 'UTC', timestamp)),
	COUNT(*) 
FROM table_1 
WHERE timestamp > CONVERT_TIMEZONE('America/Los_Angeles', 'UTC', 2017-04-01'::timestamp)
GROUP BY 1;
{% endhighlight %}

As we discussed earlier, this is bad. The difference between the above query and the below 

{% highlight sql %}
--Good. No CONVERT_TIMEZONE done on every row.
SELECT
	DATE(timestamp),
	COUNT(*) 
FROM table_1 
WHERE timestamp > 2017-04-01'::timestamp
GROUP BY 1;
{% endhighlight %}

was **~15 seconds** for the first and **~4 seconds** for the second. Therefore, **if you're going to do any group by for timestamps, seriously consider putting the timestamp in the same time zone as the user querying the table.**

### Won't a timestamp in an unmarked time zone lead to future errors?


Good forward planning. Yes, a timestamp column in a time zone that's not UTC will possibly lead to future errors. This is the reasoning behind the general advice [always use timestamptz][always-use-timestamp-tz]. There's no great solution for this, but what we have done is use two column: one timestamptz column as the source of truth, and one indexed timestamp_pst column with the timestamp in PST. That way, we have a canonical source of truth and a column to use in Looker for better performance.



### Note
Our company uses Redshift, so our benchmarks are run against Redshift. There are some differences between MySql/PostgreSQL and Redshift: notably, Redshift doesn't have indexes and instead sorts rows by columns. Therefore, the exact benchmarks would be different under MySql/Postgres. However, the overall idea should still apply, and so I will use the word "index" to mean "sortkey".


[timestamp-blog]: http://phili.pe/posts/timestamps-and-time-zones-in-postgresql/
[looker]: https://looker.com/
[always-use-timestamp-tz]: http://justatheory.com/computers/databases/postgresql/use-timestamptz.html
[looker-tz]: https://discourse.looker.com/t/timezone-conversion/159

