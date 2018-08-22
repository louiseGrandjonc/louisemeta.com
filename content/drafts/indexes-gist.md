---
author: "Louise Grandjonc"
date: 2018-06-29T10:20:21-07:00
linktitle: PostgreSQL's indexes - GIST
title: PostgreSQL's indexes - GIST
weight: 1
---

# Introduction

The GIST index is a balanced tree, but that's basically the only thing that it has in common with a BTree.

One of its particularity is that it can be extended to any type of data as long as a few functions implemented. I will mention these function in this article, and if you want to go further, you can go read the [GiST indexing Project website](http://gist.cs.berkeley.edu/)


Union: This method consolidates information in the tree. Given a set S of entries, this method returns a new key p which is true for all the data items below S. A simple way to implement Union is to return a predicate equivalent to the disjunction of the keys in S, i.e. "p1 or p2 or p3 or...".

PickSplit: As in a B-tree, pages in a GiST occasionally need to be split upon insertion of a new data item. This routine is responsible for deciding which items go to the new page, and which ones stay on the old page.


But for now we are going to focus on a few usecases, and not list all the extensions that great people have made for GiST :)

So after the last article, our birds dentists were able to look for certain syndroms and also find the most urgent appointments. All of this helping them organise their day. But what they need, is to do a schedule of course ! So in order to do that, they want to find the appointments that don't have a dentist yet, and of course, they can't be healing two crocos at the time right?

So here we are, we have a problem of data overlapping. And so we create the following index

```code
CREATE INDEX ON appointment USING GIST(schedule)
```

And, oh also, you know what would be perfect? If the dentists could find the appointments near their previous one. So that they can optimise their route and not be exhausted by the all flying.

So now we add some "nearest point" problem to that...

But the great coincidence that GiST works for these operators right ?


And of course, let's first talk about the internal data structure


## A quick word about the gevel extension

TODO

# GiST structure

So, as I said, the GiST index is also a balanced tree. If you have a doubt on what an balanced tree is, you can read [the article on BTree](/blog/indexes-btree).

As in a BTree, the keys in a GiST are a tuple (value, pointer). The main difference with a BTree index, the data isn't ordered in the pages. And the key ranges of the pages can overlap. This means that a tuple could potentially fit in different pages.

