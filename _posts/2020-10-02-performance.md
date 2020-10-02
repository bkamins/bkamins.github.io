---
layout: post
title:  "Julia Base is DataFrames.jl best friend"
date:   2020-10-02 11:22:31 +0200
categories: julialang
---

# Introduction

Recently I received [a very interesting question][question] about the performance
of operations on `GroupedDataFrame` in DataFrames.jl.

In this post I want to share some comments related to the performance of DataFrames.jl.

The TLDR summary is:
if you want top performance fall back to low-level Julia
code (and DataFrames.jl exposes API that allows you to follow that path).

In this post I am using Julia 1.5.2 and DataFrames.jl 0.21.7.

# The question and a direct answer

Assume you have a large data frame `df` that has two columns: `:id` and `:v`
(in practice also many more columns that are irrelevant so I skip them).
For each unique value of `:id` you want to keep only one row that corresponds
to the smallest value of `:v` in the group defined by column `:id`.

We start with generating the example `DataFrame` and grouping it by `:id` column:

```
julia> using DataFrames, Random

julia> Random.seed!(1234);

julia> df = DataFrame(id=rand(1:10^6, 10^7), v=rand(10^7))
10000000×2 DataFrame
│ Row      │ id     │ v        │
│          │ Int64  │ Float64  │
├──────────┼────────┼──────────┤
│ 1        │ 99353  │ 0.954945 │
│ 2        │ 371388 │ 0.103541 │
⋮
│ 9999998  │ 71163  │ 0.37311  │
│ 9999999  │ 529940 │ 0.856262 │
│ 10000000 │ 845766 │ 0.401057 │

julia> gdf = groupby(df, :id);

julia> @time gdf = groupby(df, :id)
  1.326534 seconds (30 allocations: 280.590 MiB)
GroupedDataFrame with 999963 groups based on key: id
First Group (13 rows): id = 99353
│ Row │ id    │ v         │
│     │ Int64 │ Float64   │
├─────┼───────┼───────────┤
│ 1   │ 99353 │ 0.954945  │
│ 2   │ 99353 │ 0.584677  │
⋮
│ 11  │ 99353 │ 0.838052  │
│ 12  │ 99353 │ 0.0773429 │
│ 13  │ 99353 │ 0.774979  │
⋮
Last Group (1 row): id = 502811
│ Row │ id     │ v        │
│     │ Int64  │ Float64  │
├─────┼────────┼──────────┤
│ 1   │ 502811 │ 0.953105 │
```

(note that I run here and below all commands twice to exclude compilation time
and allocations from what is reported by `@time` macro)

A direct approach to solve this problem would be the following (here I store the
result in `res1` variable to compare it to other solutions at the end of this post):

```
julia> res1 = combine(sdf -> first(sort(sdf)), gdf);

julia> @time combine(sdf -> first(sort(sdf)), gdf)
 13.587720 seconds (296.78 M allocations: 12.672 GiB, 9.73% gc time)
999963×2 DataFrame
│ Row    │ id     │ v         │
│        │ Int64  │ Float64   │
├────────┼────────┼───────────┤
│ 1      │ 99353  │ 0.0436469 │
│ 2      │ 371388 │ 0.0203823 │
⋮
│ 999961 │ 167495 │ 0.228266  │
│ 999962 │ 527369 │ 0.0525908 │
│ 999963 │ 502811 │ 0.953105  │
```

We see that the operation is quite slow and massively allocates. The problem is
that `sort(sdf)` allocates a fresh small `DataFrame` for each group. This is
inefficient.

Let us think then if we can improve the performance of this code.

# Pre-sorting

The first option to consider is pre-sorting of `df` like this:

```
julia> res2 = combine(first, groupby(sort(df, :v), :id));

julia> @time combine(first, groupby(sort(df, :v), :id))
  7.166366 seconds (14.03 M allocations: 1.811 GiB, 2.85% gc time)
999963×2 DataFrame
│ Row    │ id     │ v          │
│        │ Int64  │ Float64    │
├────────┼────────┼────────────┤
│ 1      │ 533731 │ 1.27094e-7 │
│ 2      │ 709049 │ 1.29805e-7 │
⋮
│ 999961 │ 803454 │ 0.991274   │
│ 999962 │ 107403 │ 0.99218    │
│ 999963 │ 925877 │ 0.994074   │
```

We already see much improvement. However, this solution has a problem that
it still requires sorting, which in the chain `sort`, `groupby`, `combine`
is the slowest step.

# Using `argmin`

Actually we see that we do not need to sort our data to get the result. It
is enough to find the minimum row with respect to column `:v`. Here is how
one can do it:

