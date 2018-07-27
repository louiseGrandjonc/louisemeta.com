---
author: "Louise Grandjonc"
date: 2018-06-29T10:20:21-07:00
linktitle: PostgreSQL's indexes - GIN
title: PostgreSQL's indexes - GIN
weight: 1
draft: true
---

# Introduction

GIN indexes are often used to index arrays, jsonb, and tsvector (for fulltext search) columns.
When it comes to array, they are, for example, especially performant to verify if an array contains a particular element.

[In the documentation](https://www.postgresql.org/docs/current/static/gin-builtin-opclasses.html) you can see the full list of operators.

But the question that I want to answer in this article is: "why should we use GIN indexes for this data types and operators ?"

I'm going to go over two examples, one for arrays and one for fulltext search.

So now, the plover birds want to have, for each crocodile, the list of teeth that they had to heal. Because it is much simpler when they see again the crocodile to just know which teeth could be more fragile, and I mean, they see a lot of crocodiles every day, plover birds don't have an elephant memory.

So I added a new column `healed_teeth`, an integer array (`integer[]`). So if I take a totally random crocodile, I'll have:

```code
croco=# SELECT email, number_of_teeth, healed_teeth FROM crocodile WHERE id =1;
-[ RECORD 1 ]---+--------------------------------------------------------
email           | louise.grandjonc1@croco.com
number_of_teeth | 58
healed_teeth    | {16,11,55,27,22,41,38,2,5,40,52,57,28,50,10,15,1,12,46}
```

And then I created the following index:

```code
CREATE INDEX ON crocodile USING GIN(healed_teeth);
```


As for fulltext search, in this new updated version, crocodiles can give their symptoms to help plover bird know what's wrong.

So I added a column `symptoms` to the `appointment` table. As we index `tsvector`, I also added a `symptoms_vector` column.

```code
croco=# ALTER TABLE appointment ADD COLUMN symptoms text;
ALTER TABLE
croco=# ALTER TABLE appointment ADD COLUMN symptoms_vector tsvector;
ALTER TABLE
```

I filled the symptoms with some random dentistry related words that I found on [this website](http://www.suvison.com/hp_fdi_fp.asp). So now I have records looking like this.

```code
croco=# SELECT * FROM appointment WHERE symptoms IS NOT NULL ORDER BY id LIMIT 10;
-[ RECORD 1 ]---+-------------------------------------------------------------
id              | 12809
crocodile_id    | 1
plover_bird_id  | 21970
emergency_level | 7
done            | f
area            | 0101000020E61000000000000000005CC000000000000032C0
schedule        | ["2015-01-03 17:49:00+01","2015-01-03 18:49:00+01")
symptoms        | nozzle, syringe, gingivitis, acute necrotizing ulcerative (ANUG),
                  cornification, globular (adj), loss of primary teeth (SEE shedding
                  of primary teeth), muscle, canine, wear, occlusal, stricture,
                  vitrodentine, apposition, hazard, radiation, system, nervous,
                  sympathetic, eleidine
symptoms_vector | 'acut':4 'adj':10 'anug':7 'apposit':26 'canin':21 'cornif':8
                  'eleidin':32 'gingivit':3 'globular':9 'hazard':27 'loss':11
                  'muscl':20 'necrotizing':5 'nervous':30 'nozzl':1 'occlusal':23
                  'of':12,17 'primary':13,18 'radiat':28 'se':15 'shedding':16
                  'strictur':24 'sympathetic':31 'syring':2 'system':29
                  'teeth':14,19 'ulcer':6 'vitrodentin':25 'wear':22
```

(These crocodiles have much more dentist vocabulary than me...). And then I created a GIN index on the `symptoms_vector` column.

```code
CREATE INDEX ON appointment USING GIN(symptoms_vector);
```

# GIN structure

- pas de repetition des valeurs

Basically, a GIN index is a BTree. The big difference between a BTree index and a GIN index is that the values are unique in a GIN index.


- les feuilles sont des listes (posting list) ou BTree (posting tree)

- liste en attente d'être dans l'index (optimisation de l'insert), jusqu'au prochain vacuum



Explaining the non repetition of values,
Explaining how the leaves are organized, pointer to an other BTree, or list, or only value

# Searching in a GIN

Explaining the algorithm

Example for a the emergency level of appointments (only 10 values in 2M rows), comparison with a BTree
Example of fulltext search comparison with BTree

# Inserting, deleting in a GIN

Warning with example on the loss of performances compared to a normal BTree