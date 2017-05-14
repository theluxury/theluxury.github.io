---
layout: post
title:  "Joda-Time Daylight Savings Hour"
date:   2017-05-17 16:00:00 -0700
categories: Joda-Time Timezone
---

**tl;dr:** Be very careful when parsing DateTimeFormat with PST or PDT with `joda.time.DateTime`. **`joda.time.DateTime` does not know the difference between PST/PDT when parsing a string.** In you can, use 'ZZ' (-08:00) instead of 'zzz' (PST). [Joda-Time Formats][joda-time-format]

### Daylight Savings Hour


Pop quiz: what's the difference between PST and PDT (or EST vs EDT)? The answer is the first stands for Pacific _Standard_ Time, and the second stands for Pacific _Daylight_ Time. More concretely, PST is UTC-08:00 (UTC time minus 8 hours), whereas PDT is UTC-07:00 (UTC minus 7 hours). 

### Okay, so PST/-08:00 are interchangeable in Joda-Time?


**NOOOOOOOOOOO!!!!!!** Consider this following piece of code.

{% highlight java %}
import org.joda.time.*

DateTimeFormatter dtf = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss.SSS zzz")
DateTime sooner = new DateTime(1478420000000L, DateTimeZone.forID("America/Los_Angeles")) 
DateTime later  = new DateTime(1478423000000L, DateTimeZone.forID("America/Los_Angeles")) 
// sooner is "2016-11-06 01:13:20.000-07:00", whereas later is "2016-11-06 01:03:20.000-08:00"
// Note that later has a lower time even though it is later since it just passed into daylight savings time.

System.out.println(later.isAfter(sooner))
{% endhighlight %}

What do you think the output is? Fortunately, it's `true`. So far, so good. However, look at what happens when you parse the string back into DateTime.

{% highlight java %}
String sooner_string = sooner.toString(dtf)
String later_string  = later.toString(dtf)
// sooner_string = "2016-11-06 01:13:20.000 PDT"
// later_string  = "2016-11-06 01:03:20.000 PST"


DateTime sooner_parsed = DateTime.parse(sooner_string, dtf)
DateTime later_parsed  = DateTime.parse(later_string, dtf)  

System.out.println(later_parsed.isAfter(sooner_parsed))
{% endhighlight %}

**Surprisingly, the output is `false`.** It appears that even though Joda-Time recognizes the difference between PDT/PST when writing to string, it doesn't recognize the difference when parsing the string back into DateTime. This is confirmed by

{% highlight java %}
val pdtMillis = DateTime.parse("2016-11-06 01:03:20.000 PDT", dtf).getMillis
val pstMillis = DateTime.parse("2016-11-06 01:03:20.000 PST", dtf).getMillis

System.out.println(pdtMillis)
System.out.println(pstMillis)
{% endhighlight %}

The output for both lines will be `1478430000000L`. **When parsing timezones, DateTime is indifferent to PST/PDT, even though those represent different offsets!** I haven't tried this with out Daylights Savings Times like EST/EDT, but I imagine the effect is the same.


### So does that mean all my stringified PST times are invalid when parsed?

Not necessarily. So while `joda.time.DateTime` treats PST/PDT the same when parsing, it really shouldn't affect any time except the one hour when it's possible to overlap: it's smart enough to know what you mean except for the one hour in the fall when daylight savings can [overlap][stack-overflow-dst]. However, if you have records that require/record precise timing, it may cause bugs during that hour.

### So what should I do?

While Joda-Time seems to parse PDT/PST identically, it does not do the same for the underlying offsets -08:00/-07:00. So, my advice is to use numerical offsets, and only use human readable timezones when you have to show it to users.

### Couldn't all of this complexity been avoided by just using DateTime object/unix time?

Yes, excellent point: why even go through all this hassle of serializing DateTime to string and back? In my situation, I had to to write DateTime to s3, so I had to serialize the object, and human readable string seemed better to me than byte array. In addition, I wanted time zone information, so I didn't want to use unix time. But in general, unnecessary serialization/confusing offsets should be avoided, precisely because it's a source of bugs such as this. 

[stack-overflow-dst]: https://stackoverflow.com/tags/dst/info
[joda-time-format]: http://www.joda.org/joda-time/apidocs/org/joda/time/format/DateTimeFormat.html