```
julia> res3 = combine(sdf -> sdf[argmin(sdf.v), :], gdf);

julia> @time combine(sdf -> sdf[argmin(sdf.v), :], gdf)
  1.581478 seconds (15.48 M allocations: 593.183 MiB, 4.29% gc time)
999963×2 DataFrame
│ Row    │ id     │ v         │
│        │ Int64  │ Float64   │
├────────┼────────┼───────────┤
│ 1      │ 99353  │ 0.0436469 │
│ 2      │ 371388 │ 0.0203823 │
⋮
│ 999961 │ 167495 │ 0.228266  │
│ 999962 │ 527369 │ 0.0525908 │
│ 999963 │ 502811 │ 0.953105  │
```

Now we are much faster. We can do better if we start using a low-level API
of DataFrames.jl and use Julia Base functionality.

# Going for Julia Base

I will present two solutions here that do the job faster than the last
example of `combine`. The first one uses the fact that we can iterate
`GroupedDataFrame` and `parentindices` function returns us a tuple of indices
into the parent of the passed object. Here is the approach that can be used:

```
julia> res4 = df[[parentindices(sdf)[1][argmin(sdf.v)] for sdf in gdf], :];

julia> @time df[[parentindices(sdf)[1][argmin(sdf.v)] for sdf in gdf], :]
  0.990385 seconds (5.19 M allocations: 325.875 MiB, 4.73% gc time)
999963×2 DataFrame
│ Row    │ id     │ v         │
│        │ Int64  │ Float64   │
├────────┼────────┼───────────┤
│ 1      │ 99353  │ 0.0436469 │
│ 2      │ 371388 │ 0.0203823 │
⋮
│ 999961 │ 167495 │ 0.228266  │
│ 999962 │ 527369 │ 0.0525908 │
│ 999963 │ 502811 │ 0.953105  │
```

And we see that we are somewhat faster. We can go much faster by using the
`groupindices` function that gives us a group number for each row of the `df`
data frame. The cost is that the solution is a bit more verbose:

```
julia> function getfirst(gdf, v)
           loc = zeros(Int, length(gdf))
           best = Vector{eltype(v)}(undef, length(gdf))
           for (i, (ind, vi)) in enumerate(zip(groupindices(gdf), v))
               if !ismissing(ind) && (loc[ind] == 0 || vi < best[ind])
                   best[ind] = vi
                   loc[ind] = i
               end
           end
           return parent(gdf)[loc, :]
       end
getfirst (generic function with 1 method)

julia> res5 = getfirst(gdf, df.v);

julia> @time getfirst(gdf, df.v)
  0.359210 seconds (20 allocations: 116.349 MiB)
999963×2 DataFrame
│ Row    │ id     │ v         │
│        │ Int64  │ Float64   │
├────────┼────────┼───────────┤
│ 1      │ 99353  │ 0.0436469 │
│ 2      │ 371388 │ 0.0203823 │
⋮
│ 999961 │ 167495 │ 0.228266  │
│ 999962 │ 527369 │ 0.0525908 │
│ 999963 │ 502811 │ 0.953105  │
```

We could go even faster (at least twice in my tests), but let me stop here and
leave the potential improvements as an exercise.

Additionally, it is worth to mention that the vector `groupindices(gdf)` has a
value equal to `missing` if some row from `df` is not mapped to any group in
`gdf` (note that `GroupedDataFrame` can be indexed-into).

# Conclusions

Before we conclude let us check if the five solutions give us the same
results:

```
julia> length(unique(sort.([res1, res2, res3, res4, res5])))
1
```

Indeed they do. I had to use `sort` here, as `res2` has a different row order
than the rest of data frames holding the results.

What are the take aways from these examples:
* if you think about avoiding allocations and doing unnecessary work (sorting
  in our case) you can get a reasonably fast result using high-level API of
  DataFrames.jl;
* if you want to go really fast there is always an option to fall-back to
  custom code written in Julia Base and DataFrames.jl has a low-level API
  that allows you to efficiently do this.

The reason why high-level API is slower is that it handles dozens of possible
use cases under one function and going low-level allows you do do only the thing
that needs to be done in your use case.

In general, it is worth to highlight here that, as opposed to e.g. R, `DataFrame`
in Julia is just one of the many data structures you can use. The high-level API
DataFrames.jl provides will be fast enough in most of the cases (at the benefit
of being very flexible). However, if things are too slow just go low-level,
which is a natural thing to do in Julia.

[question]: https://github.com/bkamins/Julia-DataFrames-Tutorial/issues/27
