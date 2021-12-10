---
layout: post
title:  "News features in DataFrames.jl 1.3: part 1"
date:   2021-12-10 06:28:11 +0200
categories: julialang
---

# Introduction

A few days ago we have released [DataFrames.jl][df] 1.3.0.
In the coming weeks I will discuss new features introduced in this release.
Each post will be devoted to a single topic.

Today I start with performance improvement of aggregation of rows of a data
frame since recently a related interesting question was asked on Slack.

The post was written under Julia 1.7.0 and DataFrames.jl 1.3.0.

# A typical scenario of row aggregation

Let us start with a square data frame that has 10,000 rows and columns.
In the following examples I will omit printing the computed output to reduce
the length of the post.

```
julia> using DataFrames

julia> df = DataFrame(rand(10_000, 10_000), :auto);
```

Assume you want to compute the sum of values in each row of the data frame.

Here is a simple, but inefficient way to do it (I will run the `@time` macro
twice to show the times including compilation and after compilation):

```
julia> @time sum.(eachrow(df));
 29.653202 seconds (784.98 M allocations: 13.199 GiB, 5.93% gc time, 0.42% compilation time)

julia> @time sum.(eachrow(df));
 30.152483 seconds (784.69 M allocations: 13.183 GiB, 5.69% gc time)

```

Now let us use the `select` function to achieve the same:

```
julia> @time select(df, AsTable(:) => ByRow(sum));
  1.424650 seconds (4.62 M allocations: 274.349 MiB, 4.04% gc time, 96.05% compilation time)

julia> @time select(df, AsTable(:) => ByRow(sum));
  0.064406 seconds (19.66 k allocations: 1.140 MiB)
```

The performance improvement is significant.

Here is a list of functions for
which the fast path of aggregation is implemented for the form
`AsTable(cols) => fun [=> destination]` of the DataFrames.jl mini-language:
* `sum`, `ByRow(sum)`, `ByRow(sum∘skipmissing)`;
* `length`, `ByRow(length)`, `ByRow(length∘skipmissing)`;
* `mean`, `ByRow(mean)`, `ByRow(mean∘skipmissing)`;
* `ByRow(var)`, `ByRow(var∘skipmissing)`;
* `ByRow(std)`, `ByRow(std∘skipmissing)`;
* `ByRow(median)`, `ByRow(median∘skipmissing)`;
* `minimum`, `ByRow(minimum)`, `ByRow(minimum∘skipmissing)`;
* `maximum`, `ByRow(maximum)`, `ByRow(maximum∘skipmissing)`;
* `fun∘collect` and `ByRow(f∘collect)` where `f` is any function.

You might be curious about the last form. The optimization is that if you have
any function `f` that takes a vector and makes its reduction it will be efficiently
executed when it is composed with `combine`. Let us use the `extrema` function
as an example:

```
julia> @time extrema.(eachrow(df));
 30.513559 seconds (884.87 M allocations: 14.683 GiB, 6.22% gc time, 0.27% compilation time)

julia> @time extrema.(eachrow(df));
 30.887099 seconds (884.69 M allocations: 14.673 GiB, 6.25% gc time)

julia> @time select(df, AsTable(:) => ByRow(extrema∘collect));
  1.984904 seconds (925.93 k allocations: 812.528 MiB, 1.19% gc time, 18.85% compilation time)

julia> @time select(df, AsTable(:) => ByRow(extrema∘collect));
  1.515307 seconds (49.17 k allocations: 764.989 MiB, 0.78% gc time)
```

Note that the `collect` part is important. If we have not used it the result
would be as follows:

```
julia> @time select(df, AsTable(:) => ByRow(extrema));
 58.041291 seconds (297.92 M allocations: 9.731 GiB, 0.95% gc time, 78.49% compilation time)

julia> @time select(df, AsTable(:) => ByRow(extrema));
 12.495241 seconds (295.00 M allocations: 9.617 GiB, 4.51% gc time)
```

What is the difference? With `collect` we are processing a vector, while without it
a `NamedTuple` is passed to `extrema`. The result gets computed, but, as you
can see in the output of `@time` the compilation time for the first call is huge,
and also after compilation it is faster to work with `collect` version.

# A use-case from practice

The task recently asked on Slack is the following. We have again a data frame
that has 10,000 rows and columns, but this time we have 50% of missing values
randomly scattered in it. What we want to do is to fill missing values in each
row with row means of non-missing values.

