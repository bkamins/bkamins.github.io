---
layout: post
title:  "DataFrames.jl survey: selecting columns of a data frame based on their values"
date:   2023-08-18 06:35:51 +0200
categories: julialang
---

# Introduction

Today I want to make a user survey about a future direction
of development of DataFrames.jl.

Most commonly users want to select columns of a data frame
using their names and this is the operation that has an
extensive support in DataFrames.jl.
However, in some cases one might need to perform such a
selection conditional on a value stored in a column.

I want to ask if we should add a special selector allowing for
picking columns of a data frame based on their values.
The question is given in the conclusions section.
However, before we get to it let us briefly review what is
currently supported.

The post was written using Julia 1.9.2 and DataFrames.jl 1.6.1.

# Column selection using column names

First create a simple data frame on which we are going
to perform the column selection examples:

```
julia> using DataFrames

julia> df = DataFrame(a1 = [1, 2],
                      a2=[1, missing],
                      b1=Any[3, missing],
                      b2=Any[3, 4])
2×4 DataFrame
 Row │ a1     a2       b1       b2
     │ Int64  Int64?   Any      Any
─────┼──────────────────────────────
   1 │     1        1  3        3
   2 │     2  missing  missing  4
```

Now assume we want to pick columns whose name starts with `"b"`:

```
julia> select(df, Cols(startswith("b")))
2×2 DataFrame
 Row │ b1       b2
     │ Any      Any
─────┼──────────────
   1 │ 3        3
   2 │ missing  4
```

As you can see, if you pass a function returning a `Bool`
(such a function is often called a predicate) to `Cols`
selector you get columns whose names math the condition
defined by this function.

Today I want to focus on cases when you specify the condition
using a function. However, let me mention that there
are other ways to perform selection we discussed above. For example
we could use a regular expression:

```
julia> select(df, Cols(r"^b"))
2×2 DataFrame
 Row │ b1       b2
     │ Any      Any
─────┼──────────────
   1 │ 3        3
   2 │ missing  4
```

In general, in DataFrames.jl you currently have the following ways to
select columns (Warning! The list is long.):

* a symbol, string, or integer;
* vector of symbols, strings, integers, or bools;
* regular expression;
* `All`, `Between`, `Cols`, `Not`, and `:` selectors.

# Column selection using column values

What if we wanted to select the columns using their values.
For example, assume that we want to pick columns that contain
`missing` value. In this case the easiest way to do it is to
use the `eachcol(df)` iterator over columns of our data frame:

```
julia> select(df, any.(ismissing, eachcol(df)))
2×2 DataFrame
 Row │ a2       b1
     │ Int64?   Any
─────┼──────────────────
   1 │       1  3
   2 │ missing  missing
```

Notice that the `any.(ismissing, eachcol(df))` condition
iterates all columns of `df` and for each of them returns `true`
if they contain any missing value (and `false` otherwise):

```
julia> any.(ismissing, eachcol(df))
4-element BitVector:
 0
 1
 1
 0
```

An alternative, similar condition would be to select all columns
that allow for storing `missing` value (without requiring that
they actually have it stored in them). For this we need to use
the `eltype` function on columns:

```
julia> select(df, Missing .<: eltype.(eachcol(df)))
2×3 DataFrame
 Row │ a2       b1       b2
     │ Int64?   Any      Any
─────┼───────────────────────
   1 │       1  3        3
   2 │ missing  missing  4
```

Note that the difference is column `:b2`, which does not
contain `missing` values, but could contain them since its
element type is `Any`.

# Column selection using column names and values

Now, what if we wanted to perform column selection based on both
their names and values?

The general pattern uses `pairs(eachcol(df))` which iterates
pairs of column names and values:

```
julia> pairs(eachcol(df))
Iterators.Pairs(::DataFrames.DataFrameColumns{DataFrame}, ::Vector{Symbol})(...):
  :a1 => [1, 2]
  :a2 => Union{Missing, Int64}[1, missing]
  :b1 => Any[3, missing]
  :b2 => Any[3, 4]
```

So for example if we wanted to pick columns that contain `missing` values
and start with `"a"` we can write:

```
julia> select(df, [startswith(string(n), "a") && any(ismissing, c)
                   for (n,c) in pairs(eachcol(df))])
2×1 DataFrame
 Row │ a2
     │ Int64?
─────┼─────────
   1 │       1
   2 │ missing
```

This pattern is fully general, but slightly verbose, especially column names
returned by `pairs` are `Symbols`. The same condition
can be written more naturally as follows:

```
julia> select(df, Cols(startswith("a"),
                       any.(ismissing, eachcol(df));
                       operator=intersect))
2×1 DataFrame
 Row │ a2
     │ Int64?
─────┼─────────
   1 │       1
   2 │ missing
```

Here we take advantage of the fact that our condition was a conjunction
and `Cols` selector accepts the `operator` keyword argument with allows
to get an intersection of two selectors.

If we wanted to select columns that meet
at least one of the conditions this would be even simpler:

```
julia> select(df, Cols(startswith("a"),
                       any.(ismissing, eachcol(df))))
2×3 DataFrame
 Row │ a1     a2       b1
     │ Int64  Int64?   Any
─────┼─────────────────────────
   1 │     1        1  3
   2 │     2  missing  missing
```

Note that by default `Cols` select columns that are a union of selectors
passed to it.

# Conclusions

I hope the examples I presented today will be useful when you work with
DataFrames.jl.

You might ask why, when performing column selection
based on their values one needs to invoke `eachcol(df)`. This is indeed
a bit verbose. However, we decided that `Cols(predicate)`, which is
shorter to write should apply the `predicate` function to column names
as this is a more common operation. And if user wants to apply
the `predicate` function to column values (which is a less frequent case)
writing `predicate.(eachcol(df))` is readable and easy enough.

If we added a built-in way for selection of columns by value
it would mean that instead of writing
`predicate.(eachcol(df))` you would write something like
`Vals(predicate)` (the `Vals` is an example name - the choice of name can
be done later).

The benefits of having it are:

* It is shorter.
* We do not have a redundance of having to pass the `df` data frame in the expression.

Here are the cons of adding it:

* It makes the list of things to learn longer (and the list is already quite long).
* It is ambiguous how the `Vals(predicate)` should be interpreted if it were
  used in the context of `GroupedDataFrame` as the question would be how should we
  treat the groups (so most likely it should be only allowed for `AbstractDataFrame`).
* It would require a change of internal memory layout of `DataFrame` object
  (which means that the next release of DataFrames.jl would be incompatible on
  binary level with the current release so serialization/deserialization would
  not work cross-versions).

Now comes the question:

> Should we add a new special selector that would allow picking
> columns based on their values?

If you have an opinion please vote or comment in [this issue][issue] on GitHub.

[issue]: https://github.com/JuliaData/DataFrames.jl/issues/3034#issuecomment-1683032247
