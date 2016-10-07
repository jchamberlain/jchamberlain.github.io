---
layout: post
title:  Performance of DATE vs. TIMESTAMP in PostgreSQL
date:   2016-10-06
tags: [postgresql, datetime, performance]
excerpt: This is my 100% non-scientific attempt to find a significant performance advantage in DATE.
---

After writing about [how to handle time in various databases](/2016/three-rules-for-dates-and-times/), I got to thinking more about my recommendation to use `TIMESTAMP WITH TIME ZONE` over `DATE` in Postgres. `TIMESTAMP` is stored in 8 bytes, while `DATE` takes only 4. Perhaps in large datasets the smaller disk usage of `DATE` could give it a significant performance advantage? This is my 100% non-scientific attempt to get a feeling for whether or not that advantage exists.

## Generate some dates
Let's make a million timestamps and a million dates along with some fake data:
```sql
CREATE TABLE timestamps AS
SELECT to_timestamp(floor(1420099200 + random() * 86400 * 365)) AS timestamp, floor(random() * 10000) AS value
FROM generate_series(1,1000000);

CREATE TABLE dates AS
SELECT to_timestamp(floor(1420099200 + random() * 86400 * 365))::date AS date, floor(random() * 10000) AS value
FROM generate_series(1,1000000);
```

The dates will be spread throughout 2015, and the values will be between 0 and 10,000.

## Clock the queries
Let's get the unique months in each set:
```sql
SELECT DISTINCT extract(month from timestamp)
FROM timestamps;
-- Time: 369.861 ms

SELECT DISTINCT extract(month from date)
FROM dates;
-- Time: 343.743 ms
```

Sum the values in the first six months:
```sql
SELECT SUM(value)
FROM timestamps
WHERE timestamp BETWEEN '2015-01-01' AND '2015-06-30';
-- Time: 178.716 ms

SELECT SUM(value)
FROM dates
WHERE date BETWEEN '2015-01-01' AND '2015-06-30';
-- Time: 171.187 ms
```

Sum all values grouped by month:
```sql
SELECT extract(month from timestamp) AS month, SUM(value)
FROM timestamps
GROUP BY month;
-- Time: 1708.357 ms

SELECT extract(month from date) AS month, SUM(value)
FROM dates
GROUP BY month;
-- Time: 465.671 ms
```

Whoa, did I read that last one right? Yes, those are the numbers I'm getting pretty consistently on my local (virtual) box.

Mostly it's a close call, but that huge difference in the last test seems odd to me.

## Is it the storage, or the timezone?
Maybe there's some extra processing in addition to the extra size of timestamps. Let's remove timezones from the equation while keeping the timestamps by creating and querying another table:
```sql
CREATE TABLE timestamps_2 AS
SELECT to_timestamp(floor(1420099200 + random() * 86400 * 365))::timestamp AS timestamp, floor(random() * 10000) AS value
FROM generate_series(1,1000000);

SELECT extract(month from timestamp) AS month, SUM(value)
FROM timestamps_2
GROUP BY month;
-- Time: 1593.808 ms
```

Ok, so timezones do slow things down a little bit, but the bulk of `DATE`'s advantage is something else, presumably its smaller size. Again, though I'm not doing lots of samples and rounding, the run times I'm seeing are very consistent.

## So should I use `DATE`?
As they say, YMMV, but perhaps in larger datasets where you really don't need the time component, converting to UTC and storing as `DATE` may be the best approach. Just be sure to also convert any timestamps when comparing against those dates, or you'll run into ambiguity. And unless you know `TIMESTAMP` is going to be a bottleneck, just keep using it. As always, benchmark first.

