---
layout: post
title:  "Row aggregation in DataFrames.jl"
date:   2020-09-20 07:11:13 +0200
categories: julialang
---

# Introduction

In this post I want to highlight performance considerations when doing row
aggregation using DataFrames.jl as this topic has been quite hot recently.

The post is tested under Julia 1.5 and DataFrames.jl 0.21.

# The problem

We first load the packages that we will need for this Julia session:
```
julia> using Statistics, DataFrames
```

Consider the following `DataFrame`:

```
julia> df = DataFrame([1:10^6 for _ in 1:32], :auto)
1000000×32 DataFrame. Omitted printing of 26 columns
│ Row     │ x1      │ x2      │ x3      │ x4      │ x5      │ x6      │
│         │ Int64   │ Int64   │ Int64   │ Int64   │ Int64   │ Int64   │
├─────────┼─────────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ 1       │ 1       │ 1       │ 1       │ 1       │ 1       │ 1       │
│ 2       │ 2       │ 2       │ 2       │ 2       │ 2       │ 2       │
│ 3       │ 3       │ 3       │ 3       │ 3       │ 3       │ 3       │
│ 4       │ 4       │ 4       │ 4       │ 4       │ 4       │ 4       │
│ 5       │ 5       │ 5       │ 5       │ 5       │ 5       │ 5       │
│ 6       │ 6       │ 6       │ 6       │ 6       │ 6       │ 6       │
│ 7       │ 7       │ 7       │ 7       │ 7       │ 7       │ 7       │
⋮
│ 999993  │ 999993  │ 999993  │ 999993  │ 999993  │ 999993  │ 999993  │
│ 999994  │ 999994  │ 999994  │ 999994  │ 999994  │ 999994  │ 999994  │
│ 999995  │ 999995  │ 999995  │ 999995  │ 999995  │ 999995  │ 999995  │
│ 999996  │ 999996  │ 999996  │ 999996  │ 999996  │ 999996  │ 999996  │
│ 999997  │ 999997  │ 999997  │ 999997  │ 999997  │ 999997  │ 999997  │
│ 999998  │ 999998  │ 999998  │ 999998  │ 999998  │ 999998  │ 999998  │
│ 999999  │ 999999  │ 999999  │ 999999  │ 999999  │ 999999  │ 999999  │
│ 1000000 │ 1000000 │ 1000000 │ 1000000 │ 1000000 │ 1000000 │ 1000000 │
```

It has one million rows and 32 columns. We will want to calculate row means
of this data frame.

Here is a most natural attempt to perform this task (I show all timings without
compilation time):

```
julia> @time mean.(eachrow(df))
  6.449878 seconds (225.07 M allocations: 3.843 GiB, 5.42% gc time)
1000000-element Array{Float64,1}:
      1.0
      2.0
      3.0
      4.0
      5.0
      6.0
      7.0
      8.0
      ⋮
 999993.0
 999994.0
 999995.0
 999996.0
 999997.0
 999998.0
 999999.0
      1.0e6
```

which is unfortunately slow. It is much faster to convert this data frame
into a `Matrix` and aggregate it like this:
```
julia> @time mean(Matrix(df), dims=2)
  0.156253 seconds (48 allocations: 251.772 MiB)
1000000×1 Array{Float64,2}:
      1.0
      2.0
      3.0
      4.0
      5.0
      6.0
      7.0
      8.0
      ⋮
 999993.0
 999994.0
 999995.0
 999996.0
 999997.0
 999998.0
 999999.0
      1.0e6
```

Note that the biggest cost here comes from the conversion to a `Matrix`:
```
julia> @time m = Matrix(df); @time mean(m, dims=2);
  0.137838 seconds (36 allocations: 244.142 MiB, 15.30% gc time)
  0.043450 seconds (12 allocations: 7.630 MiB)
```

If we want to avoid allocating one big object, but we are fine with many small
allocations at some cost to performance we can do:

```
julia> @time mean.(Tables.namedtupleiterator(df));
  0.250234 seconds (30 allocations: 251.774 MiB, 27.58% gc time)
```

