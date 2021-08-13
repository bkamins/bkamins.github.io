---
layout: post
title:  "The state of multiple threading in DataFrames.jl"
date:   2021-08-13 08:04:43 +0200
categories: julialang
---

# Introduction

One of the most frequent advanced questions related to [DataFrames.jl][df]
I get is about its support for multi-threaded operations.

In order to summarize the state of this topic in the manual of the
[DataFrames.jl][df] we will soon merge the [#2823 PR][pr] that hopefully
clarifies this issue. This post is meant to accompany this PR with some examples
showing the performance of selected operations when multiple threads are used.

The post is divided into two sections. In the first one I describe cases in
which multi threading is used automatically. In the second I show the most
simple pattern of how the user can invoke multiple threads manually when using
the DataFrames.jl package.

In all comparisons I will show the timing of exactly the same operations when
one uses 1 thread vs 4 threads as this is something that I can comfortably test
on my laptop.

The post was tested under Julia 1.6.1, DataFrames.jl 1.2.2,
and BenchmarkTools.jl 1.1.1.

# Automatic multi threading

In general, [DataFrames.jl][df] uses multiple threading automatically in selected
operations, provided that the data that we work with is large enough (for small
data usually the cost of managing multiple threads is too large to bring benefits).

In the comparison I show all scenarios that are listed in the [#2823 PR][pr] on
a data frame that is large enough.

First start with a single thread (I started Julia just by writing `julia` in my
terminal):

```
julia> Threads.nthreads()
1

julia> using DataFrames

julia> using BenchmarkTools

julia> df = DataFrame(reshape(1:50_000_000, :, 25), :auto);

julia> df.id1 = repeat(1:10_000, 200);

julia> df.id2 = string.(df.id1);

julia> summary(df)
"2000000×27 DataFrame"

julia> @btime copy($df); # constructor
  36.997 ms (88 allocations: 411.99 MiB)

julia> @btime $df[1:2:end, :]; # indexing
  45.684 ms (117 allocations: 206.00 MiB)

julia> @btime groupby($df, :id1); # grouping on refs
  3.537 ms (109 allocations: 15.27 MiB)

julia> @btime groupby($df, :id2); # grouping on hash
  48.679 ms (61 allocations: 62.52 MiB)

julia> @btime innerjoin($df, $df, on=:x1, makeunique=true); # joining
  270.209 ms (937 allocations: 839.28 MiB)

julia> gdf = groupby(df, :id1);

julia> @btime select($gdf, names($df, r"x") .=> x -> x .^ 2); # many transformations
  965.715 ms (1740851 allocations: 2.01 GiB)

julia> function f(x)
           s = zero(eltype(x))
           for _ in 1:1000, v in x
               s += v
           end
           return s
       end
f (generic function with 1 method)

julia> @btime combine($gdf, :x1 => f); # complex reduction
  1.513 s (339 allocations: 252.42 KiB)
```

And now the same exampls using 4 threads (start Julia with `julia -t 4`):
```
julia> Threads.nthreads()
4

julia> using DataFrames

julia> using BenchmarkTools

julia> df = DataFrame(reshape(1:50_000_000, :, 25), :auto);

julia> df.id1 = repeat(1:10_000, 200);

julia> df.id2 = string.(df.id1);

julia> summary(df)
"2000000×27 DataFrame"

julia> @btime copy($df); # constructor
  34.748 ms (265 allocations: 412.01 MiB)

julia> @btime $df[1:2:end, :]; # indexing
  34.026 ms (294 allocations: 206.01 MiB)

julia> @btime groupby($df, :id1); # grouping on refs
  3.024 ms (145 allocations: 15.30 MiB)

julia> @btime groupby($df, :id2); # grouping on hash
  43.516 ms (75 allocations: 62.52 MiB)

julia> @btime innerjoin($df, $df, on=:x1, makeunique=true); # joining
  194.656 ms (1297 allocations: 839.31 MiB)

julia> gdf = groupby(df, :id1);

julia> @btime select($gdf, names($df, r"x") .=> x -> x .^ 2); # many transformations
  419.619 ms (1740902 allocations: 2.01 GiB)

julia> function f(x)
           s = zero(eltype(x))
           for _ in 1:1000, v in x
               s += v
           end
           return s
       end
f (generic function with 1 method)

julia> @btime combine($gdf, :x1 => f); # complex reduction
  542.469 ms (388 allocations: 254.67 KiB)
```

As you can see for `copy` there is almost no benefit on a laptop as the
operation is memory bound. However, potentially on some machines with a
different hardware the gain might be bigger.

On the other end --- the most noticeable benefits are for the two last examples
where we perform transformations of `GroupedDataFrame`. This is understandable,
as in this case I have chosen operations that require computation (and not only
data movement).

Finally note that in the last case (complex reduction), since `combine` uses
multiple threads it is assumed that the transformation function is thread safe.

As a warning let me give an example of a "bad" transformation function:

```
julia> g = Int[];

julia> res = combine(gdf, :x1 => (x -> (t = Threads.threadid(); push!(g, t); t)) => :tid);

julia> combine(groupby(res, :tid), nrow)
4×2 DataFrame
 Row │ tid    nrow
     │ Int64  Int64
─────┼──────────────
   1 │     1   2500
   2 │     2   2500
   3 │     3   2500
   4 │     4   2500

julia> combine(groupby(DataFrame(tid=g), :tid), nrow)
5×2 DataFrame
 Row │ tid    nrow
     │ Int64  Int64
─────┼──────────────
   1 │     0      1
   2 │     1   2333
   3 │     2   2443
   4 │     3   2369
   5 │     4   2498
```

And we can see that we get bad result in the global `g` variable because the
`push!` operation is not thread safe.

To stress the problem let me give two more similar examples which do
not produce a correct result either:

```
julia> g = Int[];

julia> res = combine(gdf, :x1 => (x -> (Threads.threadid(), push!(g, Threads.threadid())[end])) => :tid);

julia> combine(groupby(res, :tid), nrow)
6×2 DataFrame
 Row │ tid     nrow
     │ Tuple…  Int64
─────┼───────────────
   1 │ (1, 1)   2501
   2 │ (2, 3)     41
   3 │ (2, 2)   2459
   4 │ (4, 4)   2500
   5 │ (3, 3)   2466
   6 │ (3, 2)     33

julia> g = Int[];

julia> combine(gdf, :x1 => (x -> push!(g, Threads.threadid())[end]) => :tid);
ERROR: BoundsError: attempt to access 8022-element Vector{Int64} at index [4342]
```

# Manual multi threading

If you want to run a custom code using multi threading you should follow the
recommendation given in the [#2823 PR][pr], which is: objects in [DataFrames.jl][df]
are safe to use when using multi threading for reading but they are not safe
for writing.

Let me give a simple example of how one would typically parallelize computations
(I use the simplest parallelization pattern here). It will be a simulation of
[Buffon's needle problem][buffon].

We start with a single threaded scenario:
```
julia> using DataFrames

julia> Threads.nthreads()
1

julia> function toss_needle(t, l)
           x = rand() * t / 2
           θ = rand() * π / 2
           return x < sin(θ) * l / 2
       end
toss_needle (generic function with 1 method)

julia> function sim_cross(t, l, reps)
           c = 0
           for _ in 1:reps
               c += toss_needle(t, l)
           end
           return c / reps
       end
sim_cross (generic function with 1 method)

julia> function analytical_cross(t, l)
           if t > l
               return 2 * l / (t * π)
           else
               return acos(t / l) * 2 / π + 2 * l * (1 - sqrt(1 - (t / l) ^ 2)) / (t * π)
           end
       end
analytical_cross (generic function with 1 method)

julia> function run_sim(vt, vl)
           p = Vector{Float64}(undef, length(vt))
           for i in eachindex(vt, vl)
               p[i] = sim_cross(vt[i], vl[i], 10^7)
           end
           return p
       end
run_sim (generic function with 1 method)

julia> run_sim([1.0], [1.0])
1-element Vector{Float64}:
 0.6367981

julia> 2 / π # double check if the result is correct
0.6366197723675814

julia> df = DataFrame(t = repeat(0.1:0.1:1.0, inner=9), l = repeat(0.1:0.1:1.0, outer=9))
90×2 DataFrame
 Row │ t        l
     │ Float64  Float64
─────┼──────────────────
   1 │     0.1      0.1
   2 │     0.1      0.2
   3 │     0.1      0.3
  ⋮  │    ⋮        ⋮
  88 │     1.0      0.8
  89 │     1.0      0.9
  90 │     1.0      1.0
         84 rows omitted

julia> transform!(df, [:t, :l] => ByRow(analytical_cross) => :prob_a)
90×3 DataFrame
 Row │ t        l        prob_a
     │ Float64  Float64  Float64
─────┼────────────────────────────
   1 │     0.1      0.1  0.63662
   2 │     0.1      0.2  0.837248
   3 │     0.1      0.3  0.89288
  ⋮  │    ⋮        ⋮        ⋮
  88 │     1.0      0.8  0.509296
  89 │     1.0      0.9  0.572958
  90 │     1.0      1.0  0.63662
                   84 rows omitted

julia> @time transform!(df, [:t, :l] => run_sim => :prob_s)
 27.173996 seconds (3.67 k allocations: 235.219 KiB, 0.03% compilation time)
90×4 DataFrame
 Row │ t        l        prob_a    prob_s
     │ Float64  Float64  Float64   Float64
─────┼──────────────────────────────────────
   1 │     0.1      0.1  0.63662   0.63651
   2 │     0.1      0.2  0.837248  0.83734
   3 │     0.1      0.3  0.89288   0.892879
  ⋮  │    ⋮        ⋮        ⋮         ⋮
  88 │     1.0      0.8  0.509296  0.509203
  89 │     1.0      0.9  0.572958  0.57293
  90 │     1.0      1.0  0.63662   0.636605
                             84 rows omitted

julia> extrema( df.prob_s ./ df.prob_a .- 1.0)
(-0.002207239593510546, 0.0005603464546692916)
```

Now we will use 4 threads. Note that I add `Threads.@spawn` and `@sync` in the
`run_sim` function.

```
julia> using DataFrames

julia> Threads.nthreads()
4

julia> function toss_needle(t, l)
           x = rand() * t / 2
           θ = rand() * π / 2
           return x < sin(θ) * l / 2
       end
toss_needle (generic function with 1 method)

julia> function sim_cross(t, l, reps)
           c = 0
           for _ in 1:reps
               c += toss_needle(t, l)
           end
           return c / reps
       end
sim_cross (generic function with 1 method)

julia> function analytical_cross(t, l)
           if t > l
               return 2 * l / (t * π)
           else
               return acos(t / l) * 2 / π + 2 * l * (1 - sqrt(1 - (t / l) ^ 2)) / (t * π)
           end
       end
analytical_cross (generic function with 1 method)

julia> function run_sim(vt, vl)
           p = Vector{Float64}(undef, length(vt))
           @sync for i in eachindex(vt, vl)
               Threads.@spawn p[i] = sim_cross(vt[i], vl[i], 10^7)
           end
           return p
       end
run_sim (generic function with 1 method)

julia> run_sim([1.0], [1.0])
1-element Vector{Float64}:
 0.6366706

julia> 2 / π # double check if the result is correct
0.6366197723675814

julia> df = DataFrame(t = repeat(0.1:0.1:1.0, inner=9), l = repeat(0.1:0.1:1.0, outer=9))
90×2 DataFrame
 Row │ t        l
     │ Float64  Float64
─────┼──────────────────
   1 │     0.1      0.1
   2 │     0.1      0.2
   3 │     0.1      0.3
  ⋮  │    ⋮        ⋮
  89 │     1.0      0.9
  90 │     1.0      1.0
         85 rows omitted

julia> transform!(df, [:t, :l] => ByRow(analytical_cross) => :prob_a)
90×3 DataFrame
 Row │ t        l        prob_a
     │ Float64  Float64  Float64
─────┼────────────────────────────
   1 │     0.1      0.1  0.63662
   2 │     0.1      0.2  0.837248
   3 │     0.1      0.3  0.89288
  ⋮  │    ⋮        ⋮        ⋮
  89 │     1.0      0.9  0.572958
  90 │     1.0      1.0  0.63662
                   85 rows omitted

julia> @time transform!(df, [:t, :l] => run_sim => :prob_s)
  7.612572 seconds (4.35 k allocations: 341.516 KiB, 0.11% compilation time)
90×4 DataFrame
 Row │ t        l        prob_a    prob_s
     │ Float64  Float64  Float64   Float64
─────┼──────────────────────────────────────
   1 │     0.1      0.1  0.63662   0.636561
   2 │     0.1      0.2  0.837248  0.83723
   3 │     0.1      0.3  0.89288   0.892781
  ⋮  │    ⋮        ⋮        ⋮         ⋮
  89 │     1.0      0.9  0.572958  0.573051
  90 │     1.0      1.0  0.63662   0.636735
                             85 rows omitted

julia> extrema( df.prob_s ./ df.prob_a .- 1.0)
(-0.0017633325515583609, 0.0010282866804216528)
```

As you can see since the operation we do is CPU intensive we have a noticeable
performance boost when using multiple threads.

# Conclusions

In summary when you consider using multiple threads with [DataFrames.jl][df]
remember that:
* not all operations will get a big boost when using multiple threads; this
  is especially true when the task you want to perform is memory bound;
* some operations in [DataFrames.jl][df] use multiple threads automatically (and
  you can expect that in the future the support of multi threading will grow);
  when this is the case and you use a custom transformation function make sure
  it is thread safe;
* you can safely use standard multi threading constructs offered by Julia;
  however, remember that types offered by [DataFrames.jl][df] are not
  thread safe for writing.

[df]: https://github.com/JuliaData/DataFrames.jl
[pr]: https://github.com/JuliaData/DataFrames.jl/pull/2823
[buffon]: https://en.wikipedia.org/wiki/Buffon%27s_needle_problem
