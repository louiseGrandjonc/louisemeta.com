---
author: "Louise Grandjonc"
date: 2018-06-29T10:20:21-07:00
linktitle: PostgreSQL's indexes - GIN
title: PostgreSQL's indexes - GIN
weight: 1
draft: true
---

# Introduction

GIN indexes are typically used for fulltext search, why? What are other use cases and why ?

# GIN structure

Explaining the non repetition of values,
Explaining how the leaves are organized, pointer to an other BTree, or list, or only value

# Searching in a GIN

Explaining the algorithm

Example for a the emergency level of appointments (only 10 values in 2M rows), comparison with a BTree
Example of fulltext search comparison with BTree

# Inserting, deleting in a GIN

Warning with example on the loss of performances compared to a normal BTree