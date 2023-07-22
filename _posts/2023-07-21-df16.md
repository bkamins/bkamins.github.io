---
layout: post
title:  "DataFrames.jl 1.6 for JuliaCon 2023"
date:   2023-07-21 16:11:31 +0200
categories: julialang
---

# Introduction

Next week the whole Julia community is invited to attend the
[JuliaCon 2023][juliacon] conference.

As each year wanted to have a fresh [DataFrames.jl][df] release for
this event so a few days ago DataFrames.jl 1.6 was registered.

Today I want to briefly highlight selected features of this release
that are, in my opinion, going to be useful most often in users' workflows.
If you are interested in a full list of changes please check [NEWS.md][news].

The post was written under Julia 1.9.2 and DataFrames.jl 1.6.0.

# More flexible negation selector

The `Not` selector is one of the little things that make users' life a lot easier.

Here is a basic example of how it is used:

```
julia> using DataFrames

julia> df = DataFrame(a1=1, a2=2, a3=3, b1=4, b2=5, b3=6)
1×6 DataFrame
 Row │ a1     a2     a3     b1     b2     b3
     │ Int64  Int64  Int64  Int64  Int64  Int64
─────┼──────────────────────────────────────────
   1 │     1      2      3      4      5      6

julia> df[:, Not(:a2)]
1×5 DataFrame
 Row │ a1     a3     b1     b2     b3
     │ Int64  Int64  Int64  Int64  Int64
─────┼───────────────────────────────────
   1 │     1      3      4      5      6

julia> df[:, Not([:a2, :b1])]
1×4 DataFrame
 Row │ a1     a3     b2     b3
     │ Int64  Int64  Int64  Int64
─────┼────────────────────────────
   1 │     1      3      5      6
```

In the last example you might think that adding `[...]` brackets is redundant.
With DataFrames.jl release 1.6 it is not needed anymore:

```
julia> df[:, Not(:a2, :b1)]
1×4 DataFrame
 Row │ a1     a3     b2     b3
     │ Int64  Int64  Int64  Int64
─────┼────────────────────────────
   1 │     1      3      5      6
```

This flexibility extends to arbitrary column selectors:

```
julia> df[:, Not(r"a", r"2")]
1×2 DataFrame
 Row │ b1     b3
     │ Int64  Int64
─────┼──────────────
   1 │     4      6
```

In the example we dropped columns that contained substrings `"a"` or `"2"`.

# Allow column renaming when constructing a data frame

Often we have source data that has some predefined column names.
For example consider the following named tuple of vectors:

```
julia> nt = (a=1:2, b=3:4, c=5:6)
(a = 1:2, b = 3:4, c = 5:6)
```

We can easily create a `DataFrame` from it:

```
julia> DataFrame(nt)
2×3 DataFrame
 Row │ a      b      c
     │ Int64  Int64  Int64
─────┼─────────────────────
   1 │     1      3      5
   2 │     2      4      6
```

Note that column names in the created data frame were inherited from
the source table. In the past, if we wanted different column names we
would need to rename them:

```
julia> rename!(DataFrame(nt), ["x", "y", "z"])
2×3 DataFrame
 Row │ x      y      z
     │ Int64  Int64  Int64
─────┼─────────────────────
   1 │     1      3      5
   2 │     2      4      6
```

Since DataFrames.jl release 1.6 you can do data frame creation and column renaming in one step:

```
julia> DataFrame(nt, ["x", "y", "z"])
2×3 DataFrame
 Row │ x      y      z
     │ Int64  Int64  Int64
─────┼─────────────────────
   1 │     1      3      5
   2 │     2      4      6
```

In the example - the first argument is the source table, and the second argument are target column names.

# Conclusions

I hope all users will enjoy the new release of DataFrames.jl.

If you plan to attend JuliaCon 2023 let me highlight several events
that will happen during the conference in which I am involved:

* on Tuesday, July 25 there will be "Working with DataFrames.jl beyond CSV files" workshop.
* on Wednesday, July 26, I plan to be present at the "book authors' booth" in the conference
  where I can answer any questions you have regarding my "Julia for Data Analysis" book.
  I will have several print and electronic versions of the book to share with interested
  people.
* on Thursday, July 27, There is a "Tools and techniques of working with tabular data" minisymposium.
  I will give a talk about the development status of DataFrames.jl.
* on Friday, July 28, the "Statistics symposium" minisymposium; I will give a talk on doing
  basic econometrics with GLM.jl. Next the "Future of JuliaData ecosystem" Birds of Feather meetup will take place.

[juliacon]: https://juliacon.org/2023/
[df]: https://github.com/JuliaData/DataFrames.jl
[news]: https://github.com/JuliaData/DataFrames.jl/blob/main/NEWS.md
