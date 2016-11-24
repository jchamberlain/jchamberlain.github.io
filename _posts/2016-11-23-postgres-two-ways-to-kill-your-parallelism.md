---
layout: post
title:  "PostgreSQL: Two Ways to Kill Your Parallelism"
date:   2016-11-23
tags: [postgresql, postgis, sql, performance]
excerpt: PostgreSQL 9.6 is fantastic, but here are two things that can throw cold water on your otherwise parallel queries.
---

PostgreSQL 9.6 has been out for almost two months now, and I'm finally getting a chance to play around with the all-new parallel sequential scans. I'm mostly interested in how it affects geospatial queries in PostGIS, and maybe I'll share some more complete thoughts on that soon.

In the meantime, here are two things I've discovered that can throw cold water on your otherwise parallel queries.

## LEFT JOIN

To be fair, `LEFT JOIN` isn't actually the problem, but I'm not entirely sure what is. (If you know, chime in!)

With a dataset of several million points, I'm getting the average value for each US state:

```sql
SELECT s.name, p.month, AVG(p.globvalue) AS inches
FROM states AS s
LEFT JOIN precipitation AS p ON ST_Intersects(s.wkb_geometry, ST_Transform(p.wkb_geometry, 4269))
GROUP BY s.ogc_fid, p.month
ORDER BY s.name ASC, p.month ASC;
```

Here's the result of `EXPLAIN`:

```
Sort  (cost=165485897.61..165485899.29 rows=672 width=51)
  Sort Key: s.name, p.month
  ->  HashAggregate  (cost=165485857.65..165485866.05 rows=672 width=51)
        Group Key: s.ogc_fid, p.month
        ->  Nested Loop Left Join  (cost=0.00..164696758.09 rows=105213275 width=25)
              Join Filter: st_intersects(s.wkb_geometry, st_transform(p.wkb_geometry, 4269))
              ->  Seq Scan on states s  (cost=0.00..4.56 rows=56 width=84212)
              ->  Materialize  (cost=0.00..220773.39 rows=5636426 width=43)
                    ->  Seq Scan on precipitation p  (cost=0.00..143052.26 rows=5636426 width=43)
```

I can trick Postgres into parallelizing this query only by setting `parallel_tuple_cost` to an absurdly low value, and even then Postgres can barely keep a second worker busy.

