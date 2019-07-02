---
author: "Louise Grandjonc"
date: 2019-07-01T10:20:22-07:00
linktitle: PostgreSQL's indexes - GIN algorithms
title: PostgreSQL's indexes - GIN algorithms
weight: 1
---


# Introduction

In the [previous article](/blog/indexes-gin), I introduced the internal data structure of a postgres GIN index. Now it's time to talk about searching and maintaining them.


# Searching in a GIN index

When postgres uses the GIN index, you'll notice in the EXPLAIN that it's always doing a bitmap heap scan. It means that the search algorithm returns a map of tids (pointers to rows) in physical memory order.

Considering the way the data is stored inside the posting lists and trees, it makes a lot of sense to use this method.

The function that performs the search and builds the bitmap is [`gingetbitmap`](https://github.com/postgres/postgres/blob/master/src/backend/access/gin/ginget.c#L1849).

## Step 1: scan keys
As for the search in a BTree, **scan keys are used**.

So the first thing that this function does is to **initialise the scan keys**.
- free previous scan keys,
- init scan keys

## Step 2: Scanning the pending list

As I said earlier, when looking into a GIN index, the pending list and the main BTree have to be scan. This step concerns the pending list.

[`scanPendingInsert`](https://github.com/postgres/postgres/blob/master/src/backend/access/gin/ginget.c#L1753) does that.

- First, the **metapage is locked**. This lock prevents from changes in the pending list (in case of fast insert where rows would be added or VACUUM that could empty the list)
- We get the block number of the head of the pending list. If the block number is not a valid one, it means that there is no pending list. The locks can be dropped.
- Then the list is fetched into a `pendingBuffer`. Once the buffer is set, we can now drop the previous locks.
- For each row, if the row matches all scan keys, it is added to the bitmap using the funciton `tbm_add_tuples`

## Step 3: scanning the main index

The function `startScan` is called to scan the main tree of a GIN index.

The first thing that does is to search for the entries that appear in the scan. For that it's using the function `GinFindLeafPage` for each entry. This is (again) close to the behaviour in a BTree, the main BTree is traversed, until the leaf where the entry could be found. If necessary, the posting tree has to be explored as well to retrieve the TIDs.

The next step is to actually match the scan keys. If the scan key only has one entry, no other work is necessary, we already found the TIDs matching the scankey.

This is the case for example if I search for the appointments that were for crocodile with caries.

```code
croco_new=# SELECT COUNT(*) FROM appointment WHERE symptoms_vector @@ to_tsquery('caries');
 count 
-------
 35859
(1 ligne)
```

But let's say one of the plover birds is doing a research to find the appointments for crocodiles with caries and that had osteoplasty.

In the database I have 35859 appointments with the syndrom description containing the word carie, and only 501 with the word osteoplasty.

So the query to find the appointments is:

```code
SELECT * FROM appointment
WHERE symptoms_vector @@ to_tsquery('osteoplasty & caries');
```

So first the search algorithm finds the leaves pages for the entries `osteoplasty` and `carie`.

But the scan key in this case is a `AND`. So finding the entries is not enough to match the scan key.

The function [`startScanKey`](https://github.com/postgres/postgres/blob/master/src/backend/access/gin/ginget.c#L500) optimises the match by deviding the entries into two categories, the required ones and the additionnal ones.

To do that, first the entries are ordered by frequency. In my case, caries is more frequent than osteoplasty. Postgres then tries to put as many frequent entries as possible in the additional entries.

So here my required set would contain the entry `osteoplasty`, and the additional one the entry `carie`. Thanks to that, we can take the items matching osteoplasty, and when going over the items matching carie, we can just skip any TIDs that wasn't already in the osteoplasty entry. As there are 501 appointments with osteoplasty and 35859 for caries, this optimisation is very interesting.

## Step 4: updating the bitmap

Once we retrieved from the main tree the items matching the scan keys, we want to update the bitmap. Indeed, there are already elements in it coming from the pending list.

## To sum up

In order to search in a GIN index,

- First the pending list is explored and a bitmap created with the matching rows
- Then the entries are searched in the main GIN structure
- If the scankey had a single entry, we have the TIDs of the matching rows
- Else, we need to find the right items matching the scan key. The use of required and additional entries helps with the optimisation of this step.
- Finally the bitmap is updated

# Inserting in a GIN index

There are two ways data is inserted in a GIN index
Either `fastupdate` is set to `true` and the pending list is updated and the rows are moved from the list to regular GIN structure.
Or `fastupdate` is set to `false` and for each insert the tree is updated.

First let's talk about fast update.


## Updating the pending list

`ginHeapTupleFastInsert`: write the tuples to be inserted in the pending list.

- Checks if the tuples should be in a new list. For that it checks if the tuples fit in one page, if there is already a pending list and if there is enough space in the last page of the pending list to put the new tuples. If any of these three condition fails, the function `makeSublist` is called
- If needed the pointer to head and tail of the pending list (in case of a new page) in the metapage are updated

- If no new page, the tuples are simply inserted in the tail page


## Inserting in the leaf

`ginInsertCleanup` -> move data to GIN structure. It calls `GinEntryInsert` with the list of tuple with the say key.
Once the data in moved to the main BTree, we reclean the one that might have been added during the clean up, and then mark the page as deleted. Deleting the data from the pending list after inserting makes it crash safe.

The function `GinEntryInsert` can be called with a list of tids or with a single onefor a key. This function is used for fastupdate and for "normal" insert.

The function finds the right leaf page using `GinFindLeafPage` where the entry should be inserted.

If **the value is already** in the **main BTree**, it means that the **posting tree** or **posting list** has to **be updated**.
Updating a posting list can be tricky (`addItemPointersToLeafTuple`). First, the list is updated and compressed. If the size of the list reaches the `GinMaxItemSize`, a posting tree has to be created.
In this case, the function `GinEntryInsert` calls `createPostingTree` with the pointers that were in the posting list. As they are already in order, it makes it faster to create the posting tree.

If the **value wasn't in the main BTree**, the function `buildFreshLeafTuple` is called to add a leaf tuple in the current leaf page with a posting list (or tree) containing the new items. This might lead to a page split. To know more about page split, you can refer to the [previous article](/blog/indexes-btree-algorithms#inserting-the-tuple-and-page-splits) as the page splits here are very similar. 

## Recursing to the parents

If a new tuple had to be inserted in a leaf page and it led to a page split, the parent needs to be updated to contain the downlink to the new page. I'm not going to go into much details here as this is quite similar to the behaviour in a BTree.

If you want to have fun you can look into the source code of the function [`ginPlaceToPage`](https://github.com/postgres/postgres/blob/master/src/backend/access/gin/ginbtree.c#L333) that recursively inserts in the parents and handles the page splits.


## To sum up

Inserting in a GIN index can be in 2 or 3 steps.

- If `fastupdate` is set to true: the new tuples are inserted in the pending list
- Once the pending list is full, or there is a VACUUM, or `fastupdate` was set to false, the tuples are inserted in the main GIN tree. To do that:
  - We look for the right leaf page to update
  - If the value is already in it, the posting list or tree is updated
  - Else a new item is inserted in the leaf, and if a page split occured, it recurses back to the parent.

So basically, the only big difference with a BTree in this algorithm is the pending list and the insert in the leaves as the values are unique in a GIN index. This is also what slows inserts. I would probably advise you to use `fastupdate` (with a proper use of vacuum) as it will help have better write performances. But it really depends of your use case.

In my story, crocodiles are going to have tooth ache and come to ask for an appointment, describe their syndrom etc. thousands of times a day (not each croco though), so it would make crocodiles very angry to have to wait for the GIN index to be updated. And I mean, crocodiles are not the animal you want to mess up with...


# Deleting from a GIN index


file ginvacuum
- A page from a posting tree has to be deleted
- Update the posting list, delete single pointer
- An item has to be deleted from leaf (no more pointer to this value)


# Conclusion

The GIN index is powerful because of many reasons, it can be very efficient for fulltext search, or array values, but it's also an heavy index. Updating it can be up to 10 times slower than a GiST index if `fast_update` is set to `False`.

I will soon post an article of GiST that has some similar use cases, and we will compare the two to help you choose widely.