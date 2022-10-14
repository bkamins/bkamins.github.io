---
layout: post
title:  "130 graded exercises to train your Julia for data analysis muscle"
date:   2022-10-14 10:51:52 +0200
categories: julialang
---

# Introduction

My [Julia for Data Analysis][jda] book will be soon published (now all its
chapters are already available in preview for free).

An important part of the book is [its GitHub repository][gh] containing
all the codes used in the book and ensuring their reproducibility.

Since the book was prepared to fit one semester course on data analysis using
Julia I am now preparing supporting teaching materials that accompany it.

Today together with Daniel Kaszyński we have released first part of these
supporting materials. In the [exercises][gh1] folder of the
[book's GitHub repository][gh] we have added 130 exercises that should help you
master the material covered in the book.

[The exercises][gh1] are grouped by book chapter. There are 10 exercises for
each chapter. Each exercise has a proposed solution. We have prepared the
exercises so that they have a varying difficulty level. The exercises from
initial chapters should be relatively easy. However, to solve exercises
from the final chapters you might need to have a significant knowledge of
Julia's ecosystem for data analysis.

In the post I use Julia 1.8.2, and DataFrames.jl 1.4.1.

# A sample exercise

To have some concrete example of what a typical exercise is I have picked a
[question][discourse] that was asked today on Discourse that I liked. The
problem is stated as follows.

Consider the following data frame:

```
julia> using DataFrames

julia> df = DataFrame(country=["Poland", "Poland", "Canada", "Canada"],
                      city=["Olecko", "Ełk", "Toronto", "Mississauga"])
4×2 DataFrame
 Row │ country  city
     │ String   String
─────┼──────────────────────
   1 │ Poland   Olecko
   2 │ Poland   Ełk
   3 │ Canada   Toronto
   4 │ Canada   Mississauga
```

The task is to reduce it by unique value in `country` column. More specifically
we want to create a new data frame with two columns. One of them should be
`country` that will store unique values of `country` column in the source data
frame `df`. The second column should be `cities` that should store a vector
of values in the `city` column from `df` that correspond to a given country.

Now let me show three ways how you can do it using the `combine` function.
The key to the solution is the following rule of how `combine` works
(taken from the documentation):

> In all of these cases, `function` can return either a single row or multiple
> rows. As a particular rule, values wrapped in a `Ref` or a
> `0-dimensional AbstractArray` are unwrapped and then treated as a single row.

This means that in order to make a vector to be treated as a single row we have
three options:
* wrap a vector in another vector as its single element (so we have a multi-row
  object but with a single row);
* wrap a vector in `Ref`;
* wrap a vector in a `0-dimensional AbstractArray`, which can be done using the
  `fill` function.

So the three solutions to our problem are:

```
julia> combine(groupby(df, :country, sort=true), :city => (x -> [x]) => :cities)
2×2 DataFrame
 Row │ country  cities
     │ String   SubArray…
─────┼─────────────────────────────────────
   1 │ Canada   ["Toronto", "Mississauga"]
   2 │ Poland   ["Olecko", "Ełk"]

julia> combine(groupby(df, :country, sort=true), :city => Ref => :cities)
2×2 DataFrame
 Row │ country  cities
     │ String   SubArray…
─────┼─────────────────────────────────────
   1 │ Canada   ["Toronto", "Mississauga"]
   2 │ Poland   ["Olecko", "Ełk"]

julia> combine(groupby(df, :country, sort=true), :city => fill => :cities)
2×2 DataFrame
 Row │ country  cities
     │ String   SubArray…
─────┼─────────────────────────────────────
   1 │ Canada   ["Toronto", "Mississauga"]
   2 │ Poland   ["Olecko", "Ełk"]
```

# Conclusions

I hope you will enjoy and benefit from doing the exercises that I have added
to the book's GitHub repository.

Expect that soon two other sets of materials will be added to this repository:

* For each chapter you will get a notebook which can serve as a starting point
  to develop teaching materials for a given chapter.
* Several openly available notebooks with additional data analysis problems that
  are solved end-to-end. They will be prepared in a
  similar style to [Hands-on Data Science with Julia][hdsj] notebooks and are
  meant to be studied as a follow-up material after you have studied the book.

When these are added I will post about it.

[jda]: https://www.manning.com/books/julia-for-data-analysis?utm_source=bkamins&utm_medium=affiliate&utm_campaign=book_kaminski2_julia_3_17_22
[gh]: https://github.com/bkamins/JuliaForDataAnalysis
[gh1]: https://github.com/bkamins/JuliaForDataAnalysis/tree/main/exercises
[discourse]: https://discourse.julialang.org/t/reduce-dataframe-by-unique-value/88723
[hdsj]: https://www.manning.com/liveprojectseries/data-science-with-julia-ser
