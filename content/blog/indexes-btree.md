---
author: "Louise Grandjonc"
date: 2018-06-22T10:20:21-07:00
linktitle: PostgreSQL's indexes - BTrees
title: PostgreSQL's indexes - BTrees
weight: 1
draft: true
---

# Introduction

Index used by default on `CREATE INDEX`. As I said in the [introduction to indexes article](/blog/intro-to-indexes), BTrees are also created automatically for primary key and unique constraints.

Before I start going into the internal structure of BTree, I'd like to talk to you about a [tiny python library](https://github.com/louiseGrandjonc/pageinspect_inspector) to have a graphical representation of what's going on inside PostgreSQL's BTrees.
You pass your index to a command line and it generates a HTML allowing you to explore pages and items for your index.
I wrote it using the extension [pageinspect](https://www.postgresql.org/docs/10/static/pageinspect.html). All images in this article (except the crocodiles ;) ) come from it.

Here is a little video to show you how it works (where you *might* notice that I'm not a great frontend developer).

{{< youtube YbdMG3wIGEc >}}


# What is a BTree ?

BTree stands for balanced tree. The root and each parent node can have more than one child which minimize the depth of the tree and all the leaves at equal distance from the root. BTrees have O(log(n)) complexity for inserts, deletes and searches. But we will go into details of this algorithm in just a bit.


ADD DRAWING OF BTREE

Postgres implements the Lehman & Yao Btree. We will go into the specitifities of this algorithm and the Postgres implementation in the next part.

# BTrees in postgres


Let's talk again about crocos. So as we said, crocodiles eat candies like 10yo left alone with money for dinner, and their number of teeth changes over life. It grows and then when they get old, they fall. Like humans. (Honestly, this is probably not 100% accurate, if you're a vet, sorry about that, also, what are you doing here?).
So, we want to be able to filter on the number of teeth and run queries like:

```code
SELECT email FROM crocodile WHERE number_of_teeth < 18;
```

For that, we decided to create the following index.

```code
CREATE INDEX ON crocodile (number_of_teeth);
```

##  Metapage

Using my `pageinspect` inspector, if we generate the html for this index we would see

![Alt text](/images/indexes/metapage_root.png)

Here we see the metapage of the index and the root. The metapage is always the first page. It contains:

- The block number of the root
- The level of the root
- A block number for the fast root
- The level of the fast root

Keeping a pointer to the root is usefull because the root can change when after inserting the root page has to be split. In which case the root can change. Pages splits will be detailed in the insert algorithm part of this article.

So your metapage contains pointers to the root which can be represented this way.


But it you prefer using `pageinspect` here is the query returning information on the metapage.

```code
croco=# SELECT * FROM bt_metap('crocodile_number_of_teeth_idx');
 magic  | version | root | level | fastroot | fastlevel
--------+---------+------+-------+----------+-----------
 340322 |       2 |  290 |     2 |      290 |         2
(1 row)
```

You may be confused by this fast root thing that here is redundant with the root, but let's keep some of the suspense for the deletion algorithm part...

## Pages and items

### Pages

The root, the parents and the leaves are all pages with the same structure. Pages have:

- A block number, here the root block number is 290
- A pointer to the next and previous pages
- A high key
- Items

![Alt text](/images/indexes/root_pointers.png)

So pages are linked list, this is a specificity of the Yao and Lehmann BTree. This is for example very usefull in the leaves for `ORDER BY` queries. 
Here would be the pages on the level 0 (leaves)

![Alt text](/images/indexes/leaves_only.png)

If we wanted to use an `ORDER BY number_of_teeth ASC`, we could start from the first leaf page and thanks to the next page pointer, have directly all rows in the right order.

The high key is also a specificity of Lehman & Yao BTrees. Any item in the page will have a value inferior or equal to the high key.

`pageinspect` allows you to have more information on BTree pages

```code
croco_test=# SELECT * FROM bt_page_stats('crocodile_number_of_teeth_idx', 289);
-[ RECORD 1 ]-+-----
blkno         | 289
type          | i
live_items    | 285
dead_items    | 0
avg_item_size | 15
page_size     | 8192
free_size     | 2456
btpo_prev     | 3
btpo_next     | 575
btpo          | 1
btpo_flags    | 0
```


### Items

![Alt text](/images/indexes/parent_items.png)

The items of a page (that are in green in my drawings) have a value and a ctid.
If you use the `pageinspect` extension, you will see that the first item of a page defines the high key like following:

```code
croco_test=# SELECT * FROM bt_page_items('crocodile_number_of_teeth_idx', 12);
-[ RECORD 1 ]-----------------------
itemoffset | 1
ctid       | (163,87)
itemlen    | 16
nulls      | f
vars       | f
data       | 02 00 00 00 00 00 00 00
-[ RECORD 2 ]-----------------------
itemoffset | 2
ctid       | (49,46)
itemlen    | 16
nulls      | f
vars       | f
data       | 02 00 00 00 00 00 00 00
-[ RECORD 3 ]-----------------------
itemoffset | 3
ctid       | (49,52)
itemlen    | 16
nulls      | f
vars       | f
data       | 02 00 00 00 00 00 00 00
```


The difference between the parents/root level and leaves is that the ctid of a parent level page would point to a child page. In the leaves, the ctid is the one from the row.
The value of an item in a leaf page is the value of the column(s) of the row.
The value of an item in a parent page is the value of the first item in the child page.

Here is what leaf level items look like.

![Alt text](/images/indexes/leaves_items.png)



# Searching in a BTree

Explaining the algorithm, and so, why it fits best for certain operations
Case with mutliple rows with same value

Example of looking for a row
Example of order by


# Inserting in a BTREE
Two levels order, the rows need to be ordered at the leaf level and at the parent level

# Deleting in a BTree

- Fast root
- Pages deleted only when no node and after VACCUUM
- What happens after delete from large set of data