However, changing the `LEFT JOIN` to a `JOIN` causes Postgres to have a change of heart and allocate 8 workers (the max I've allowed)!

```
Sort  (cost=24219614.72..24219616.40 rows=672 width=51)
  Sort Key: s.name, p.month
  ->  Finalize GroupAggregate  (cost=24219507.56..24219583.16 rows=672 width=51)
        Group Key: s.ogc_fid, p.month
        ->  Sort  (cost=24219507.56..24219521.00 rows=5376 width=51)
              Sort Key: s.ogc_fid, p.month
              ->  Gather  (cost=24218628.45..24219174.45 rows=5376 width=51)
                    Workers Planned: 8
                    ->  Partial HashAggregate  (cost=24217628.45..24217636.85 rows=672 width=51)
                          Group Key: s.ogc_fid, p.month
                          ->  Nested Loop  (cost=0.00..23428528.89 rows=105213275 width=25)
                                Join Filter: st_intersects(s.wkb_geometry, st_transform(p.wkb_geometry, 4269))
                                ->  Parallel Seq Scan on precipitation p  (cost=0.00..93733.53 rows=704553 width=43)
                                ->  Seq Scan on states s  (cost=0.00..4.56 rows=56 width=84212)
```

Postgres will also choose a parallel plan if I switch the tables, using `precipitation` as the base and joining `states`, or if I do a right join instead of left.

I'll update this if I find out more, but for now, be careful when left joining a huge table to a tiny one.

## Common Table Expressions

Using the same dataset, I want to find the total normal annual rainfall within 200 miles of Dallas, Texas. Here's the query:

```sql
WITH
dallas AS (
  SELECT ST_Transform(ST_GeomFromText('POINT(-96.7970 32.7767 )', 4326), 32614) AS center
),
monthly AS (
  SELECT p.month, AVG(p.globvalue) AS inches
  FROM precipitation AS p, dallas AS d
  WHERE ST_DWithin(d.center, ST_Transform(p.wkb_geometry, 32614), 200 * 1609.34)
  GROUP BY p.month
)

SELECT SUM(inches) AS total
FROM monthly;
```

Again, by setting `parallel_tuple_cost` unrealistically low (0.001), I can get Postgres to use some parallelism for this query, but not where I want it. Take a look:

```
Aggregate  (cost=2984341.71..2984341.72 rows=1 width=32)
  CTE dallas
    ->  Result  (cost=0.00..0.01 rows=1 width=32)
  CTE monthly
    ->  HashAggregate  (cost=2984341.28..2984341.43 rows=12 width=37)
          Group Key: p.month
          ->  Nested Loop  (cost=1000.00..2974947.24 rows=1878808 width=11)
                Join Filter: st_dwithin(d.center, st_transform(p.wkb_geometry, 32614), '321868'::double precision)
                ->  CTE Scan on dallas d  (cost=0.00..0.02 rows=1 width=32)
                ->  Gather  (cost=1000.00..100369.96 rows=5636426 width=43)
                      Workers Planned: 8
                      ->  Parallel Seq Scan on precipitation p  (cost=0.00..93733.53 rows=704553 width=43)
  ->  CTE Scan on monthly  (cost=0.00..0.24 rows=12 width=32)
```

Notice how low in the plan the `Gather` occurs. As mentioned in the <a href="https://wiki.postgresql.org/wiki/Parallel_Query" target="_blank">wiki</a>, we want `Gather` to be as high up as possible. In this case we want `Gather` to happen above `Nested Loop` so that the loop itself is run in parallel. As it is, the query with parallelism takes almost twice as long as without. Bummer.

That same wiki page also mentions that "common table expressions may not appear below" `Gather`, and there's our clue. I put the center point of Dallas into a CTE both for readability and so that it only runs once (CTEs are materialized).

My CTE appears to be preventing `Gather` from moving up, and thereby foils my attempt to parallelize the loop. Let's remove it and see what happens.

```sql
WITH
monthly AS (
  SELECT p.month, AVG(p.globvalue) AS inches
  FROM precipitation AS p
  WHERE ST_DWithin(ST_Transform(ST_GeomFromText('POINT(-96.7970 32.7767 )', 4326), 32614), ST_Transform(p.wkb_geometry, 32614), 200 * 1609.34)
  GROUP BY p.month
)

SELECT SUM(inches) AS total
FROM monthly;
```

That totally works!

```
Aggregate  (cost=448198.71..448198.72 rows=1 width=32)
  CTE monthly
    ->  Finalize GroupAggregate  (cost=448197.32..448198.43 rows=12 width=37)
          Group Key: p.month
          ->  Sort  (cost=448197.32..448197.56 rows=96 width=37)
                Sort Key: p.month
                ->  Gather  (cost=448184.41..448194.16 rows=96 width=37)
                      Workers Planned: 8
                      ->  Partial HashAggregate  (cost=447184.41..447184.56 rows=12 width=37)
                            Group Key: p.month
                            ->  Parallel Seq Scan on precipitation p  (cost=0.00..446010.16 rows=234851 width=11)
                                  Filter: st_dwithin('0101000020667F000096ACDD3C3A8E2541EE21773544AF4B41'::geometry, st_transform(wkb_geometry, 32614), '321868'::double precision)
  ->  CTE Scan on monthly  (cost=0.00..0.24 rows=12 width=32)
```

8 workers planned, and it takes about 1/8 of the time. And that's with `parallel_tuple_cost` set back to the default `0.1`. So be careful with your common table expressions. They can cut both ways.