---
layout: post
title:  "DataFrames.jl at the Journal of Statistical Software"
date:   2023-09-29 17:31:13 +0200
categories: julialang
---

# Introduction

This week, I have reached a small personal milestone as
the Journal of Statistical Software has just published
the [DataFrames.jl: Flexible and Fast Tabular Data in Julia][jss]
paper that I co-authored with Milan Bouchet-Valat.

Therefore, in this post, I thought to summarize various resources
I have been working on over the years that can help you master DataFrames.jl.

# Types of documentation

It is hard to document a package properly. The reason is that different users
have different expectations. In [this post][docs] there is an excellent summary of
four typical kinds of documentation:

* learning-oriented tutorials;
* goal-oriented how-to guides;
* understanding-oriented discussions;
* information-oriented reference material.

Each kind of documentation requires a slightly different approach, and
having it all in a single place is hard. In the following sections, I will go through
various materials I have prepared over the years and explain my intention
behind them.

# The Journal of Statistical Software paper

The objective of the [DataFrames.jl: Flexible and Fast Tabular Data in Julia][jss] paper
is understanding-oriented. Together with Milan we tried to explain in it how the
DataFrames.jl package was designed, and what were the motivations behind these decisions.

For this reason, you most likely cannot learn how to work with DataFrames.jl after reading
the paper. However, you will get an intuition what are the basic building blocks
of the package.

It is similar to learning languages. I have recently decided to learn French.
As a part of this process, I watched the [ALL THE RULES OF FRENCH IN 20 MINUTES][yt] video on YouTube.
I did not directly learn French from it, but it helped a lot in understanding the "design" of the French language.

# Julia for Data Analysis book

Another major resource I have created is my [Julia for Data Analysis][jda] book.
This resource is learning-oriented. I start it from the basics of the Julia language
and gradually add more complex elements so that eventually, the reader should be able to:

* read and write data in various formats;
* work with tabular data, including subsetting, grouping, and transforming;
* visualize data;
* build predictive models;
* create data processing pipelines;

and more.

As you can probably guess, the DataFrames.jl package is a backbone of this material.
The book was prepared as a textbook that can be used in a 1-semester introductory course on data analysis using Julia
and is accompanied by numerous extra materials that can be found [here][ghjda].

# Tutorials

Over the years, I have created a lot of goal-oriented how-to guides.
You can find their list [here][guides].

Also, [my blog][blog] since year 2020 brings you each week some practical information on working with Julia
(quite often DataFrames.jl oriented).

Finally, as a part of DataFrames.jl documentation [here][docs1] and [here][docs2] there are available introductory tutorials to DataFrames.jl.

Since the volume of the tutorials is large, it might be sometimes a bit hard to navigate, but probably this is unavoidable, as there is a substantial variety of questions that users might have.

# Reference

I strongly prefer implementing the functionality of DataFrames.jl following the contract specified in the documentation
of provided functions. Therefore, I believe that we have quite a strong collection of reference materials that make it precise
how DataFrames.jl functionality is implemented. It is divided into four major parts:

* specification of how types exposed by DataFrames.jl are designed is given [here][rtypes];
* reference on provided functions can be found [here][rfunctions];
* a complete description of how indexing works in DataFrames.jl is available [here][rindexing];
* information on how data frames handle table and column metadata is given [here][rmetadata].

It is essential to highlight that these materials aim to be complete and precise. Unfortunately, this means
that they are verbose and sometimes hard to digest by new users. Unfortunately, I think this cannot be helped,
and that is why we provide other kinds of documentation to make it easier to get started with DataFrames.jl

# Conclusions

I hope that this post can serve DataFrames.jl users as a helpful guide to different resources I have co-authored
that are provided to make learning and using the package easy and fun. Enjoy!

[jss]: https://www.jstatsoft.org/article/view/v107i04
[docs]: https://www.writethedocs.org/videos/eu/2017/the-four-kinds-of-documentation-and-why-you-need-to-understand-what-they-are-daniele-procida/
[yt]: https://www.youtube.com/watch?v=UVlmXWhJbzI
[jda]: https://www.manning.com/books/julia-for-data-analysis
[ghjda]: https://github.com/bkamins/JuliaForDataAnalysis
[guides]: https://dataframes.juliadata.org/stable/#DataFrames.jl
[blog]: https://bkamins.github.io/
[docs1]: https://dataframes.juliadata.org/stable/man/basics/
[docs2]: https://dataframes.juliadata.org/stable/man/getting_started/
[rtypes]: https://dataframes.juliadata.org/stable/lib/types/
[rfunctions]: https://dataframes.juliadata.org/stable/lib/functions/
[rindexing]: https://dataframes.juliadata.org/stable/lib/indexing/
[rmetadata]: https://dataframes.juliadata.org/stable/lib/metadata/
