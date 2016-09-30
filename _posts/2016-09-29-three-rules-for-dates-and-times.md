---
layout: post
title:  Three Rules for Dates and Times
date:   2016-09-29
tags: [datetime, timezones, mysql, postgresql, mongodb]
excerpt: If it were the New Year, these would be my resolutions.
---


The other day at GeoStrategies we found some user-defined timestamps that somehow got into our database with an extra seven hours tagged on. The number of hours was of course a dead giveaway: this was a timezone problem.

Though a quick fix, it led me to ponder the strange way in which MySQL handles dates and to clarify how I wish I had handled them myself. These three rules are where I landed. Consider using them if you aren't already–they might save you a lot of confusion. If it were the New Year, these would be my resolutions.

## The Rules
1. **Generate all dates &amp; timestamps with a UTC offset.** Your best bet is to go with a standard like ISO8601. Is your user picking a JavaScript date on a calendar? Use their current time or zero it out, but keep their local offset. `Date.prototype.toString()` is all you need. Why an offset, and not a timezone name like "PDT"? Timezone definitions are political and subject to change; UTC offsets are not.

1. **Consume dates &amp; timestamps with offsets.** Everywhere that handles temporal types must expect, understand, and respect their offsets. In your validation, don't expect a "Y-m-d" format, but require something ISO8601-like instead. Passing around a date in PHP? You can put just about any valid date string in the `DateTime()` constructor and all the work is done for you, even if your server isn't in UTC.

1. **Stores dates &amp; timestamps in UTC.** If you've done #1 and #2, this may be handled automatically by your database. If not, do this at the application layer. MySQL will try its best to thwart you either way...more on that later.

## It's all about decisions

In practice, all temporal data refers to both time and place, even simple dates. Don't fool yourself into thinking that, since you only need the date component, you can throw away the timezone. The string "2015-01-01" can fall anywhere in a 24-hour period, and at some point someone has to decide how to interpret it.

When your user wants to see rainfall numbers between January 1 and February 1, some part of your application has to decide the exact moment when that period begins, even though your user hasn't thought about it. Do you start at midnight UTC? Or whatever timezone your server is in? Or something relative to the data? **Most importantly, who gets to make this decision?**

Rule #1 means that whoever generates the date makes that decision, and that's final. Design whatever code that sets the date to set it unambiguously so no one down the line has to guess. In the example above, your client-side app can decide (or let the user decide) what timezone it wants to see the data in, and neither your REST API nor database ever need to care. This is loads better than sending "2015-01-01" and getting unpredictable data back.

Rule #2 means that consuming code (your API in the above example) never has to guess about anything. Dates and timestamps are always exactly what they claim to be, and never need reinterpreting. The decision has already been made.

Rule #3 means that all data is consistent, and may be queried and retrieved with minimal effort. Your rainfall data may come from California, but anyone can search it without knowing its source. Again, someone else decides exactly what the date is–we just need to pick the storage format.

Put these three together and dates will simply flow through your application like any other data. And you'll write less code to boot!

## How to do it
Briefly let's discuss what these rules look like using different databases, since that's what got me on the subject.

### PostgreSQL
Store your timestamps using `TIMESTAMP WITH TIME ZONE`. This makes Postgres abide by Rule #2 (it respects the timezone in the date you give it), and from there it automatically executes Rule #3 by converting the date to UTC. When you select a date without formatting, the string you get out may look a bit different and will have "+00" on the end, but it will represent the exact date you stored.

A `TIMESTAMP WITHOUT TIME ZONE` (or just `TIMESTAMP`) stores the date ambiguously, ignoring any offset in the input string. Don't use it unless you have a very special reason.

Same goes for `DATE`. It's tempting to use a smaller data type if you don't need the time component, but you'll probably need the time component more often than you might think. The data exists to be queried and, if you've followed #1 &amp; #2, you might end up with a query like:

```sql
SELECT *
FROM data
WHERE date_col >= '2015-01-02 03:00:00+09';
```

Assume `date_col` is a `DATE` containing UTC values. The whole time component of your comparison date string will be dropped, making it "2015-01-02", and thus excluding January 1st. If, however, you were comparing against "2015-01-01 13:00:00-05", your string would become "2015-01-01" and January 1st would be included. Did you know those timestamps refer to the same exact time, one in Japan, the other in the US? Postgres won't know either unless you tell it to respect time zones. Just use `TIMESTAMP WITH TIME ZONE`.

See the <a href="https://www.postgresql.org/docs/9.6/static/datatype-datetime.html" target="_blank">docs</a>.

### MySQL
MySQL makes life a little harder in that it *always* <a href="http://dev.mysql.com/doc/refman/5.7/en/datetime.html" target="_blank">ignores time zones in your date strings</a>. Your two options are `DATETIME` and `TIMESTAMP`. `DATETIME` behaves just like `TIMESTAMP WITHOUT TIME ZONE` in Postgres, while `TIMESTAMP` tries to be more...helpful. Though it still ignores UTC offsets in your dates, `TIMESTAMP` assumes you want to use the local timezone of your database connection and converts your dates accordingly.

Don't use `TIMESTAMP`. Its automatic conversion of times from system time to UTC ends up being just another source of bugs and no help at all.

Use `DATETIME`, but you'll have to convert everything to UTC in your application. Since MySQL itself doesn't follow rule #2, your app will have to be the final arbiter of dates.

See also the <a href="http://dev.mysql.com/doc/refman/5.7/en/time-zone-support.html" target="_blank">timezone docs</a>.

### MongoDB
MongoDB handles dates very similarly to MySQL. When you pass a string into `new Date()` without a time zone, it assumes local time and converts to UTC, just like MySQL's `TIMESTAMP` (so don't use it!). If you do specify a time zone, it can only be "Z" for UTC–trying to give an offset will result in an invalid date. Thus the version with a timezone is much like MySQL's `DATETIME` in that your application will need to convert dates in advance.

Thankfully, in neither scenario does it entirely ignore time zones, so you can easily tell whether you're doing it right.

See the <a href="https://docs.mongodb.com/manual/reference/method/Date/" target="_blank">docs</a>.

## Speaking the same language
Whatever you do, do it consistently, and have as few layers of decision-makers as possible. If the date's originator gets to decide the date's precise interpretation, and every consumer understands the same unambiguous language for communicating dates, you've basically eliminated datetime bugs. At least, that's what I'm hoping.

If you have additional thoughts or have encountered situations that can't be handled as I've suggested, let me know! I'd love to hear about them.