In order to be able to visualise the data in a GiST index, I used the extension [Gevel](http://www.sai.msu.su/~megera/wiki/Gevel). 

![Alt text](/images/indexes/gist_root.png)

Here is only the root level and its children (I didn't display the items for readability). But what you can see is that, indeed, the items are not ordered like in a BTree. 

Moreover if you zoom in, you can see that, if we wanted to insert an appointment scheduled from August 14th 2014 at 7:30am to 8:30am, if could fit in both branches (circled in red) of the tree.

![Alt text](/images/indexes/gist_root_insert_multiple.png)

The rest of the structure of a GiST index is quite close to the BTree, the leaf pages have pointers to the heap tuples in the table. There is nothing quite fancy to explain except the values in the pages overlapping and unordered.

But I believe that it will become clearer why a GiST index has such odd values if we talk about the search algorithm.

# Searching in a GiST index

Let's say that we have a bird that hasn't schedule any appointment on a morning and wants to see the appointments overlapping that period as he's not super strict about the start and finish time.

Here is the result of the query

```code
croco_new=# SELECT c.email, schedule, done, emergency_level
            FROM appointment
            INNER JOIN crocodile c ON (c.id=crocodile_id)
            WHERE schedule && '[2018-05-17 08:00:00, 2018-05-17 13:00:00]'::tstzrange
            AND done IS FALSE ORDER BY emergency_level DESC LIMIT 5;
-[ RECORD 1 ]---+----------------------------------------------------
email           | louise.choudhuri238732@croco.com
schedule        | ["2018-05-17 12:03:00+02","2018-05-17 13:03:00+02")
done            | f
emergency_level | 9
-[ RECORD 2 ]---+----------------------------------------------------
email           | louise.romano225288@croco.com
schedule        | ["2018-05-17 08:30:00+02","2018-05-17 09:30:00+02")
done            | f
emergency_level | 9
-[ RECORD 3 ]---+----------------------------------------------------
email           | david.chakma246035@croco.com
schedule        | ["2018-05-17 08:40:00+02","2018-05-17 09:40:00+02")
done            | f
emergency_level | 9
-[ RECORD 4 ]---+----------------------------------------------------
email           | pablo.galli249346@croco.com
schedule        | ["2018-05-17 08:40:00+02","2018-05-17 09:40:00+02")
done            | f
emergency_level | 9
-[ RECORD 5 ]---+----------------------------------------------------
email           | harry.rizzi254582@croco.com
schedule        | ["2018-05-17 09:22:00+02","2018-05-17 10:22:00+02")
done            | f
emergency_level | 9
```

Without a GiST index, what postgres uses is an index scan and check if each row has a common point with the value `[2018-05-17 08:00:00, 2018-05-17 13:00:00]`.

This is what you see when you look into the EXPLAIN from the query.

```code
QUERY PLAN
-------------------------------------------------------------------------------
 Limit  (cost=... rows=5 width=56)
   ->  Sort  (cost=... rows=102 width=56)
         Sort Key: appointment.emergency_level DESC
         ->  Nested Loop  (cost=... rows=102 width=56)
               ->  Seq Scan on appointment
                   (cost=... rows=102 width=31)
                      Filter: ((done IS FALSE)
                      AND (schedule && '["2018-05-17 08:00:00+02",
                                         "2018-05-17 13:00:00+02"]'::tstzrange))
               ->  Index Scan using crocodile_pkey on crocodile c
                   (cost=... rows=1 width=33)
                     Index Cond: (id = appointment.crocodile_id)
(10 lignes)
```

With a GiST index, in the EXPLAIN the index is used. As the query plan is a bit long, here is the line where you can see it.


```code
->  Bitmap Heap Scan on appointment  (cost=... rows=123 width=31)
      Recheck Cond: (schedule && '["2018-05-17 08:00:00+02",
                                   "2018-05-17 13:00:00+02"]'::tstzrange)
      Filter: (done IS FALSE)
      ->  Bitmap Index Scan on appointment_schedule_idx  (cost=... rows=246 width=0)
            Index Cond: (schedule && '["2018-05-17 08:00:00+02",
                                       "2018-05-17 13:00:00+02"]'::tstzrange)
```

The search algorithm for GiST also uses scan keys. If scan keys are new to you, you can read about them in the [first article](/blog/indexes-btree) of this serie.

Once scan keys are initialised, the function [`gistgetbitmap`](https://github.com/postgres/postgres/blob/master/src/backend/access/gist/gistget.c#L630) returns a bitmap of all the heap tuple pointers (tids) matching the scan keys.

To do that the GiST tree is explored starting from the root.

The function [`GistScanPage`](https://github.com/postgres/postgres/blob/master/src/backend/access/gist/gistget.c#L324) scans the items from a page.

As in a BTree search, it checks if there was a page split caused by a concurrent insert. The difference is that, as the items are unordered and can overlap, if there was a page split, both pages can contain tuples matchin the scan key. So, the right sibling is added to a queue of pages to explore. In a BTree, it simply moves to the right sibling.

Then for each item of the page, if it's not marked as deleted, we check thanks to the `gistindex_keytest` function if the item matches the scan key.

This function calls the method `Consistent` that has to be implemented for the key class that you want to index. This methods returns NO If the tree page can't contain the scan key from the query, else MAYBE.
In the cases of ranges or geometries the `Consistent` method checks if the scan key overlaps the item value. That's why GiST indexes are very efficient for overlapping, ranges containing data, or geographic data with GIS (like if I wanted to check if two geographic areas overlap)

If the item matches and the page is a leaf page, the pointer is the TID of a heap tuple in the table, so it's added to the tuples bitmap.
If the scan is ordered. The distance from the scankey if evaluated thanks to the `Distance` function, that is a key class function.

Else, if the page is not a leaf page, the item is pushed in a seach queue. The page pointed by the item will then be recursively explored.


Let's go back to our query. So if we are looking for the rows overlapping the period '[2018-05-17 08:00:00, 2018-05-17 13:00:00]', as you can see in the following figure, there are two items in the root (at least, I didn't look into ALL the root items) that could lead to leaves with matching rows. So this branches are going to be explored recursively until the leaves.

![Alt text](/images/indexes/gist_search.png)


## To sum up


# Inserting a new tuple

As I said earlier, a tuple can be inserted in several different leaf pages, it could actually be inserted anywhere in the tree but that would be very bad for the search performance.

## Step 1: finding the best page
The function `gistdoinsert` calls the function `gistchoose` which finds the best page to insert the new tuple in.

To do so, the algorithm starts from the root and recurses until a leaf page.

- First it check if there was a split. If there was one, it can't be sure it's the best page anymore and goes back to the parent page.
- For each item of the current page, it will calculate the penalty. This is a key class function. It gives a number representing how bad the value to insert would fit in the child page.


## Step 2: inserting the tuple in the page

Once the right page in found, the function [`gistinserttuples`](https://github.com/postgres/postgres/blob/master/src/backend/access/gist/gist.c#L1213) calls the function `gistplacetopage` that does the work of inserting the tuple(s) to the page and if necessary split it.

[`gistplacetopage`](https://github.com/postgres/postgres/blob/master/src/backend/access/gist/gist.c#L215) handles two cases, when it's the update of an item key, and when it's a new tuple to be inserted.

Updating the key can be necessary in two cases. The first is when there was a page split (see the [article on btree](/blog/indexes-btree) if you are not familiar with what a page split is), an item has to be inserted in the parent page, and the range of the old page probably changed, to the value of the item pointing to it has to be updated.

ADD IMAGE

The second is when the value I'm inserting changes the range of the child page, the item has to be updated. For example, let's say I insert an appointment with the schedule value [2013-11-20 05:30:00, 2013-11-20 06:30:00]. As you can see in the following images, the range of the page where it was inserted changed which led to an update of the value in the root page.

![Alt text](/images/indexes/gist_before_insert_no_split.png)

![Alt text](/images/indexes/gist_after_insert_no_split.png)


# GiST or GIN for a fulltext search

TODO