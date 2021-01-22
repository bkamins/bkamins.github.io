---
layout: post
title:  "Mass transformations of data frames how-to"
date:   2021-01-22 09:51:53 +0200
categories: julialang
---

# Introduction

A very common question related to the usage of [DataFrames.jl][df] is how
to perform mass transformations of data frames. Typically users want to apply
the same function to all columns, rows, or individual cells of a data frame.

In this post I want to summarize basic patterns allowing to perform these tasks.
I split the examples by the type of task performed and the requested type of the
output of the operation.

The code was tested under Julia 1.5.3 and DataFrames 0.22.2.

In the post we will consider the following source data frame:

```
julia> using DataFrames

julia> df = DataFrame(reshape(1:24, 6, 4), :auto)
6×4 DataFrame
 Row │ x1     x2     x3     x4
     │ Int64  Int64  Int64  Int64
─────┼────────────────────────────
   1 │     1      7     13     19
   2 │     2      8     14     20
   3 │     3      9     15     21
   4 │     4     10     16     22
   5 │     5     11     17     23
   6 │     6     12     18     24
```

Note that it is important that all columns of the data frame have the same type
as usually when we apply mass transformations to different columns this
condition is required to be met (it is not a strict rule that this is the case,
but I have found that e.g. trying to apply a function that works on floats to
strings is one of the most common cases causing confusion of the users).

# Each column to a vector

If you want to apply a transformation to each column and get a vector as a
result then use the `eachcol` iterator for this. Here are some options you might
find useful:

```
julia> sum.(eachcol(df))
4-element Array{Int64,1}:
  21
  57
  93
 129

julia> map(sum, eachcol(df))
4-element Array{Int64,1}:
  21
  57
  93
 129

julia> [sum(x) for x in eachcol(df)]
4-element Array{Int64,1}:
  21
  57
  93
 129

julia> [name => sum(x) for (name, x) in pairs(eachcol(df))]
4-element Array{Pair{Symbol,Int64},1}:
 :x1 => 21
 :x2 => 57
 :x3 => 93
 :x4 => 129
```

# Each column to a data frame

If you want to produce a data frame as a result of applying a function to all
columns you can either use `mapcols` or `combine`:

```
julia> combine(df, names(df) .=> sum)
1×4 DataFrame
 Row │ x1_sum  x2_sum  x3_sum  x4_sum
     │ Int64   Int64   Int64   Int64
─────┼────────────────────────────────
   1 │     21      57      93     129

julia> combine(df, names(df) .=> sum, renamecols=false)
1×4 DataFrame
 Row │ x1     x2     x3     x4
     │ Int64  Int64  Int64  Int64
─────┼────────────────────────────
   1 │    21     57     93    129

julia> combine(df, names(df) .=> sum .=> names(df), renamecols=false)
1×4 DataFrame
 Row │ x1     x2     x3     x4
     │ Int64  Int64  Int64  Int64
─────┼────────────────────────────
   1 │    21     57     93    129

julia> mapcols(sum, df)
1×4 DataFrame
 Row │ x1     x2     x3     x4
     │ Int64  Int64  Int64  Int64
─────┼────────────────────────────
   1 │    21     57     93    129
```

In general, as you can see `mapcols` was designed to handle this scenario, while
`combine` can be used when you would want to perform more different
transformations of the passed data frame (at the cost of being more verbose).

# Each row to a vector

In this case you have two major. The basic one is to use `eachrow`:

```
julia> sum.(eachrow(df))
6-element Array{Int64,1}:
 40
 44
 48
 52
 56
 60

julia> map(sum, eachrow(df))
6-element Array{Int64,1}:
 40
 44
 48
 52
 56
 60

julia> [sum(x) for x in eachrow(df)]
6-element Array{Int64,1}:
 40
 44
 48
 52
 56
 60
```

This should be OK for most cases. The problem with this approach is that
`eachrow` is not type stable. So when you have very many rows or need column
type information in the values passed to the aggregation function use
`Tables.namedtupleiterator`:

