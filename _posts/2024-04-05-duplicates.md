---
layout: post
title:  "Deduplication of rows in DataFrames.jl"
date:   2024-04-05 06:12:34 +0200
categories: julialang
---

# Introduction

Deduplication of rows in a table is one of the basic functionalities that
is often needed when working with data frames. Today I discuss the
`allunique`, `nonunique`, `unique`, and `unique!` functions that
are provided by DataFrames.jl and can help you with this task.

The post was written under Julia 1.10.1 and DataFrames.jl 1.6.1.

# Checking if a data frame has duplicate rows

Let us start with discussing how one can check if a data frame has duplicate rows
as this is the simplest check and the functionalities that we discuss here
carry-over to other functions that we discuss later.

First create a simple data frame:

```
julia> using DataFrames

julia> df = DataFrame(x=1:6, y=[1.0, 2.0, 1.0, 2.0, 0.0, -0.0])
6×2 DataFrame
 Row │ x      y
     │ Int64  Float64
─────┼────────────────
   1 │     1      1.0
   2 │     2      2.0
   3 │     3      1.0
   4 │     4      2.0
   5 │     5      0.0
   6 │     6     -0.0
```

By just calling the `allunique` function we can check if whole rows of this data frame are unique:

```
julia> allunique(df)
true
```

In this case we get `true` as indeed all rows are unique. It is guaranteed by the column `"x"` which holds
consecutive integers.

However, we can pass a second positional argument to `allunique`. In this case we can narrow down the list of
checked columns:

```
julia> allunique(df, "y")
false
```

Here we checked uniqueness of only column `"y"`, which contains duplicates, e.g. row 1 and row 3 contain the same value `1.0`,
so we got `false`.

But this is not all. The second positional argument can be any transformation that is supported by the `select` function.
Therefore, for example, we can run:

```
julia> allunique(df, "x" => ByRow(iseven))
false
```

We got `false`, as applying the `iseven` to the `x` column creates duplicates since we have multiple even and odd values in it.
But e.g. we have:

```
julia> allunique(df, "x" => ByRow(x -> x^2))
true
```

Now we get `true` as squares of consecutive integers are unique.

We can pass several transformations as well:

```
julia> allunique(df, ["x" => ByRow(x -> mod(x, 3)), "y" => identity])
true
```

To convince ourselves that the `true` result is correct let us run the `select` operation with the same argument:

```
julia> select(df, ["x" => ByRow(x -> mod(x, 3)), "y" => identity])
6×2 DataFrame
 Row │ x_function  y_identity
     │ Int64       Float64
─────┼────────────────────────
   1 │          1         1.0
   2 │          2         2.0
   3 │          0         1.0
   4 │          1         2.0
   5 │          2         0.0
   6 │          0        -0.0
```

Indeed the rows produced by this operation are unique.

# Finding duplicate rows

To get a vector with indicators of duplicate rows in a data frame use the `nonunique` function. Here are three examples of its usage
(note it also can take a second positional argument just like `allunique`):

```
julia> nonunique(df)
6-element Vector{Bool}:
 0
 0
 0
 0
 0
 0
```

All rows are unique in `df`, as we already know, so we got a vector of `false`s in the call above.

Now the second example:

```
julia> nonunique(df, "x" => ByRow(iseven))
6-element Vector{Bool}:
 0
 0
 1
 1
 1
 1
```

Here we see that we get `true` for all rows for which there was already a duplicate row before. So first two rows get `false` (non-duplicated)
and the following rows have the `true` indicator (as we have already seen an even and an odd number in column `"x"`).

Now look at the last example:

```
julia> nonunique(df, "y")
6-element Vector{Bool}:
 0
 0
 1
 1
 0
 0
```

You might be surprised by the last `false`. The reason is that all the de-duplication functions use `isequal` to compare values for equality,
and `0.0` is not equal to `-0.0` in this comparison:

```
julia> isequal(0.0, -0.0)
false
```

This behavior matches the way how dictionaries work in Julia.

Additionally the `nonunique` has a `keep` keyword argument. It allows us to change the default behavior which rows are marked as duplicate.
If we pass `keep=:last` then the last of the duplicated rows is marked as unique. See for example:

```
julia> nonunique(df, "x" => ByRow(iseven); keep=:last)
6-element Vector{Bool}:
 1
 1
 1
 1
 0
 0
```

We get false in last two rows as `5` and `6` are last even and odd numbers respectively.

The third option is `keep=:noduplicates` in which case only rows that have no duplicates are marked as unique. So we have:

```
julia> nonunique(df, "x" => ByRow(iseven); keep=:noduplicates)
6-element Vector{Bool}:
 1
 1
 1
 1
 1
 1
```

as no row was truly unique, but we have:

```
julia> nonunique(df, "y"; keep=:noduplicates)
6-element Vector{Bool}:
 1
 1
 1
 1
 0
 0
```

as first four rows were duplicated, but rows with `0.0` and `-0.0` are indeed unique.

# Removing duplicate rows from a data frame

The `nonunique` function returns a vector of duplicate indicators. Often we just want to get rid of them from our data frame.
The `unique` and `unique!` functions can be used to perform this operation. They support the same arguments as `nonunique`.
You have three options how you cen get your result:

* using `unique` you get a new data frame by default;
* using `unique` with `view=true` keyword argument passed you get a view of the source data frame with duplicates removed;
* using `unique!` you drop the duplicates in-place from the source data frame.

Let us see how it works. First plain `unique`:

```
julia> unique(df, "y")
4×2 DataFrame
 Row │ x      y
     │ Int64  Float64
─────┼────────────────
   1 │     1      1.0
   2 │     2      2.0
   3 │     5      0.0
   4 │     6     -0.0
```

We got a new data frame. The `df` data frame is unchanged. The second option is a view:

```
julia> unique(df, "y"; view=true)
4×2 SubDataFrame
 Row │ x      y
     │ Int64  Float64
─────┼────────────────
   1 │     1      1.0
   2 │     2      2.0
   3 │     5      0.0
   4 │     6     -0.0
```

Note that still `df` is untouched:

```
julia> df
6×2 DataFrame
 Row │ x      y
     │ Int64  Float64
─────┼────────────────
   1 │     1      1.0
   2 │     2      2.0
   3 │     3      1.0
   4 │     4      2.0
   5 │     5      0.0
   6 │     6     -0.0
```

And finally we can change the `df` data frame in place:

```
julia> unique!(df, "y")
4×2 DataFrame
 Row │ x      y
     │ Int64  Float64
─────┼────────────────
   1 │     1      1.0
   2 │     2      2.0
   3 │     5      0.0
   4 │     6     -0.0

julia> df
4×2 DataFrame
 Row │ x      y
     │ Int64  Float64
─────┼────────────────
   1 │     1      1.0
   2 │     2      2.0
   3 │     5      0.0
   4 │     6     -0.0
```

In this case, as you can see, the `df` data frame was updated.

# Conclusions

I hope that you will find this review of the functionalities of the
`allunique`, `nonunique`, `unique`, and `unique!` functions useful.

As a summary remember that:
* You can determine uniqueness of rows based on transformations of data contained in the source data frame.
* You can decide which rows are marked as duplicate using the `keep` keyword argument.
