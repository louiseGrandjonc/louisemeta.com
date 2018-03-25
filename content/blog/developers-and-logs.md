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

After that I became a bit obsessed with performances and realized that I would have loved to see a talk on that subject when I started working with postgreSQL and Django.

So that's the story of why I submitted a talk at the [django con](https://2017.djangocon.eu/), and then at the [pgdayParis](https://2018.pgday.paris/).

The conference at pgdayParis lasting 50 minutes, I had a bit more time to get into details. I will add the video as soon as I have it. But in the meantime here is the one from the djangocon, it's more django oriented (obviously ;) ) than the one at pgday.

{{< youtube Ph2hXpTW-Zg >}}

And here are the [slides](https://fr.slideshare.net/LouiseGrandjonc/becoming-a-better-developer-with-explain)
