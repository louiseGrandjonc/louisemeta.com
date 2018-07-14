---
author: "Louise Grandjonc"
date: 2018-06-22T10:20:21-07:00
linktitle: PostgreSQL's indexes - BTrees
title: PostgreSQL's indexes - BTrees
weight: 1
draft: true
---

# Introduction

BTree is the index type used by default on `CREATE INDEX`. As I said in the [introduction to indexes article](/blog/intro-to-indexes), BTrees are also created automatically for primary key and unique constraints.

Before I start going into the internal structure of BTree, I'd like to talk to you about a [tiny python library](https://github.com/louiseGrandjonc/pageinspect_inspector) to have a graphical representation of what's going on inside PostgreSQL's BTrees.
You pass your index to a command line and it generates a HTML allowing you to explore pages and items of your index.
I wrote it using the extension [pageinspect](https://www.postgresql.org/docs/10/static/pageinspect.html). All images in this article (except the crocodiles ;) ) come from it.

Here is a little video to show you how it works (where you *might* notice that I'm not a great frontend developer).

{{< youtube YbdMG3wIGEc >}}


# What is a BTree ?

BTree stands for balanced tree. The root and each parent node can have more than one child which minimize the depth of the tree and all the leaves at equal distance from the root. BTrees have O(log(n)) complexity for inserts, deletes and searches. But we will go into details of this algorithm in just a bit.


ADD DRAWING OF BTREE


# BTrees in postgres

Postgres implements the Lehman & Yao Btree which has a few specificities that you will discover in this article, or if you're **really** into it, go read their [research paper](https://www.csd.uoc.gr/~hy460/pdf/p650-lehman.pdf). I did and it was a lot of fun (I guess).

Let's talk again about crocos. So as we said in the [previous article](/blog/intro-to-indexes), crocodiles eat candies like 10yo left alone with money for dinner, and their number of teeth changes over life. It grows and then when they get old, they fall. Like humans. (Honestly, this is probably not 100% accurate, if you're a vet, sorry about that, also, what are you doing here?).
So, we want to be able to filter on the number of teeth and run queries like:

```code
SELECT email FROM crocodile WHERE number_of_teeth < 18;
```

For that, we decided to create the following index.

```code
CREATE INDEX ON crocodile (number_of_teeth);
```

##  Metapage

I generated the html for this index using `pageinspect_inspector` and here a first screenshot.

![Alt text](/images/indexes/metapage_root.png)

Here we see the metapage of the index and the root. The metapage is always the first page of a posgres index. It contains:

- The block number of the root
- The level of the root
- A block number for the fast root
- The level of the fast root

The pointer to the root can change. It's a bit early to go into the details, but after an insert, it can be necessary to split the current root into two pages, in which case a new root is created with pointers on the splitted pages. Pages splits will be detailed in [the insert algorithm part of this article](/blog/indexes-btree#inserting-in-a-btree).


If you prefer using `pageinspect` here is the query returning information on the metapage.

```code
croco=# SELECT * FROM bt_metap('crocodile_number_of_teeth_idx');
 magic  | version | root | level | fastroot | fastlevel
--------+---------+------+-------+----------+-----------
 340322 |       2 |  290 |     2 |      290 |         2
(1 row)
```

You may be confused by this fast root thing that here is redundant with the root, but let's keep some of the suspense for [the deletion algorithm part...](/blog/indexes-btree#deleting-from-a-btree)

## Pages and items

### Pages

The root, the parents and the leaves are all pages with the same structure. Pages have:

- A block number, here the root block number is 290
- A pointer to the next and previous pages
- A high key
- Items

![Alt text](/images/indexes/root_pointers.png)

**Purpose of the next and previous pages pointers**

Thanks to the pointers to next and previous pages, pages in the same level are a linked list. This is a specificity of the Yao and Lehmann BTree. For example it's very usefull in the leaves for `ORDER BY` queries. 
Here would be the pages on the level 0 (leaves)

![Alt text](/images/indexes/leaves_only.png)

If we wanted to run the query

`SELECT email FROM crocodile ORDER BY number_of_teeth ASC`

Postgres would start at the first leaf page and thanks to the next page pointer, has directly all rows in the right order.


**About the high key**

The high key is also specific to Lehman & Yao BTrees. Any item in the page will have a value inferior or equal to the high key.

The root doesn't have a high key, but for example in a parent level of my crocodile's teeth index, like this one

![Alt text](/images/indexes/parent.png)

I can see that in the page 3, I will find crocodiles with 3 or less teeth, in page 289, with 31 and less, and in page 575, there is no high key as it's the rightmost page.


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

The items of a page (that are in green in my drawings) have a value and a ctid.

![Alt text](/images/indexes/parent_items.png)

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


The difference between the parents/root level and leaves is that the ctid of a parent level page would point to a child page.
In the leaves, the ctid is the one from the row.

The value of an item in a leaf page is the value of the column(s) of the row. 
The value of an item in a parent page is the value of the first item in the child page.

Here is what leaf level items look like.

![Alt text](/images/indexes/leaves_items.png)


## To sum it up

- A Btree is a balanced tree. PostgreSQL implements the Lehmann & Yao algorithm.
- A BTree starts with a metapage containing information on the root and fast root block numbers and levels.
- Root, parent and leaves are pages. Each level is a linked list making it easier to move from one page to an other within the same level.
- Pages have a high key defining the biggest value that can be found and inserted in the page.
- A page has items. In the root or the parent levels, the items point to a page on the child level. In the leaves, it points to the heap of the row in the table.

Sometimes an image is worth a thousand words, so don't hesitate to try [pageinspect_inspector](https://github.com/louiseGrandjonc/pageinspect_inspector) to visualize one of your own BTree indexes.

# Searching in a BTree

In order to search in a BTree, postgres first uses the query scan to define scan keys. If possible, redundants keys in your query are eliminated to keep only the tightest bounds. If you did something real smart like

```code
SELECT email, number_of teeth FROM crocodile WHERE number_of_teeth > 4
AND number_of_teeth > 5 ORDER BY number_of_teeth ASC;
```

 The tightest bound is `number_of_teeth > 5`, so the result of the query would be

```code
                 email                  | number_of_teeth
----------------------------------------+-----------------
 anne.chow222131@croco.com              |               6
 valentin.williams222154@croco.com      |               6
 pauline.lal222156@croco.com            |               6
 han.yadav232276@croco.com              |               6
```

The scankeys are passed to the `_bt_search` function. Its purpose is to find the first leaf page where the scankey can be found. A boolean passed to the function defines if we are looking for the first item with the value >= scankey or strictly > scankey.
To do that, it's going to descend the tree.

The algorithm is the following:

- A page pointer (`bufP`) is initalized with pointer to root
- While the page is not a leaf
  - expliquer le move right
  - 





## Searching in a multicolumn index


Creation de Scankey qui peut n'avoir que les premières colonnes de l'index, pas obligatoire d'avoir toutes les colonnes de l'index

bt_binsrch() -> va utiliser scankey pour trouver le premier item d'une page ayant cette key
Dans une page leaf -> retourne l'offset de la première occurence >= à la clé (si nextkey=False, sinon >)

si scankey > à toutes les clés de la page, retourne nb keys de la page + 1

Dans un niveau parent, bt_binsrch retourne l'offset du dernier item dont la valeur < item.value si nextkey est false, sinon dernier item ayant value <= scankey


bt_binsrch appelle bt_compare. A priori pour mettre en place le binary search et trouver la mid key




The algorithm used to search in a BTree

If count or only columns from index, index only scan. Otherwise retrieve rows from table using the pointers

Explaining the algorithm, and so, why it fits best for certain operations
Multiple pages can have the same high key, difference with Lehmann & Yao algorithm

Insertion
scankeys are built within the btree code (eg, by _bt_mkscankey()) and are
used to locate the starting point of a scan, as well as for locating the
place to insert a new index tuple.  (Note: in the case of an insertion
scankey built from a search scankey, there might be fewer keys than
index columns, indicating that we have no constraints for the remaining
index columns.)  After we have located the starting point of a scan, the
original search scankey is consulted as each index entry is sequentially
scanned to decide whether to return the entry and whether the scan can
stop (see _bt_checkkeys()).

Explain locks


# Inserting in a BTREE
Finding the page where we can insert
Page splits
Two levels order, the rows need to be ordered at the leaf level and at the parent level

# Deleting from a BTree

- Fast root
- Pages deleted only when no node and after VACCUUM
- What happens after delete from large set of data