Now let us switch to examples using the `source => fun => destination`
mini-language that is supported in `select` etc.:

```
julia> @time select(df, AsTable(:) => ByRow(mean) => :mean);
  0.183326 seconds (133 allocations: 251.782 MiB)
```
which is not very bad given the examples above.

Note that in the above example the `AsTable` wrapper passes `NamedTuple`s created
from the rows of our data frame to `mean`. In some scenarios it might be
problematic, as the methods have then to be recompiled as the set of columns changes
(even if their element types do not change and are homogeneous the `NamedTuple`
type changes):

```
julia> @time select(df, AsTable(:) => ByRow(mean) => :mean); # this was compiled earlier
  0.185655 seconds (133 allocations: 251.782 MiB)

julia> @time select(df, AsTable(1:31) => ByRow(mean) => :mean);
  0.545427 seconds (653.74 k allocations: 279.140 MiB, 11.39% gc time)

julia> @time select(df, AsTable(2:30) => ByRow(mean) => :mean);
  0.431746 seconds (609.41 k allocations: 261.616 MiB)
```

There is a way to avoid passing column names to a function in the transformation
mini language - you just drop `AsTable` and get positional arguments. In this case
it is not only less convenient to write but also slower:

```
julia> @time select(df, (:) => ByRow((x...) -> mean(x)) => :mean);
  0.749447 seconds (2.20 M allocations: 102.608 MiB)
```

So this is not an ideal solution. Clearly we need another way in DataFrames.jl
to signal passing a homogeneous part of the row to the aggregation function.

In some cases, the *positional arguments passing* API also hits a limitation
that Julia will give up on specializing the function that is called.

First we make a bit wider but shorter data frame:

```
julia> df = DataFrame([1:10^5 for _ in 1:64], :auto)
100000×64 DataFrame. Omitted printing of 57 columns
│ Row    │ x1     │ x2     │ x3     │ x4     │ x5     │ x6     │ x7     │
│        │ Int64  │ Int64  │ Int64  │ Int64  │ Int64  │ Int64  │ Int64  │
├────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┤
│ 1      │ 1      │ 1      │ 1      │ 1      │ 1      │ 1      │ 1      │
│ 2      │ 2      │ 2      │ 2      │ 2      │ 2      │ 2      │ 2      │
│ 3      │ 3      │ 3      │ 3      │ 3      │ 3      │ 3      │ 3      │
│ 4      │ 4      │ 4      │ 4      │ 4      │ 4      │ 4      │ 4      │
│ 5      │ 5      │ 5      │ 5      │ 5      │ 5      │ 5      │ 5      │
⋮
│ 99995  │ 99995  │ 99995  │ 99995  │ 99995  │ 99995  │ 99995  │ 99995  │
│ 99996  │ 99996  │ 99996  │ 99996  │ 99996  │ 99996  │ 99996  │ 99996  │
│ 99997  │ 99997  │ 99997  │ 99997  │ 99997  │ 99997  │ 99997  │ 99997  │
│ 99998  │ 99998  │ 99998  │ 99998  │ 99998  │ 99998  │ 99998  │ 99998  │
│ 99999  │ 99999  │ 99999  │ 99999  │ 99999  │ 99999  │ 99999  │ 99999  │
│ 100000 │ 100000 │ 100000 │ 100000 │ 100000 │ 100000 │ 100000 │ 100000 │
```

Now consider this example (now showing the timings twice to also include
compilation cost information).
We consider both column-wise and row-wise operations:

```
julia> @time select(df, (:) => (+) => :sum);
 18.861715 seconds (343.03 M allocations: 12.386 GiB, 9.24% gc time)

julia> @time select(df, (:) => (+) => :sum);
 15.378663 seconds (335.23 M allocations: 12.105 GiB, 9.41% gc time)

julia> @time select(df, AsTable(:) => sum => :sum);
  0.141843 seconds (247.37 k allocations: 61.651 MiB)

julia> @time select(df, AsTable(:) => sum => :sum);
  0.025270 seconds (292 allocations: 48.092 MiB)

julia> @time select(df, (:) => ByRow(+) => :sum);
 15.761646 seconds (336.07 M allocations: 12.136 GiB, 9.50% gc time)

julia> @time select(df, (:) => ByRow(+) => :sum);
 15.397908 seconds (335.23 M allocations: 12.105 GiB, 9.24% gc time)

julia> @time select(df, AsTable(:) => ByRow(sum) => :sum);
  0.402675 seconds (609.12 k allocations: 81.987 MiB)

julia> @time select(df, AsTable(:) => ByRow(sum) => :sum);
  0.043448 seconds (176 allocations: 49.615 MiB)
```