Let us first generate the data frame (I start a fresh session again:

```
julia> using DataFrames

julia> using Statistics

julia> df = DataFrame(rand([1.0, missing], 10_000, 10_000), :auto) .* (1:10_000);
```

I first show you how I would have done this operation before DataFrames.jl 1.3
release if I wanted to avoid excessive memory allocation. First I make a vector
of columns of this data frame without copying them:

```
julia> cols = identity.(eachcol(df));
```

Note that I broadcast `identity` to make the element type of the `cols` vector concrete.

Now we compute a vector of of fill values for each row:

```
julia> @time fill_vals = [(mean(skipmissing(v[i] for v in cols))) for i in 1:nrow(df)];
  2.167842 seconds (178.27 k allocations: 8.515 MiB, 5.18% compilation time)

julia> @time fill_vals = [(mean(skipmissing(v[i] for v in cols))) for i in 1:nrow(df)];
  2.212677 seconds (160.38 k allocations: 7.513 MiB, 4.38% compilation time)
```

As you can see this step is quite fast. If we skipped the `identity` call things
would be slower:

```
julia> @time fill_vals = [(mean∘skipmissing)(v[i] for v in eachcol(df)) for i in 1:nrow(df)];
 16.641022 seconds (395.08 M allocations: 6.638 GiB, 6.65% gc time, 0.77% compilation time)

julia> @time fill_vals = [(mean∘skipmissing)(v[i] for v in eachcol(df)) for i in 1:nrow(df)];
 15.949596 seconds (395.07 M allocations: 6.637 GiB, 5.16% gc time, 0.78% compilation time)
```

If we just iterated rows things also would be even slower:

```
julia> @time fill_vals = (mean∘skipmissing).(eachrow(df));
 25.953399 seconds (634.99 M allocations: 10.218 GiB, 5.18% gc time, 0.51% compilation time)

julia> @time fill_vals = (mean∘skipmissing).(eachrow(df));
 25.739528 seconds (634.72 M allocations: 10.203 GiB, 4.91% gc time)
```

Now let us check how fast the `select` machinery we have just learned works:

```
julia> @time fill_vals = select(df, AsTable(:) => ByRow(mean∘skipmissing) => :fill_vals).fill_vals;
  1.773402 seconds (4.42 M allocations: 260.296 MiB, 3.73% gc time, 69.85% compilation time)

julia> @time fill_vals = select(df, AsTable(:) => ByRow(mean∘skipmissing) => :fill_vals).fill_vals;
  0.549109 seconds (19.66 k allocations: 1.217 MiB)
```

As a final step let us check the performance if we kept the data in a matrix instead
(I am not counting the cost of conversion of a data frame to a matrix here):

```
julia> mat = Matrix(df);

julia> @time (mean∘skipmissing).(eachrow(mat));
  1.581645 seconds (7 allocations: 468.906 KiB)

julia> @time (mean∘skipmissing).(eachrow(mat));
  1.547717 seconds (7 allocations: 468.906 KiB)

julia> mat2 = permutedims(mat);

julia> @time (mean∘skipmissing).(eachcol(mat2));
  0.689801 seconds (344.41 k allocations: 19.172 MiB, 19.95% compilation time)

julia> @time (mean∘skipmissing).(eachcol(mat2));
  0.553592 seconds (7 allocations: 468.906 KiB)
```

As you can see the performance of `select` is comparable to the performance
on a matrix when we work on columns (which means we perform the operation using
the fact that Julia uses column major storage of matrices).

To conclude the task we produce a new table with imputed values:

```
julia> @time coalesce.(df, fill_vals);
  0.989387 seconds (370.37 k allocations: 780.629 MiB, 5.10% gc time, 14.19% compilation time)

julia> @time coalesce.(df, fill_vals);
  0.842541 seconds (149.03 k allocations: 768.343 MiB, 4.01% gc time)
```

Before I finish let me comment how you could have filled missing values in
columns with means of columns:

```
julia> @time select(df, names(df) .=> (x -> coalesce.(x, mean(skipmissing(x)))), renamecols=false);
  1.192647 seconds (1.89 M allocations: 861.187 MiB, 7.53% gc time, 40.42% compilation time)

julia> @time select(df, names(df) .=> (x -> coalesce.(x, mean(skipmissing(x)))), renamecols=false);
  0.971668 seconds (1.29 M allocations: 826.755 MiB, 1.91% gc time, 21.57% compilation time)
```

Next week in the post I will discuss how you could have replaced `names(df)`
selector in the last expression with `All()` selector (and how it
is implemented).

# Conclusions

In this post you have learned how to perform fast reductions over rows of
a data frame by using the new features of the mini-language implementation.
I have shown you that the implementation we have is quite efficient. Also since
we support the `f∘collect` composition it is quite extensible to any custom
reduction function that accepts vectors.

[df]: https://github.com/JuliaData/DataFrames.jl
