---
author: "Louise Grandjonc"
date: 2018-06-23T10:20:21-07:00
linktitle: PostgreSQL's indexes - BTrees algorithms
title: PostgreSQL's indexes - BTrees algorithms
weight: 1
---

# Introduction

In the [previous article](/blog/indexes-btree), I introduced the internal data structure of a postgres BTree index. Now it's time to talk about searching and maintaining them.

I think that understanding this can help us developers, DBAs know when using a BTree makes sense and when it wouldn't be very efficient.

# Searching in a BTree

## Initializing the scankeys

In order to search in a BTree, postgres first uses the query scan to define scankeys. If possible, redundants keys in your query are eliminated to keep only the tightest bounds. If you did something real smart like

```code
SELECT email, number_of teeth FROM crocodile
WHERE number_of_teeth > 4
      AND number_of_teeth > 5
ORDER BY number_of_teeth ASC;
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

## The BTree search function

The scankeys are passed to the `_bt_search` function.
Its purpose is to **find the first leaf page** where the **scankey could be found**.

In order to explain the `_bt_search` algorithm, I'm going to use the following query.

```code
SELECT email FROM crocodile WHERE number_of_teeth >= 20;
```

**Step 1: Initialize**
`_bt_search` **initialize a page pointer** `bufP` to the **root**.
At this step, if we take my BTree, `bufP` would be on the root with block number 290.


**Step 2: Let's climb down that tree**
The second step is a loop iterating until the `bufP` pointer is a leaf page.

*Step 2.1: should I stop looking ?*

If the `bufP` page is a leaf page, we break the loop and return a stack containing:

- the block number of the parent page of the leaf
- the block number of the leaf page
- the offset of the first item in the leaf page matching our scankey
- the stack of the parent

This way we have the path leading to the leaf page.

*Step 2.2: move right or not move right ?*

At this point the function `_bt_moveright` is called.
It decides **if it's necessary to follow the right pointer** of the current `bufP` page. This happens if, while descending on the tree, **there was an insert**.
It could then be possible that the current `bufP` page was split.

For example, the `_bt_search` for my previous query is looking for the first leaf page that could contain crocodiles with more than 20 teeth. The root doesn't have a right page, so `bufP` wouldn't change.
It would then descend on the page 289 where the high key is 31 and the minimum value is 16 like you can see here.

![Alt text](/images/indexes/root_pointers.png)

So the new `bufP` is
![Alt text](/images/indexes/bufp_1.png)

But let's say, while the parents pages of the index were being explored, an insert forced the page to be split in two and the BTree looks like that

![Alt text](/images/indexes/parents_17.png)

You can see that the high key of the page 289 went from 31 to 19 and the page has a new right sibling because it was split.
In this case, we know that the crocodiles with 20 teeth or more won't be on the branch of the page 289 anymore, and the new `bufP`is set to the direct right sibling, so page with block number 840.
The algorithm **moves right if the page high key is stricly lower to the scankey**.


*Step 2.3: looking into the `bufP` page items*

Once the possibility of moving right has been explored, we now want to look into the `bufP` page items to next children page to follow.

At this point the function `_bt_binsrch` is called which performs a binary search on the page's items.
As I'm trying to actually make this article not too long (maybe you haven't noticed ;) ) I won't explain the binary search algorithm, but you can read [the wikipedia article](https://en.wikipedia.org/wiki/Binary_search_algorithm) and [the source code](https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/nbtsearch.c#L296)

If the page is not a leaf, it returns the offset of the last item with a value stricly lower to the scankey.

Let's get back to our index and the query.

![Alt text](/images/indexes/metapage_root.png)

`_bt_binsrch` would return the offset of the item with value 16, as it's not a leaf page and it's the last item with a key inferior to the scankey (which is 20)

In a leaf page however, `_bt_binsrch` returns the offset of the first element with a value superior or equal to the scankey.

*Step 2.4: locking and moving*

Once we have the offset of the item
- the block number of the page pointed by this item is retrieved
- the stack that will be returned at the end of the search is built
- the `bufP` pointer is updated to point to the item child page
- a read lock is put on the new `bufP` page and the read lock on the parent is released

**About read locks**

I mentionned the fact that we put read locks on the currently examined page. This **read locks** ensure that the **records on that page are not motified while reading** it.
Of course, it doesn't mean that there is not a concurrent insert on a child page, which is why **we still need to possibly move to the right page**.

## To sum up

I hope that this made searching in a BTree clearer to you. I haven't talked about a few elements mostly because I don't have enough time.
First I didn't explain the case where we look for a key stricly above to the scankey, but it doesn't change the algorithm much.
And I didn't get into multicolumn indexes and the scankeys in this case that can be only on the first columns.

All of this is very interesting and I hope you'll want to explore this :)

But now let's talk about inserts !

# Inserting in a BTREE

Inserting into a BTree reuses a lot from the seach algorithm, so I will mainly focus on page splits.

In order to insert, scankeys are built.

## Finding the right insert location

Then what we want is to find on which page our new tuple (key, pointer) should be inserted. Here there are two cases.

### Index on an auto-incremented value

Very often an index is on an **auto-incremented value**, for example the primary key I used for the `crocodile` table is a serial, and of course postgreSQL created the `crocodile_pkey` on its own.

In this case a new key will always be **inserted in the right-most leaf page.**
To avoid using the search algorithm to find the right page, postgres uses a fast path.

**Step 1: retrieving the cached page**
The **right-most leaf** of the index is **cached into a buffer**. So we retrieve the page using this buffer.

** Step 2: checking the page**

There are three condictions that need to be verified in order to be able to use a fast path:

- The cached page must **still be the right-most leaf page**
- The first key on the page must be **strictly lower than the scan key**
- There must be **enough free space** on the page for the new tuple

If this conditions are met, the `fastpath` variable is set to `true`.

### Index on a "normal" value

If the value of `fastpath` is `false`, it means that the indexes BTree has to be searched in order to **find the right page for our tuple**.

**Step 1: searching for the right page**
The insert uses **`_bt_search`** to find the first page containing the key.

**Step 2: locking**
In the `_bt_search` we locked the page being red. Here we **trade the read lock for a write lock**.

**Step 3: move right ?**
It's possible that, during the lock trade, the page was split. So as in the search algorithm, we re-use the **`_bt_moveright`** function to decide if it's necessary to **change the page to its right sibling**.

## Checking constraints

If we're not allowing duplicates, make sure the key isn't already in the index.
Postgres can only detect that the value is not already in the index. So the write lock protects against concurrent writing on the page.

For example, I have a unique constraint on crocodiles email. Let's say I run the query

```code
INSERT INTO crocodile (first_name, last_name, birthday, number_of_teeth, email)
VALUES ('Louise', 'Grandjonc', '1991-12-21', 15, 'louise@croco.com');
```

The index has to be updated as the data in the index has to be consistent with the rows in the table. A write lock is aquired on the index page where the tuple (`louise@croco.com` and the pointer to the new heap) will be inserted.
If there's a concurrent writing, the second insert has to wait until the write lock is dropped. At this point the value `louise@croco.com` is already in the index.
So there is a conflict and the new tuple can't be inserted and as the operations of insert and updating the index are atomic, the query fails.


## Inserting the tuple and page splits

The function `_bt_insertonpg()` inserts a tuple on the page previously found.

### 1. Checking the tuple

The first thing that the function `_bt_insertonpg()` does is to check if the tuple can be inserted. It checks for example if the number of attributes in the tuple is appropriate.  

### 2. Splitting the page if necessary

First, you might wonder what it means to **split a page**. It's when there **isn't enought space** to add the item **in the target page**, so **a new page is created on the right of the page**.

These screenshots are before and after inserts of crocodiles. (I re-created a new one which is why the root block number changed ;))

![Alt text](/images/indexes/split_no_item_1.png)

On the following screenshot, you can see that a **new page appeared** on the right and **the high key changed**. That's because of the page split.

![Alt text](/images/indexes/split_no_item_2.png)

**Step 1: is a split necessary?**

In order to decide if the page should be split, `_bt_insertonpg()` compares the size of the item that we want to insert to the free space on the page.

If the **free space is lower than the item's size**, then a split is necessary.

**Step 2: finding the split point**

You could think that it would be logical to simply create a new empty page and start filling it with the new tuples.

But let's look more closely at the page split after the insert of crocodiles. Focus on the first item of the new page.

![Alt text](/images/indexes/split_item_2.png)

You can see here that it used to be in the target page.

![Alt text](/images/indexes/split_item_1.png)

Which means that, when postgres performs a page split, it does not simply create an empty page !

Postgres wants to **equalize the free space on each page** to limit page splits in future inserts. The goal being to limit as much as possible the number of pages in an index.

If it's the **right-most page on a level** (so the last page of the level), instead of cutting in half, and having two pages with 50% of the data, the **left page is filled as much as possible**.
It's useful **for serial indexes**, like the index on the id of my crocodile table `crocodile_pkey`. There is no reason to insert data on any other page than the right-most page as it's a serial integer. So when a page split is necessary, if the left page only had 50% of the items, it would be forever half empty. Which makes no sense !

**Step 3: splitting**

So now that we have the split point.

A right page is created. In the right page, **the data after the split point is copied**. The data of the target page is updated to only keep the data before split point.

Then the pointers of the pages are updated. The right pointer of the target page becomes the new page. The right pointer of the new page is the old target page right pointer.

At last the high key of the pages are updated. The **high key of the new page** becomes **the high key of the splitted target page**.
The **high key of the target page** becomes the **first value of the right page**.


### 3. Inserting on the parent level

### 4. Updating the metadata of the tree



This recursive procedure does the following things:
 *
 *			+  if necessary, splits the target page (making sure that the
 *			   split is equitable as far as post-insert free space goes).
 *			+  inserts the tuple.
 *			+  if the page was split, pops the parent stack, and finds the
 *			   right place to insert the new child pointer (by walking
 *			   right using information stored in the parent stack).
 *			+  invokes itself with the appropriate tuple for the right
 *			   child page on the parent.
 *			+  updates the metapage if a true root or fast root is split.
 

Page splits
Two levels order, the rows need to be ordered at the leaf level and at the parent level
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



# Deleting from a BTree

- Fast root
- Pages deleted only when no node and after VACCUUM
- What happens after delete from large set of data

# Conclusion

Now that you are a bit familiar with searching/inserting/deleting in a BTree, maybe you'll understand why this is the index that is used for `>, <, >=, <=, =` operators.
Because of the structure of the BTree and the way the index is traversed to look for scankey, this makes this index efficient for this operations.

The next article will cover `GIN` indexes.