What you can see is that the positional argument API is struggling very much
when it has 64 columns to process. Let us reduce this number:

```
julia> @time select(df, 1:32 => (+) => :sum);
  0.004199 seconds (130 allocations: 791.703 KiB)

julia> @time select(df, 1:33 => (+) => :sum);
  0.539459 seconds (13.65 M allocations: 407.434 MiB, 15.47% gc time)

julia> @time select(df, 1:32 => ByRow(+) => :sum);
  0.004287 seconds (127 allocations: 792.141 KiB)

julia> @time select(df, 1:33 => ByRow(+) => :sum);
  0.543523 seconds (13.65 M allocations: 407.433 MiB, 15.72% gc time)
```

As we can see something very bad happens when we switch from 32 to 33 positional
arguments. Actually what is going on is that the compiler gives up doing
type inference, and just treats the values as `Any`.

Here is a simpler example with a small function where the same happens when we
switch from 17 to 18 positional arguments:

```
julia> f(x...) = .+(x...)
f (generic function with 1 method)

julia> @code_warntype f(1:17...)
Variables
  #self#::Core.Compiler.Const(f, false)
  x::NTuple{17,Int64}

Body::Int64
1 ─ %1 = Core.tuple(Main.:+)::Core.Compiler.Const((+,), false)
│   %2 = Core._apply_iterate(Base.iterate, Base.broadcasted, %1, x)::Base.Broadcast.Broadcasted{Base.Broadcast.DefaultArrayStyle{0},Nothing,typeof(+),NTuple{17,Int64}}
│   %3 = Base.materialize(%2)::Int64
└──      return %3

julia> @code_warntype f(1:18...)
Variables
  #self#::Core.Compiler.Const(f, false)
  x::NTuple{18,Int64}

Body::Any
1 ─ %1 = Core.tuple(Main.:+)::Core.Compiler.Const((+,), false)
│   %2 = Core._apply_iterate(Base.iterate, Base.broadcasted, %1, x)::Any
│   %3 = Base.materialize(%2)::Any
└──      return %3
```

It is not always easy to determine when the compiler will give up doing type
inference, but a safe estimate is that for 16 or less positional arguments
all should be good.

# Concluding remarks

In conclusion the take aways are:
* In principle DataFrames.jl is a column oriented storage so working across
  its rows can be expected to be slower than in other data types that are
  optimized for this use case.
* If you have homogeneous data you want to aggregate row-wise often doing a
  conversion first (e.g. to a `Matrix` as in our example) and then performing
  the operation will be a sound choice.
* Using positional arguments API like `(:) => (+)` in our examples has a
  limitation that it does not handle very well huge number of columns and should
  be avoided. This API is intended mainly for a few positional arguments (which
  is a common use case in practice).
* You can expect `AsTable` API to be reasonably fast, but note that you might
  have to pay a significant compilation cost at each call if the schema of what
  you are aggregating is changing.
* This indicates that we need yet a new wrapper to support fast row-wise
  aggregation that would not have a limitation the `AsTable` API has now.

If you want to be updated on how this topic is resolved then you can
track [this issue][issue]. Most likely we will add some new wrappers
(similar to `AsTable`, but which will be efficient when the aggregation is performed
across columns of homogeneous types and column name information is not required).

Update on 2020-04-07: In current release of DataFrames.jl (1.3.2 as of time
of writing this note) performance of row aggregation has been improved, see
[this PR][pr] for details.

[issue]: https://github.com/JuliaData/DataFrames.jl/issues/2440
[pr]: https://github.com/JuliaData/DataFrames.jl/pull/2869
