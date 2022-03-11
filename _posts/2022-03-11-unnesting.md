---
layout: post
title:  "Nesting and unnesting columns in DataFrames.jl"
date:   2022-03-11 06:01:12 +0200
categories: julialang
---

# Introduction

Today I want to discuss ways to nest and unnest columns of a data frame.

We say that we nest several columns, when we take them together and turn
into one column, usually containing `NamedTuple`s.

Unnesting is a reverse process, we take a column storing e.g. `NamedTuple`s,
and create several columns out of it.

The post was written under Julia 1.7.0, DataFrames.jl 1.3.2, and Tables.jl
1.7.0.

# Column nesting

Column nesting is relatively simple in DataFrames.jl. You just need to use
`ByRow(identity)` transformation on `AsTable` source. Here is an example where
we nest all columns from a source data frame:

```
julia> using DataFrames

julia> df = DataFrame(a=1:3, b=4:6, c=7:9)
3×3 DataFrame
 Row │ a      b      c
     │ Int64  Int64  Int64
─────┼─────────────────────
   1 │     1      4      7
   2 │     2      5      8
   3 │     3      6      9

julia> transform(df, AsTable(:) => ByRow(identity) => :nested)
3×4 DataFrame
 Row │ a      b      c      nested
     │ Int64  Int64  Int64  NamedTup…
─────┼────────────────────────────────────────────
   1 │     1      4      7  (a = 1, b = 4, c = 7)
   2 │     2      5      8  (a = 2, b = 5, c = 8)
   3 │     3      6      9  (a = 3, b = 6, c = 9)
```

This works because `AsTable` passes `NamedTuple` objects to the function,
so we just need to apply `identity` row-wise to get the desired result.


# Basic column unnesting

If you want to perform a reverse process things are also relatively simple, you
just pass the nested column name as source and `AsTable` as target column name:

```
julia> df2 = select(df, AsTable(:) => ByRow(identity) => :nested)
3×1 DataFrame
 Row │ nested
     │ NamedTup…
─────┼───────────────────────
   1 │ (a = 1, b = 4, c = 7)
   2 │ (a = 2, b = 5, c = 8)
   3 │ (a = 3, b = 6, c = 9)

julia> transform(df2, :nested => AsTable)
3×4 DataFrame
 Row │ nested                 a      b      c
     │ NamedTup…              Int64  Int64  Int64
─────┼────────────────────────────────────────────
   1 │ (a = 1, b = 4, c = 7)      1      4      7
   2 │ (a = 2, b = 5, c = 8)      2      5      8
   3 │ (a = 3, b = 6, c = 9)      3      6      9
```

# Complex column unnesting

Sometimes you might have a situation where you have a nested column
that has heterogeneous contents (i.e. has different column names in different rows).
In such a scenario basic unnesting pattern does not work as it requires all rows
to have the same schema:

```
julia> df3 = DataFrame(nested = [(a=1, b=2), (b=3, c=4), (a=5, c=6)])
3×1 DataFrame
 Row │ nested
     │ NamedTup…
─────┼────────────────
   1 │ (a = 1, b = 2)
   2 │ (b = 3, c = 4)
   3 │ (a = 5, c = 6)

julia> transform(df3, :nested => AsTable)
ERROR: ArgumentError: keys of the returned elements must be identical
```

If you have such a situation you can use `Tables.dictcolumntable` as a
transformation function:

```
julia> transform(df3, :nested => Tables.dictcolumntable => AsTable)
3×4 DataFrame
 Row │ nested          a        b        c
     │ NamedTup…       Int64?   Int64?   Int64?
─────┼───────────────────────────────────────────
   1 │ (a = 1, b = 2)        1        2  missing
   2 │ (b = 3, c = 4)  missing        3        4
   3 │ (a = 5, c = 6)        5  missing        6
```

As you can see the `Tables.dictcolumntable` has "column unioning" behavior.
When some row does not have a column that is present in other rows it gets
a `missing` value instead.

# Conclusions

Column nesting and unnesting is needed when you work with data that has
hierarchical structure. A common example of such a scenario is JSON data. I
hope you will find the patterns I have discussed in this post useful in your
work.
