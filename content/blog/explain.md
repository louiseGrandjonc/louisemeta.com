---
author: "Louise Grandjonc"
date: 2018-03-24T16:39:21-07:00
linktitle: Understanding EXPLAIN - part 1 - Costs and actual time
title: Understanding EXPLAIN - part 1 - Costs and actual time
weight: 1
draft: true
---


# Introduction

If you didn't read the articles on [logs](/blog/developers-and-logs/) and [pg_stat_statements](/blog/pg-stat-statements/) this article is based on a talk that I did at the [pgdayParis](http://2018.pgday.paris) and here are the [slides](https://fr.slideshare.net/LouiseGrandjonc/becoming-a-better-developer-with-explain). The video should be online soon, you can watch the [short version](https://www.youtube.com/watch?v=Ph2hXpTW-Zg) from the djangoConEurope 2017.

If you want to test the queries, **here is the [github project](https://github.com/louiseGrandjonc/owl-conference)** for the talk. You can find a **dump of the database**, and the [SQL](https://github.com/louiseGrandjonc/owl-conference/blob/master/sql/01_generate_data.sql) using `generate_series` to fill the DB.

Let's say you have a slow query, if you never have this problem, a few advices:

- close this page, it's useless
- start working a bit, if you've never written any stupid code, what do you do ?

So, back to our slow query, what you would want is understand why it's slow, so basicaly "what the hell is postgres doing?". To do that, we are going to look into the query plan.

## What is a query plan?

You're wondering what I'm talking about? Well let's start at the begining of the story, you are sending a query to your DB server and you get a result. Like this
![Alt text](/images/explain/query.png)

What is happening there ?

- First the query is **parsed into a query tree** (an internal data structure). The syntax errors are detected.
- Then the planner and optimizer generates **query plans** (ways to execute the query) and **calculate the cost** of this plans.
- The best **query plan is executed** and then the results are returned to you.

![Alt text](/images/explain/query_path.png)

# About EXPLAIN

To use EXPLAIN, take a query and run

`EXPLAIN (ANALYZE) your_query;`

If you use `ANALYZE`, your query will be executed, without it you get only the query plan with the estimated costs.

If you want to use `EXPLAIN ANALYZE` on an `UPDATE` or `DELETE`, you can rollback

```code
BEGIN;
EXPLAIN ANALYZE my_query;
ROLLBACK
```

You will get a result that will look like this:

![Alt text](/images/explain/explain.png)

# What are costs and actual times ?

## Costs

The costs are calculated using database statistics. They look like this :

`(cost=0.00..205.01 rows=1 width=35)`

- The part `cost=0.00..205.01` has two numbers, the first one indicates the **cost of retrieving the first row**, and the second one, the estimated **cost of retrieving all the rows**.
  You might wonder why the first cost is 0.00 and the second one 205.01 if there is only one row. Why are the numbers not the same ? It's because it is using a sequential scan (explained in [this article](/blog/explain-2/), the cost of reading the first row of the table is 0, but the cost of retrieving the one matching the `WHERE` clause is higher. It's like with a book, reading the first line costs you nothing.
- `rows=1` is the estimated number of rows returned in your result
- `width=35` is the average size of a row in kB.

If you're using `EXPLAIN` without `ANALYZE` and the `rows` seem really of from the reality, you can force postgres to collect statistics on a table by running.

`ANALYZE my_table;`

## Actual times

The actual time is only displayed if you are using `ANALYZE`.

`(actual time=1.945..1.946 rows=1 loops=1)` means that the `seq scan` was executed once (`loops=1`), retrieved one row `rows=1` and took 1.946ms.

Be careful about the `loops`, if the scan takes 5ms but is executed 1000 times in a loop, it can be the reason of a slow query !

# Conclusion

Now it's time to understand the different scans used by postgreSQL. I mentionned the sequential scan (`seq scan`), the next [article](/blog/explain-2/) focuses on that. 