```
julia> sum.(Tables.namedtupleiterator(df))
6-element Array{Int64,1}:
 40
 44
 48
 52
 56
 60

julia> map(sum, Tables.namedtupleiterator(df))
6-element Array{Int64,1}:
 40
 44
 48
 52
 56
 60

julia> [sum(x) for x in Tables.namedtupleiterator(df)]
6-element Array{Int64,1}:
 40
 44
 48
 52
 56
 60
```

whih will be faster and type stable (but at the cost of having to be compiled,
which can be problematic if you have a lot of columns in you data frame as I
have recently explained in [this post][post]).

You might ask when one wants type stability in the context of small tables. Here
is an example:

```
julia> df2 = DataFrame(x1=[1, 2, missing], x2 = [1, missing, missing])
3×2 DataFrame
 Row │ x1       x2
     │ Int64?   Int64?
─────┼──────────────────
   1 │       1        1
   2 │       2  missing
   3 │ missing  missing

julia> (sum∘skipmissing).(Tables.namedtupleiterator(df2))
3-element Array{Int64,1}:
 2
 2
 0

julia> (sum∘skipmissing).(eachrow(df2))
ERROR: ArgumentError: reducing over an empty collection is not allowed
```

As you can see in the last row of `df2` we have only `missing` values. If we are
in a type stable context, `sum` knows that it should produce an integer `0`,
while in a type unstable context we get an error as it is impossible to tell
what should be the type of `0` that should be produced.

# Each row to a data frame

This case is typically handled by using the `combine` or the `select` functions
(which in the considered scenario produce the same output) along with the
`ByRow` wrapper. Here are two examples differing in whether we pass rows as
consecutive positional arguments or as a `NamedTuple` to an aggregation
function:

```
julia> combine(df, names(df) => ByRow(+) => :sum)
6×1 DataFrame
 Row │ sum
     │ Int64
─────┼───────
   1 │    40
   2 │    44
   3 │    48
   4 │    52
   5 │    56
   6 │    60

julia> combine(df, AsTable(names(df)) => sum => :sum)
6×1 DataFrame
 Row │ sum
     │ Int64
─────┼───────
   1 │    40
   2 │    44
   3 │    48
   4 │    52
   5 │    56
   6 │    60
```

Note that in the `NamedTuple` passing option we are type stable so the following
code works as in the example from the previous section:

```
julia> combine(df2, AsTable(names(df2)) => ByRow(sum∘skipmissing) => :sum)
3×1 DataFrame
 Row │ sum
     │ Int64
─────┼───────
   1 │     2
   2 │     2
   3 │     0
```

# Each cell to a matrix

In order to transform each cell and store the result in a matrix you have the
following basic options:

```
julia> Matrix(df) .^ 2
6×4 Array{Int64,2}:
  1   49  169  361
  4   64  196  400
  9   81  225  441
 16  100  256  484
 25  121  289  529
 36  144  324  576

julia> Matrix(df .^ 2)
6×4 Array{Int64,2}:
  1   49  169  361
  4   64  196  400
  9   81  225  441
 16  100  256  484
 25  121  289  529
 36  144  324  576

julia> [df[i, j]^2 for i in axes(df, 1), j in axes(df, 2)]
6×4 Array{Int64,2}:
  1   49  169  361
  4   64  196  400
  9   81  225  441
 16  100  256  484
 25  121  289  529
 36  144  324  576
```

In general the first of them (conversion to a `Matrix` and then working with it)
should be fastest.

# Each cell to a data frame

In this case one can use the same pattern as the second one above. Just write:

```
julia> df .^ 2
6×4 DataFrame
 Row │ x1     x2     x3     x4
     │ Int64  Int64  Int64  Int64
─────┼────────────────────────────
   1 │     1     49    169    361
   2 │     4     64    196    400
   3 │     9     81    225    441
   4 │    16    100    256    484
   5 │    25    121    289    529
   6 │    36    144    324    576
```

# Conclusions

Now you should have a basic understanding of different options how data frame
can be transformed by-row, by-column, or by-cell. I have skipped the discussion
of analogous operations for `GroupedDataFrame`. If you would want to perform
them per-group then using the `combine` or `select` examples given above will
just work also for `GroupedDataFrame` as a source of data.

[df]: https://github.com/JuliaData/DataFrames.jl
[post]: https://bkamins.github.io/julialang/2021/01/08/typestable.html
