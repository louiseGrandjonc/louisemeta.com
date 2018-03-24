---
author: "Louise Grandjonc"
date: 2018-03-23
linktitle: Understanding query's EXPLAIN
title: Understanding query's EXPLAIN
weight: 1
---


# Introduction


As a developer I've written a lot of stupid code. And I think that if I didn't you could make the conclusion that I don't work much. But one example was when I was an intern at [Ulule](www.ulule.com). I was working on something and the CTO asked me if I checked how many queries were executed on the page, turns out there were over a hundred...
I realized that I was missing some basics :

- Like a lot of web developers I didn't think of looking into my logs to know what my ORM was doing
- Once I knew, I didn't understand why some queries were slow

After that I became a bit obsessed with performances and realized that I would have loved to see a talk on that subject when I started working with postgreSQL and Django.

So that's the story of why I submitted a talk at the [django con](https://2017.djangocon.eu/), and then at the [pgdayParis](https://2018.pgday.paris/).

The conference at pgdayParis lasting 50 minutes, I had a bit more time to get into details. I will add the video as soon as I have it. But in the meantime here is the one from the djangocon, it's more django oriented (obviously ;) ) than the one at pgday.

{{< youtube Ph2hXpTW-Zg >}}

And here are the [slides](https://fr.slideshare.net/LouiseGrandjonc/becoming-a-better-developer-with-explain)


# A word on logs

![Alt text](/images/Owls_metal_01.png)

If you're using an ORM and have never looked for your logs, let me show you how to find them.

First go in your psql:

```code
 $ psql -U user -d your_database_name
```

Then you will find the path of your log files.

```code

owl_conference=# show data_directory ;
     data_directory
-------------------------
 /usr/local/var/postgres

owl_conference=# show log_directory ;
 log_directory
---------------
 pg_log

owl_conference=# show log_filename ;
      log_filename
-------------------------
 postgresql-%Y-%m-%d.log
```

So in this case you would find your logs in

```code
/usr/local/var/postgres/pg_log/postgresql-2018-03-23.log
```

If you're a developer, working localy, in order to have all you queries into your logs you will need to change your `postgresql.conf` and put

```code
log_statement = 'all'
logging_collector = on
log_min_duration_statement = 0
```

# About EXPLAIN

To use EXPLAIN, take a query and run

`EXPLAIN ANALYZE your_query;`

You will get a result that will look like this:

![Alt text](/images/explain/explain.png)


## Sequential scan

On the image above, you can see a `seq scan`. The sequential scan simply reads the entire table and retrieves the rows matching the `where` clause.

## Index scan

If you see instead of the `seq scan` and `index scan` it means that and index is used to filter you rows. If the concept of indexes isn't quite clear, I encourage you to [watch](https://youtu.be/Ph2hXpTW-Zg?t=16m43s) my conference where I explain it.

## Bitmap heap scan

## Nested loops

## Hash join

## Merge join

## Quicksort

## Top-N heap sort

## A word on offset

# Conclusion
