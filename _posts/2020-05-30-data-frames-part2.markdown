---
layout: post
title:  "Tutorials for DataFrames.jl release 0.21. Part II"
date:   2020-05-30 14:07:21 +0200
categories: julialang
---

This is a follow up post to Part I that covered new functionalities in
DataFrames.jl release 0.21, which you can find [here][part1].

# New material

I have created two additional notebooks that can be downloaded from
[this GitHub Gist][gist].

The content is shared as two notebooks:

1. *airports.ipynb* that mostly focuses on transformation operations and
   the syntax `source => function => sink`.
2. *bison.ipynb* that discusses new functionalities related to population of
   data frames with heterogeneous data that were added to `push!` and `append!`.

I hope you will find the new examples useful.

# Environment setup

The codes were tested under Julia 1.4.1.

Please make sure that you download all materials from the Gist. In particular
having proper *Project.toml* and *Manifest.toml* files will ensure that you have
right versions of the packages that were used.

Also note that for the first part of the tutorial (*airports.ipynb*) you need to
download the file I used from Kaggle (it is quite large). For the second
notebook *bison.ipynb* the JSON file *bison.json* is bundled in the gist.

If you would have any questions regarding the materials please do not hesitate
to [contact me][contact].

[part1]: https://bkamins.github.io/julialang/2020/05/25/data-frames-part1.html
[gist]: https://gist.github.com/bkamins/c6b7b536aefda6c2be67f133972a62e8
[contact]: https://bkamins.github.io/about/
