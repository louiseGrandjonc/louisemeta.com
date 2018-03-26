---
author: "Louise Grandjonc"
date: 2018-03-24T18:39:21-07:00
linktitle: PostgreSQL's logs - a developer's best friend
title: PostgreSQL's logs - a developer's best friend
weight: 1
---


# Introduction


As a developer I've written a lot of stupid code. And I think that if I didn't you could make the conclusion that I don't work much. But one example was when I was an intern at [Ulule](www.ulule.com). I was working on something and the CTO asked me if I checked how many queries were executed on the page, turns out there were over a hundred...
I realized that I was missing some basics :

- Like a lot of web developers I didn't think of looking into my logs to know what my ORM was doing
- Once I knew, I didn't understand why some queries were slow

After that I became a bit obsessed with performances and that's the story of why I submitted a talk at the [django con](https://2017.djangocon.eu/), and then at the [pgdayParis](https://2018.pgday.paris/).

The conference at pgdayParis lasting 50 minutes, I had a bit more time to get into details. I will add the video as soon as I have it. But in the meantime here is the one from the djangocon, it's more django oriented (obviously ;) ) than the one at pgday.

{{< youtube Ph2hXpTW-Zg >}}

And here are the [slides](https://fr.slideshare.net/LouiseGrandjonc/becoming-a-better-developer-with-explain)

This first article covers why and how we, developers, should always **look into our logs** !

If you want to skip this, you can directly read about [pg_stat_statements](/blog/pg-stat-statements/) and how you can use it to find slow queries, or directly the part on [EXPLAIN](/blog/explain/).

# Why do we use ORMs
![Alt text](/images/Owls_breakdown.png)
If you are lucky enough to know DBAs, you probably have heard them tell you how ORMs are evil and anyone using them should burn in hell (hell being somewhere without beer).

So if ORMs are that bad ? Why are we using them?

- Technical debt, the project is too big to rewrite it, and even if we wanted to, often this kind of tasks are postponed because of the priority given by business to new fun features.
- The application doesn’t need any fun SQL feature so the ORM is enough
- We are a bit afraid of SQL (Something between being lazy and under a pile of tickets and not finding time to learn...)
- The object mapping, simply, it's easier to keep working in python, php, ruby and retrieve objects, isn't it ?
- The ORM handles the DB connections
- We don't have to worry about SQL injections

If you are a DBA, you are already complaining about these reasons, but well, that's unfortunatly a reality... Like Trump being president of the US, you kind of have to accept it even if it drives you crazy... But I'm being a bit harsh on ORMs, sorry. 

# Why should we be careful about them
![Alt text](/images/Owls_gnihihi.png)
So if ORMs are often used, what's wrong with them?

- The ORM **executes queries that you might not expect**
- Your queries might **not be optimised** and you won’t know about it

I'm going to give the example of the loops to show you how ugly it can get if you're not careful...

```python
owls = Owl.objects.filter(employer_name=‘Hogwarts’)
  for owl in owls:
        print(owl.job)
```

With this piece of django code you would end up in your logs with those queries:

```code
SELECT id, name, employer_name, favourite_food, job_id, feather_color FROM owl WHERE employer_name = 'Hogwarts'
SELECT id, name FROM job WHERE id = 1
SELECT id, name FROM job WHERE id = 1
SELECT id, name FROM job WHERE id = 1
SELECT id, name FROM job WHERE id = 1
SELECT id, name FROM job WHERE id = 1
SELECT id, name FROM job WHERE id = 1
SELECT id, name FROM job WHERE id = 1
SELECT id, name FROM job WHERE id = 1
…
```

So it's pretty obvious that ORMs have to be handled with a lot of **care** if you don't want to have a slow website and an angry DBA.

# So where are my logs ?

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


# Conclusion

So here we are, you have your logs and you know why it's so important to look into them ! The next step would be having something to help you find slow queries with [pg_stat_statements](/blog/pg-stat-statements/) :)