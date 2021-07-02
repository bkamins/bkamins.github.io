---
layout: post
title:  "What is new in DataFrames.jl 1.2.0?"
date:   2021-07-02 06:15:24 +0200
categories: julialang
---

# Introduction

[DataFrames.jl][df] version 1.2.0 has just been released. In this post I want
to discuss the main new user visible features we have introduced.

The codes were run under Julia 1.6.1 and DataFrames.jl 1.2.0.

# New functionalities

There are three major new functionalities introduced by the 1.2.0 release. Let
me explain them one by one.

#### `matchmissing=:notequal` keyword argument in joins

Before 1.2.0 release `missing` values in `on`-columns in joins either were
considered to be equal (when `matchmissing=:equal` was passed) or produced an
error (when `matchmissing=:error`, this is a default behavior). Now you can
also pass `matchmissing=:notequal` in which case `missing` values are considered
as not matching. Here is a simple example comparing the three options:

```
julia> using DataFrames

julia> df1 = DataFrame(id=[1, missing, 3], left=1:3)
3×2 DataFrame
 Row │ id       left
     │ Int64?   Int64
─────┼────────────────
   1 │       1      1
   2 │ missing      2
   3 │       3      3

julia> df2 = DataFrame(id=[1, missing, missing], right=1:3)
3×2 DataFrame
 Row │ id       right
     │ Int64?   Int64
─────┼────────────────
   1 │       1      1
   2 │ missing      2
   3 │ missing      3

julia> innerjoin(df1, df2, on=:id)
ERROR: ArgumentError: missing values in key columns are not allowed when matchmissing == :error

julia> innerjoin(df1, df2, on=:id, matchmissing=:equal)
3×3 DataFrame
 Row │ id       left   right
     │ Int64?   Int64  Int64
─────┼───────────────────────
   1 │       1      1      1
   2 │ missing      2      2
   3 │ missing      2      3

julia> innerjoin(df1, df2, on=:id, matchmissing=:notequal)
1×3 DataFrame
 Row │ id      left   right
     │ Int64?  Int64  Int64
─────┼──────────────────────
   1 │      1      1      1
```

#### A new syntax for column expansion in transformation functions

Users often store nested data structures in columns of a data frame.
In such cases, a frequent request is to unnest such a column.

Before 1.2.0 release one had to perform this operation like this:

```
julia> df = DataFrame(col=[Dict("a"=>1, "b"=>2), Dict("a"=>3, "b"=>4)])
2×1 DataFrame
 Row │ col
     │ Dict…
─────┼──────────────────────
   1 │ Dict("b"=>2, "a"=>1)
   2 │ Dict("b"=>4, "a"=>3)

julia> transform(df, :col => identity => AsTable)
2×3 DataFrame
 Row │ col                   b      a
     │ Dict…                 Int64  Int64
─────┼────────────────────────────────────
   1 │ Dict("b"=>2, "a"=>1)      2      1
   2 │ Dict("b"=>4, "a"=>3)      4      3
```

Now, a simpler syntax is allowed, that does not require the user to write
`identity` part of the transformation specification (just like in column
renaming syntax), so the following code works

```
julia> transform(df, :col => AsTable)
2×3 DataFrame
 Row │ col                   b      a
     │ Dict…                 Int64  Int64
─────┼────────────────────────────────────
   1 │ Dict("b"=>2, "a"=>1)      2      1
   2 │ Dict("b"=>4, "a"=>3)      4      3
```

and produces the same result.

#### `subset!` now correctly updates passed `GroupedDataFrame`

The `subset!` function was a new addition in 1.0.0 release. Therefore,
given the user feedback, we are adding some polishing touches to it.

Before 1.2.0 passing a `GroupedDataFrame` to `subset!` produced a correct
result, but could potentially corrupt the passed `GroupedDataFrame` (a proper
information about this was given in the documentation; such a design
was chosen to improve performance). However, such a behavior was found to be
error prone. Therefore in 1.2.0 release an efficient algorithm updating not
only the parent data frame but also `GroupedDataFrame` itself was implemented.

