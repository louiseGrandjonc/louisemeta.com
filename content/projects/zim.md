+++
date = "2018-03-23T22:25:02-07:00"
title = "stephanzimmerli.com"
image = "zimmerli.png"
alt = "stephanzimmerli.com - A website for an artist"
color = "#60d2d3"
link1 = "/projects/zimmerli"
+++



![Alt text](/images/zimmerli.png)

[Stephan Zimmerli](https://stephanzimmerli.com) is an artist who has a tendancy to be brilliant in several fields: architecture, stage design, drawing, photography, music... So when he came to me with the project of a website that would allow him to show his work, always in between this art forms, I was thrilled.

The website is still in construction, we are building what he calls a mnemotopy, an architecture, like a house, to visit twenty years of notebooks, music, and videos.

There was also an other aspect, he wanted this website to also be his archives, to be able to search in it, and to have all the pictures, movies, and sounds saved so that, if his computer died he wouldn't loose all of it. For that I used AWS S3 to store the files. But some of the files are quite heavy, so I had to find a better way so that the website would load well. I am currently working on the fulltext search with PostgreSQL, and on the deployment.

This project was quite amazing for me for many reasons, on a technical level and because of the creativity that was needed.

You can find the code [here](https://github.com/louiseGrandjonc/mnemotopy).
The website is [here](https://stephanzimmerli.com) but it's still under contruction :)