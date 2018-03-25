---
author: "Louise Grandjonc"
date: 2018-03-24T15:39:21-07:00
linktitle: Understanding EXPLAIN - part 2 - Scans
title: Understanding EXPLAIN - part 2 - Scans
weight: 1
---


## Sequential scan

On the image above, you can see a `seq scan`. The sequential scan simply reads the entire table and retrieves the rows matching the `where` clause.

## Index scan

If you see instead of the `seq scan` and `index scan` it means that and index is used to filter you rows. If the concept of indexes isn't quite clear, I encourage you to [watch](https://youtu.be/Ph2hXpTW-Zg?t=16m43s) my conference where I explain it.

## Bitmap heap scan

# Conclusion
