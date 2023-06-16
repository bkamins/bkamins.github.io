---
layout: post
title:  "Homographs in DataFrames.jl"
date:   2023-06-16 10:43:13 +0200
categories: julialang
---

# Introduction

Last week I [posted about graphs][lastpost], so I thought to post about homographs today.

From your English lessons you probably remember that homographs are words that share
the same written form but have a different meaning.

Starting with Julia 1.9 release we have a homograph in DataFrames.jl. This is the
`stack` function and I will cover it today.

The post was written under Julia 1.9.1 and DataFrames.jl 1.5.0.

# Homographs and multiple dispatch

Julia supports multiple dispatch. This means that you can define specialized methods
for the same function depending on the types of its arguments.

For example if you write `1 + 2` and `1.0 + 2.0` internally different methods of the
`+` are invoked, one working with integers, and the other working with floats.

However, there is one important rule that should be followed. If you add methods
to some function they should perform conceptually similar operations. For example,
`1 + 2` produces `3` and `1.0 + 2.0` produces `3.0`. In both cases an addition was done.

The reason for this rule is that otherwise when you see code like `f(x)` you would not
know what the function `f` does until you know the type of `x`.

Unfortunately, sometimes such situations happen. Let me give you a story of the `stack`
function. It has been defined in DataFrames.jl for many years and is used to perform
wide to long conversion of data frames. Recently Julia maintainers decided that
it makes sense to add the `stack` function to `Base` module and make it combine
a collection of arrays into one larger array.

In such a situation, as DataFrames.jl maintainers we had two options:

1. keep `Base.stack` and `DataFrames.stack` functions separate;
2. make `stack` from DataFrames.jl add a method to `Base.stack`.

In general the first option is preferable. `Base.stack` and `DataFrames.stack`
do different things so they should be separate functions. However, there is one
problem with this approach. All legacy code that used `stack` from DataFrames.jl
would stop working and users would need to write `DataFrames.stack` instead.
This is something we did not want so we decided to go for option 2, that is,
add a method for `Base.stack` that would handle data frames. The reason why we
decided for this is that there is very low risk of confusion since `stack` from
DataFrames.jl always requires an `AbstractDataFrame` as its first argument.
You can see it here:

```
julia> using DataFrames

julia> methods(stack)
# 6 methods for generic function "stack" from Base:
 [1] stack(df::AbstractDataFrame)
     @ DataFrames ~\.julia\packages\DataFrames\LteEl\src\abstractdataframe\reshape.jl:136
 [2] stack(df::AbstractDataFrame, measure_vars)
     @ DataFrames ~\.julia\packages\DataFrames\LteEl\src\abstractdataframe\reshape.jl:136
 [3] stack(df::AbstractDataFrame, measure_vars, id_vars; variable_name, value_name, view, variable_eltype)
     @ DataFrames ~\.julia\packages\DataFrames\LteEl\src\abstractdataframe\reshape.jl:136
 [4] stack(iter; dims)
     @ abstractarray.jl:2743
 [5] stack(f, iter; dims)
     @ abstractarray.jl:2772
 [6] stack(f, xs, yzs...; dims)
     @ abstractarray.jl:2773
```

At the same time standard `Base.stack` does not work with data frames at all.

OK, enough theory. Let us have a look at `stack` from `Base` and from DataFrames.jl
in action.

# Combining collections of arrays

I will concentrate here on the simplest (and most often needed scenarios).
Assume you have a vector of vectors of equal length:

```
julia> x = [1:2, 3:4, 5:6]
3-element Vector{UnitRange{Int64}}:
 1:2
 3:4
 5:6
```

We could want to turn it into a matrix. There are two ways you could want to do it.
The first is to put these vectors as rows in the produced matrix, like this:

```
julia> stack(x)
2×3 Matrix{Int64}:
 1  3  5
 2  4  6
```

The second is to put them as columns (this is often needed and in the past I always
used `permutedims` to get this, which was a bit cumbersome):

```
julia> stack(x, dims=1)
3×2 Matrix{Int64}:
 1  2
 3  4
 5  6
```

The third commonly used pattern is when you want to apply a function to a vector of
values and this function returns another vector or e.g. a tuple. Let us have a look
at an example:

```
julia> using Statistics

julia> f(x) = [sum(x), mean(x)]
f (generic function with 1 method)

julia> f.(x)
3-element Vector{Vector{Float64}}:
 [3.0, 1.5]
 [7.0, 3.5]
 [11.0, 5.5]
```

This is the traditional, broadcasting way, to apply the function to such a vector.
However, often you want the result in a flat matrix. Now you can do:

```
julia> stack(f.(x))
2×3 Matrix{Float64}:
 3.0  7.0  11.0
 1.5  3.5   5.5
```

which can be done even simpler by just writing:

```
julia> stack(f, x)
2×3 Matrix{Float64}:
 3.0  7.0  11.0
 1.5  3.5   5.5
```

In summary, `Base.stack` is a super nice little utility that comes handy very often
if you work with arrays.

# Transforming data from wide to long format

In DataFrames.jl the `stack` transforms data from wide to long format.
Assume you have the following input data frame:

```
julia> df = DataFrame(year=[2020, 2021], Spring=1:2, Summer=3:4, Autumn=5:6, Winter=7:8)
2×5 DataFrame
 Row │ year   Spring  Summer  Autumn  Winter
     │ Int64  Int64   Int64   Int64   Int64
─────┼───────────────────────────────────────
   1 │  2020       1       3       5       7
   2 │  2021       2       4       6       8
```

It is in wide format. For each year you have four columns for each season holding some values.
Assume we want instead a narrow data frame with year-season combinations and one column with values.
With `DataFrames.stack` it is enough to pass a data frame and specify which columns hold the values:

```
julia> stack(df, Not(:year))
8×3 DataFrame
 Row │ year   variable  value
     │ Int64  String    Int64
─────┼────────────────────────
   1 │  2020  Spring        1
   2 │  2021  Spring        2
   3 │  2020  Summer        3
   4 │  2021  Summer        4
   5 │  2020  Autumn        5
   6 │  2021  Autumn        6
   7 │  2020  Winter        7
   8 │  2021  Winter        8
```

Or, if you want to be more fancy you can e.g. change the generated column names:

```
julia> stack(df, Not(:year), variable_name="season", value_name="number")
8×3 DataFrame
 Row │ year   season  number
     │ Int64  String  Int64
─────┼───────────────────────
   1 │  2020  Spring       1
   2 │  2021  Spring       2
   3 │  2020  Summer       3
   4 │  2021  Summer       4
   5 │  2020  Autumn       5
   6 │  2021  Autumn       6
   7 │  2020  Winter       7
   8 │  2021  Winter       8
```

# Conclusions

In this post I have given the basic examples of `Base.stack` and `DataFrames.stack` usage.
I recommend you to have a look at their documentation for more complete information.
However, the point is that both functions are quite useful in daily data wrangling so it is
useful to know them.

Additionally, I wanted to highlight some general considerations of package design in Julia
and the challenges that maintainers face. In the specific example of `stack` we decided to break
the *all methods of a function should do a similar thing* rule in favor of user convenience and making
sure that we keep the legacy DataFrames.jl code working.

[lastpost]: https://bkamins.github.io/julialang/2023/06/09/graphs.html
