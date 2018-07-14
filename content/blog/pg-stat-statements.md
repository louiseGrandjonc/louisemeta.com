---
author: "Louise Grandjonc"
date: 2018-03-23T17:39:21-07:00
linktitle: pg_stat_statements - finding ugly queries
title: pg_stat_statements - finding ugly queries
weight: 1
---


# Introduction

As a reminder, if you ended up on this page by looking up on bing `pg_stat_statements`, first I'm impressed, you get results with bing ? But also, this article is based on a talk that I did at the [pgdayParis 2018](http://2018.pgday.paris) and here are the [slides](https://fr.slideshare.net/LouiseGrandjonc/becoming-a-better-developer-with-explain)


{{< youtube IwahVdNboc8 >}}


## What is pg_stat_statements ?

pg_stat_statement is a **postgreSQL extension**, once you enable it, it **tracks statistics on the queries** executed by a server. It will help you find slow queries.
![Alt text](/images/Owls_attack.png)

You can use it in your local environment but also on production, if you are afraid of the performances loss, here is an [article](http://pgsnaga.blogspot.fr/2011/10/performance-impact-of-pgstatstatements.html) on the subject.

# How can I activate it ?

First go in your psql:

```code
 $ psql -U owl -d owl_conference
```

And create the extension. Only superuser or database owner can create extensions. So you might need to check the privileges of the user you are connecting with.

```code
CREATE EXTENSION pg_stat_statements;
```

You then have to change your `postgresql.conf` file and restart. You can't only reload your conf file because pg_stat_statements needs to be added to the `shared_preload_libraries`. So here is the configuration.

```code
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

You can change the `pg_stat_statements.max` if you want to track more queries or are afraid of the size of the table.


# And now what?

Well, now you can simply find statistics about the queries tracked by running

```code
SELECT total_time, min_time, max_time, mean_time, calls, query
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 100;
```

You will get a set of results looking like:

```code
-[ RECORD 1 ]---------------------------------------------------------
total_time | 643.624
min_time   | 54.424
max_time   | 531.417
mean_time  | 214.541333333333
calls      | 3
query      | SELECT COUNT(*) FROM letters;
```

So from that you understand:

- The query has been executed 3 times (`calls`)
- The execution time was between 54 and 531 ms (`min_time`, `max_time`)
- The `total_time` is the sum of the time of each execution
- The `mean_time` is total_time/calls


# Conclusion

Well that's it ! But now that you know which queries are slow, you probably want to understand what's wrong. To do that, I encourage you to use EXPLAIN, if you want to read about it, I am currently writing articles on `EXPLAIN`, so be patient, they should be available in the next few days :)