Here is an example of the current behavior:

```
julia> using Statistics

julia> df = DataFrame(id=repeat([1, 2], 4), x=1:8)
8×2 DataFrame
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2
   3 │     1      3
   4 │     2      4
   5 │     1      5
   6 │     2      6
   7 │     1      7
   8 │     2      8

julia> gd = groupby(df, :id)
GroupedDataFrame with 2 groups based on key: id
First Group (4 rows): id = 1
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     1      3
   3 │     1      5
   4 │     1      7
⋮
Last Group (4 rows): id = 2
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     2      2
   2 │     2      4
   3 │     2      6
   4 │     2      8

julia> subset!(gd, :x => x -> x .> mean(x)) # pick rows with :x above group mean
4×2 DataFrame
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     1      5
   2 │     2      6
   3 │     1      7
   4 │     2      8

julia> gd
GroupedDataFrame with 2 groups based on key: id
First Group (2 rows): id = 1
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     1      5
   2 │     1      7
⋮
Last Group (2 rows): id = 2
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     2      6
   2 │     2      8
```

In the above operation both `df` and `gd` get properly updated in-place
(previously only `df` was changed, but `gd` was left unchanged, and thus
it was corrupted).

# Deprecated functionality

In the beginning of development of DataFrames.jl the design of `DataFrame`
was very close to a matrix. Over the years the consensus was reached that
we should rather treat it as Tables.jl table. However, the legacy thinking
was still reflected in the design of `hcat` function, which allowed horizontal
concatenation of a data frame with a vector, just like it is allowed for
matrices. Unfortunately this approach conflicts with the fact that currently
many vectors are supporting Tables.jl table interface and when doing `hcat`
users would prefer them to be treated as such.

I think the issue is easiest explained with an example. The Julia session
shown below was started with `--depwarn=yes` flag:

```
julia> using DataFrames

julia> df = DataFrame(col1='a':'c')
3×1 DataFrame
 Row │ col1
     │ Char
─────┼──────
   1 │ a
   2 │ b
   3 │ c

julia> hcat(df, [(x=i, y=10+i) for i in 1:3])
┌ Warning: horizontal concatenation of data frame with a vector is deprecated. Pass DataFrame(x1=x) instead.
│   caller = ip:0x0
└ @ Core :-1
3×2 DataFrame
 Row │ col1  x1
     │ Char  NamedTup…
─────┼───────────────────────
   1 │ a     (x = 1, y = 11)
   2 │ b     (x = 2, y = 12)
   3 │ c     (x = 3, y = 13)
```

As you can see a vector of `NamedTuples`, although it is a Tables.jl table,
is just horizontally concatenated to `df` as a new column with auto generated
name `:x1`.

However, most likely the user expected the following result (but without having
to use `DataFrame` constructor):

```
julia> hcat(df, DataFrame([(x=i, y=10+i) for i in 1:3]))
3×3 DataFrame
 Row │ col1  x      y
     │ Char  Int64  Int64
─────┼────────────────────
   1 │ a         1     11
   2 │ b         2     12
   3 │ c         3     13

```

In order to allow this behavior in the future, as you see above, passing a
vector to `hcat` when the other argument is a data frame is currently deprecated.


# Conclusions

I hope you will enjoy the new features we have shipped in the 1.2.0 release of
the DataFrames.jl.

Apart from the changes discussed above several minor ones, mostly in the areas
of performance, display, and documentation have been made. You can find a more
detailed list of things changed in the [1.2.0 release notes][notes].

Also remember that the NEWS.md file in the project repository is maintained
to give synthetic information of the most important changes introduced in
the releases of DataFrames.jl.

[df]: https://github.com/JuliaData/DataFrames.jl
[notes]: https://github.com/JuliaData/DataFrames.jl/releases/tag/v1.2.0
