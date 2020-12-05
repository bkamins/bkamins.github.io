---
layout: post
title:  "DataFrames 0.22 @ JuliaAcademy"
date:   2020-12-05 17:12:14 +0200
categories: julialang
---

# Introduction

This time the blog post will be shorter than usual.

[Logan Kilpatrick][lk] has just released a new version of JuliaAcademy
course for DataFrames.jl that was updated to its 0.22 release and also contains
some new material.

You can find course materials on GitHub [here][gh], while the videos will
be released in the coming days; first two
[1. Environment Setup][v1] and
[2. First Steps With Data Frames][v2] are already
available for watching.

# Before you go

Not to leave you with just bare links let me present a short example how you can
process dates in DataFrames.jl while taking into account the possibility that
there might be `missing` values in the data (this is a question I was recently
asked how to do it).

I am using Julia 1.5.3, DataFrames 0.22.1, and Missings 0.4.4.

First we prepare a sample data frame:

```
julia> using Dates, Missings, DataFrames

julia> df1 = DataFrame(date = Date.(2020, 1, 1:10));

julia> allowmissing!(df1);

julia> df1.date[5] = missing;

julia> df1
10×1 DataFrame
 Row │ date
     │ Date?
─────┼────────────
   1 │ 2020-01-01
   2 │ 2020-01-02
   3 │ 2020-01-03
   4 │ 2020-01-04
   5 │ missing
   6 │ 2020-01-06
   7 │ 2020-01-07
   8 │ 2020-01-08
   9 │ 2020-01-09
  10 │ 2020-01-10
```

We now want to split `:date` column into three columns that will contain year,
month and day respectively. Here is the way how you can achieve it:

```
julia> df2 = transform(df1, @. :date =>
                               ByRow(passmissing([year, month, day])) =>
                               [:year, :month, :day])
10×4 DataFrame
 Row │ date        year     month    day
     │ Date?       Int64?   Int64?   Int64?
─────┼───────────────────────────────────────
   1 │ 2020-01-01     2020        1        1
   2 │ 2020-01-02     2020        1        2
   3 │ 2020-01-03     2020        1        3
   4 │ 2020-01-04     2020        1        4
   5 │ missing     missing  missing  missing
   6 │ 2020-01-06     2020        1        6
   7 │ 2020-01-07     2020        1        7
   8 │ 2020-01-08     2020        1        8
   9 │ 2020-01-09     2020        1        9
  10 │ 2020-01-10     2020        1       10
```

Note that we are using here a common pattern that you can use broadcasting to
easily specify multiple operations on the same object (in this case this is the
same source column).

Finally we go back and collect the `:year`, `:month` and `:day` columns into one
column that contains the original `Date` values:

```
julia> df3 = transform(df2, [:year, :month, :day] =>
                            ByRow(passmissing(Date)) =>
                            :date2)
10×5 DataFrame
 Row │ date        year     month    day      date2
     │ Date?       Int64?   Int64?   Int64?   Date?
─────┼───────────────────────────────────────────────────
   1 │ 2020-01-01     2020        1        1  2020-01-01
   2 │ 2020-01-02     2020        1        2  2020-01-02
   3 │ 2020-01-03     2020        1        3  2020-01-03
   4 │ 2020-01-04     2020        1        4  2020-01-04
   5 │ missing     missing  missing  missing  missing
   6 │ 2020-01-06     2020        1        6  2020-01-06
   7 │ 2020-01-07     2020        1        7  2020-01-07
   8 │ 2020-01-08     2020        1        8  2020-01-08
   9 │ 2020-01-09     2020        1        9  2020-01-09
  10 │ 2020-01-10     2020        1       10  2020-01-10
```

This time we took advantage of the fact that `Date` takes three positional
arguments and this is the default behavior of transformation specifications
in DataFrames.jl in which multiple source columns are provided.

This is all for today. Bye!

[lk]: https://github.com/logankilpatrick
[ja]: https://juliaacademy.com/p/introduction-to-dataframes-jl1
[gh]: https://github.com/JuliaAcademy/DataFrames
[v1]: https://juliaacademy.com/courses/introduction-to-dataframes-jl1/lectures/27575941
[v2]: https://juliaacademy.com/courses/introduction-to-dataframes-jl1/lectures/27575942
