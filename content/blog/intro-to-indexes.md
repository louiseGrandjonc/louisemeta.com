---
author: "Louise Grandjonc"
date: 2018-06-21T10:20:21-07:00
linktitle: Introduction to PostgreSQL's indexes
title: Introduction to PostgreSQL's indexes
weight: 1
draft: true
---


# Introduction

When I understand what's happening precisely in my database, it helps me make smarter choices. It's why I focused on `EXPLAIN` and the algorithms used as you can read [here](/blog/explain/). Because to me it was the best way to understand why things were not working like expected.

And I believe that when it comes to indexes, it's the same. Of course you could use your ORM to add an index on a column you kind of randomly picked, but there's a smarter way...
First because ORMs create BTrees and if you're using PostgreSQL there are other types that can be very handy. Understanding their structure, how they are searched and updated might help you pick the one that fits your usecase the best.
I also hope that studying what's going on internally might raise, us developers, our awareness on the cost of having multiple (and some unused ;) ) indexes.

So in this series of article, I will cover the magical world of the following indexes types:

- BTrees
- Gin Indexes
- GiST
- SPGiST
- BRIN
- Hash

This first article will focus on BTrees and their implementation in PostgreSQL.
But first, I'd like to talk a bit more about what are indexes are used for. We talked a little bit about tunning queries, but that's not their only purpose.

# Let's talk about crocodiles

First of all, I'd like to introduce you to the data model I'm going to use for the examples.

Did you know that crocodiles love candies ? [Especially marshmallows](https://youtu.be/RujAuz28TzQ) so it's very important for them to get their teeth cleaned, and the best at it are [plover birds](http://smallscience.hbcse.tifr.res.in/crocodile-and-the-plover-bird/). If crocodiles don't do it, they're much slower at chewing things, and that's a big problem !

So they have a website to get an appointment and give their current location (obviously it's much easier for birds to fly to the crocodile than it is for crocodiles to walk to the bird). The goal is to find the closest free bird to intervene. Also birds have birds teachers. How would they learn how to clean teeth otherwise?

So here is the data model

![Alt text](/images/indexes/datamodel.png)

And here is the SQL used to create the tables.

```code

CREATE TABLE IF NOT EXISTS crocodile (
       id serial,
       email varchar(250) UNIQUE NOT NULL,
       first_name varchar(250) NOT NULL,
       last_name varchar(250) NOT NULL,
       birthday date NOT NULL,
       number_of_teeth integer NOT NULL,
       last_check_up timestamp with time zone,

       CONSTRAINT crocodile_pkey PRIMARY KEY (id),
       CONSTRAINT crocodile_email_uq UNIQUE (email)
);

CREATE TABLE IF NOT EXISTS plover_bird (
       id serial,
       first_name varchar(250) NOT NULL,
       last_name varchar(250) NOT NULL,
       teacher_id INTEGER REFERENCES plover_bird (id) ON DELETE CASCADE DEFERRABLE INITIALLY DEFERRED,

       CONSTRAINT plover_bird_pkey PRIMARY KEY (id)
);

CREATE EXTENSION btree_gist;

CREATE TABLE IF NOT EXISTS appointment (
       id serial,
       crocodile_id INTEGER NOT NULL REFERENCES crocodile (id) ON DELETE CASCADE DEFERRABLE INITIALLY DEFERRED,
       plover_bird_id INTEGER REFERENCES plover_bird (id) ON DELETE CASCADE DEFERRABLE INITIALLY DEFERRED,
       emergency_level integer NOT NULL CHECK (emergency_level > 0 AND emergency_level < 11),
       schedule tstzrange NOT NULL,
       done BOOLEAN DEFAULT FALSE,
       area GEOMETRY(Point, 4326),

       EXCLUDE USING GIST (crocodile_id WITH =, schedule WITH &&),
       CONSTRAINT appointment_pkey PRIMARY KEY (id)
);
```

# What's are indexes for


Indexes have two purposes: constraints and query optimization.


## Constraints

When you're working on your data modeling, you might have noticed that some constraints transform into indexes. There are three cases where an index is automatically created to insure the consistency of data when you add a constraint:

- PRIMARY KEY
- UNIQUE
- EXCLUDE USING

So for example, my crocodile table has the following indexes when I create it:

```code
"crocodile_pkey" PRIMARY KEY, btree (id)
"crocodile_email_uq" UNIQUE CONSTRAINT, btree (email)
```

And same goes with the appointment table

```code
Indexes:
    "appointment_pkey" PRIMARY KEY, btree (id)
    "appointment_crocodile_id_schedule_excl" EXCLUDE USING gist (crocodile_id WITH =, schedule WITH &&)
```

Postgres created these indexes for me and they're ready to ensure consistency.
In my example I used a GiST index for the `EXCLUDE`. I'll get back to it in the article on GiST. But what you can see is that the default index for constraint is a BTree.
And more generally, BTree is the default index when you use `CREATE INDEX`.

So in conclusion, indexes can be used as a datamodeling strategy. If you one day have write latencies and trace it back to having too many indexes on a table, maybe these constraints indexes should not be the first one to get rid of...

If you want to know more about constraints, I really recommend watching [Will Leinweber](https://bitfission.com/)'s [talk](https://www.youtube.com/watch?v=hWh8QoV8z8k&feature=youtu.be) on the subject, and goes more into details of unique partial indexes.

## Query optimization

The second use for indexes is to tune your queries. As I'm going to talk a lot about that in the following articles, I'm not going to bore you right now with it.

But that's for that usecase that you might need to understand better the indexes internal data structure. It will help you choose an indexing strategy.

There's a lot to think about when you decide to add an index.
Indeed the index that you need will depend of the operations that you're doing on the columns, the type of data, 

-> choose the queries you need to optimize, remember there is a cost to maintaining indexes on create/update/deletes.
Indexes are redundant, they repeat data from your table and have to be consistent with the data in the rows. Which mean that on commit of a data change (on inserts, delete or update), it will need to be updated.
For each index type, I will explain the algorithm used to maintain the data in the index, and I hope that it will help understand why there's for some indexes more risk to slower your data manipulation queries.
It's important to think about the strategy that you choose when you decide to add an index.

So wether it's for constraint indexes, postgres uses Btree as the default, postgres has its own implementation of BTrees.

But other types of indexes that can fit more, if you're a developer handling your indexes with an ORM chances that you won't use them. Which is a shame
