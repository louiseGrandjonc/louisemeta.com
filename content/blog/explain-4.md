---
author: "Louise Grandjonc"
date: 2018-03-24T14:39:21-07:00
linktitle: Understanding EXPLAIN - part 4 - Ordering and a word on offset
title: Understanding EXPLAIN - part 4 - Ordering and a word on offset
weight: 1
draft: true
---


## Quicksort

```code
owl_conference=# EXPLAIN ANALYZE SELECT * FROM letters ORDER BY delivered_by DESC;
                                                      QUERY PLAN
----------------------------------------------------------------------------------------------------------------------
 Sort  (cost=54680.83..55715.48 rows=413861 width=24) (actual time=455.722..669.053 rows=413861 loops=1)
   Sort Key: delivered_by DESC
   Sort Method: external merge  Disk: 15304kB
   ->  Seq Scan on letters  (cost=0.00..7582.61 rows=413861 width=24) (actual time=0.030..69.871 rows=413861 loops=1)
 Planning time: 0.063 ms
 Execution time: 706.017 ms
(6 rows)
```

Let's add an index on the order by column.

`CREATE INDEX ON letters (delivered_by);`

```code
owl_conference=# EXPLAIN ANALYZE SELECT * FROM letters ORDER BY delivered_by DESC LIMIT 10;
                                                                         QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.42..1.02 rows=10 width=24) (actual time=0.141..0.150 rows=10 loops=1)
   ->  Index Scan Backward using letters_delivered_by_idx on letters  (cost=0.42..24532.28 rows=413861 width=24) (actual time=0.140..0.147 rows=10 loops=1)
 Planning time: 0.423 ms
 Execution time: 0.193 ms
(4 rows)
```

## Top-N heap sort

```code
owl_conference=# EXPLAIN ANALYZE SELECT * FROM letters ORDER BY delivered_by DESC LIMIT 10;
                                                         QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=16526.00..16526.02 rows=10 width=24) (actual time=122.296..122.299 rows=10 loops=1)
   ->  Sort  (cost=16526.00..17560.65 rows=413861 width=24) (actual time=122.295..122.296 rows=10 loops=1)
         Sort Key: delivered_by DESC
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Seq Scan on letters  (cost=0.00..7582.61 rows=413861 width=24) (actual time=0.023..58.215 rows=413861 loops=1)
 Planning time: 0.083 ms
 Execution time: 122.392 ms
(7 rows)
```


## A word on offset


Ordering is often used in order to paginate results. So it's a good time to talk about `OFFSET`.

Let's look at the `EXPLAIN` of retrieving the first 10 rows out of my letters table.

```code
owl_conference=# EXPLAIN ANALYZE SELECT * FROM letters ORDER BY delivered_by DESC OFFSET 0 LIMIT 10;
                                                                         QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.42..1.02 rows=10 width=24) (actual time=0.028..0.067 rows=10 loops=1)
   ->  Index Scan Backward using letters_delivered_by_idx on letters  (cost=0.42..24532.28 rows=413861 width=24) (actual time=0.027..0.064 rows=10 loops=1)
 Planning time: 0.105 ms
 Execution time: 0.115 ms
(4 rows)
```

Of nice, the execution time is under a millisecond, it's using my index. `Offset` is awesome no?

Okay, now let's try to retrieve rows at the page 40001, so with an offset of `400000 * 10 = 400000`.

```code
owl_conference=# EXPLAIN ANALYZE SELECT * FROM letters ORDER BY delivered_by DESC OFFSET 400000 LIMIT 10;
                                                                            QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=23710.66..23711.25 rows=10 width=24) (actual time=881.710..881.718 rows=10 loops=1)
   ->  Index Scan Backward using letters_delivered_by_idx on letters  (cost=0.42..24532.28 rows=413861 width=24) (actual time=0.013..854.924 rows=400010 loops=1)
 Planning time: 0.106 ms
 Execution time: 654.810 ms
(4 rows)
```

Now it takes 654ms, still using the index... But what you can see is `rows=400010`. You only wanted 10 rows, so why is it fetching the 400 000 first ?
`OFFSET` is used to skip the first N rows of a query. 

offset instructs the databases skip the first N results of a query. However, the database must still fetch these rows from the disk and bring them in order before it can send the following ones.

The good news is that you can live without `OFFSET`, here is a [wonderful article](https://use-the-index-luke.com/no-offset) on the subject.