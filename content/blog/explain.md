---
author: "Louise Grandjonc"
date: 2018-03-24T16:39:21-07:00
linktitle: Understanding EXPLAIN - part 1 - Costs and actual time
title: Understanding EXPLAIN - part 1 - Costs and actual time
weight: 1
---


# Introduction

As a reminder, if you ended up on this page by looking up on bing `EXPLAIN`, first I'm impressed, you get results with bing ? But also, this article is based on a talk that I did at the [pgdayParis](http://2018.pgday.paris) and here are the [slides](https://fr.slideshare.net/LouiseGrandjonc/becoming-a-better-developer-with-explain)

If you want to test the queries, here is the [github project](https://github.com/louiseGrandjonc/owl-conference) for the talk. You can find a dump of the database, and the [SQL](https://github.com/louiseGrandjonc/owl-conference/blob/master/sql/01_generate_data.sql) using `generate_series` to fill the DB.

Let's say you have a slow query, if you never have this problem, a few advices:

- close this page, it's useless
- start working a bit, if you've never written any stupid code, what do you do ?

So, back to our slow query, what you would want is understand why it's slow, so basicaly "what the hell is postgres doing?". To do that, we are going to look into the query plan.

## What is a query plan?

You're wondering what I'm talking about? Well let's start at the begining of the story, you are sending a query to your DB server and you get a result. Like this
![Alt text](/images/explain/query.png)

What is happening there ?

- First the query is parsed into a query tree (an internal data structure). The syntax errors are detected.
- Then the planner and optimizer generates query plans (ways to execute the query) and calculate the cost of this plans.
- The best query plan is executed and then the results are returned to you.

![Alt text](/images/explain/query_path.png)

# About EXPLAIN

To use EXPLAIN, take a query and run

`EXPLAIN ANALYZE your_query;`

You will get a result that will look like this:

![Alt text](/images/explain/explain.png)

# What are costs and actual times ?


If you're using `EXPLAIN` without `ANALYZE` and the `rows` seem really of from the reality, you can force postgres to collect statistics on a table by running.

`ANALYZE my_table;`

# Conclusion