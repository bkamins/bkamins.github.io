---
layout: post
title:  "Categorical vs pooled arrays"
date:   2022-01-21 10:32:13 +0200
categories: julialang
---

# Introduction

When working with data in Julia you most likely have encountered two packages
[PooledArrays.jl][pool] and [CategoricalArrays.jl][cat]. They both provide
custom implementations of `AbstractArray` that are useful when you have low
cardinality data.

In this post I want to highlight major similarities and differences between
these two packages.

The examples were tested under Julia 1.7.0, DataFrames.jl 1.3.1,
PooledArrays.jl 1.4.0, CategoricalArrays.jl 0.10.2, FreqTables.jl 0.4.5,
and BenchmarkTols 1.2.2.

# PooledArrays.jl

As I have explained in my [recent post][post] you can use PooledArrays.jl when
you have large vectors that have few unique values. The simplest way of thinking
about `PooledArray` type is that it can be used as a way to compress the data.

Here is an example:

```
julia> using PooledArrays

julia> v = repeat(["a", "b", "c", "d"], 10^6)
4000000-element Vector{String}:
 "a"
 "b"
 "c"
 "d"
 ⋮
 "a"
 "b"
 "c"
 "d"

julia> Base.summarysize(v)
32000076

julia> pv = PooledArray(v)
4000000-element PooledVector{String, UInt32, Vector{UInt32}}:
 "a"
 "b"
 "c"
 "d"
 ⋮
 "a"
 "b"
 "c"
 "d"

julia> Base.summarysize(pv)
16000580
```

As you can see even for short strings we have achieved significant compression.
You can have even better compression by passing `compress=true` keyword argument:

```
julia> pv2 = PooledArray(v, compress=true)
4000000-element PooledVector{String, UInt8, Vector{UInt8}}:
 "a"
 "b"
 "c"
 "d"
 ⋮
 "b"
 "c"
 "d"

julia> Base.summarysize(PooledArray(v, compress=true))
4000532
```

There is one more benefit of using `PooledArray` when working with DataFrames.jl.
You can expect grouping and join operations to be performed faster. Here are
some examples:

```
julia> using DataFrames

julia> using BenchmarkTools

julia> df = DataFrame(;v, pv, pv2)
4000000×3 DataFrame
     Row │ v       pv      pv2
         │ String  String  String
─────────┼────────────────────────
       1 │ a       a       a
       2 │ b       b       b
       3 │ c       c       c
    ⋮    │   ⋮       ⋮       ⋮
 3999998 │ b       b       b
 3999999 │ c       c       c
 4000000 │ d       d       d
              3999994 rows omitted

julia> @benchmark groupby($df, :v)
BenchmarkTools.Trial: 61 samples with 1 evaluation.
 Range (min … max):  66.280 ms … 170.877 ms  ┊ GC (min … max): 0.68% … 58.51%
 Time  (median):     79.254 ms               ┊ GC (median):    0.62%
 Time  (mean ± σ):   82.754 ms ±  17.021 ms  ┊ GC (mean ± σ):  5.53% ± 10.36%

     █          ▇
  ▃▃▃█▅▁▄█▅▄▁▁▃▄█▆▃▁▁▃▃▁▁▁▁▃▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▃ ▁
  66.3 ms         Histogram: frequency by time          154 ms <

 Memory estimate: 125.04 MiB, allocs estimate: 37.

julia> @benchmark groupby($df, :pv)
BenchmarkTools.Trial: 678 samples with 1 evaluation.
 Range (min … max):  3.920 ms … 20.284 ms  ┊ GC (min … max): 0.00% …  4.13%
 Time  (median):     4.835 ms              ┊ GC (median):    0.00%
 Time  (mean ± σ):   7.373 ms ±  4.111 ms  ┊ GC (mean ± σ):  5.02% ± 10.96%

  ▃█▇▃▂    ▁▅                       ▁ ▄▁▃▂
  ██████▇▆▇██▇▆▆▆▅▆▇▆█▆▄▅▁▁▆▁▁▆▄▇▅▅▆█▄█████▇▄▅▄▄▄▆▅▅▅▅▁▄▁▅▄▅ ▇
  3.92 ms      Histogram: log(frequency) by time     18.6 ms <

 Memory estimate: 30.52 MiB, allocs estimate: 61.

julia> @benchmark groupby($df, :pv2)
BenchmarkTools.Trial: 710 samples with 1 evaluation.
 Range (min … max):  3.462 ms … 19.995 ms  ┊ GC (min … max): 0.00% …  8.23%
 Time  (median):     4.356 ms              ┊ GC (median):    0.00%
 Time  (mean ± σ):   7.035 ms ±  4.189 ms  ┊ GC (mean ± σ):  5.63% ± 11.95%

  ▂▄█▃▁   ▁  ▄▁                       ▃  ▃
  ██████▇▆█▇▇██▇▇▇▆▅▆▇▆▄▅▁▁▁▄▄▆▄▄▇▄▁▄▇██████▇▅▇▅▄▅▄▅▄▁▄▄▅▅▅▇ ▇
  3.46 ms      Histogram: log(frequency) by time       18 ms <

 Memory estimate: 30.52 MiB, allocs estimate: 61.

julia> df2 = df[1:4, :]
4×3 DataFrame
 Row │ v       pv      pv2
     │ String  String  String
─────┼────────────────────────
   1 │ a       a       a
   2 │ b       b       b
   3 │ c       c       c
   4 │ d       d       d

julia> @benchmark innerjoin($df, $df2, on=:v, makeunique=true)
BenchmarkTools.Trial: 26 samples with 1 evaluation.
 Range (min … max):  153.954 ms … 281.650 ms  ┊ GC (min … max): 1.63% … 34.70%
 Time  (median):     189.137 ms               ┊ GC (median):    1.12%
 Time  (mean ± σ):   193.827 ms ±  26.318 ms  ┊ GC (mean ± σ):  5.15% ±  8.92%

           ▁   ▁ ▁█  ▄ ▁
  ▆▁▁▆▁▁▁▁▁█▆▆▁█▁██▆▁█▁█▆▆▆▁▆▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▆▁▁▁▁▁▁▁▁▆ ▁
  154 ms           Histogram: frequency by time          282 ms <

 Memory estimate: 143.01 MiB, allocs estimate: 262.

julia> @benchmark innerjoin($df, $df2, on=:pv, makeunique=true)
BenchmarkTools.Trial: 32 samples with 1 evaluation.
 Range (min … max):  124.786 ms … 258.973 ms  ┊ GC (min … max): 2.46% … 42.85%
 Time  (median):     152.645 ms               ┊ GC (median):    2.56%
 Time  (mean ± σ):   157.173 ms ±  28.157 ms  ┊ GC (mean ± σ):  7.05% ±  9.68%

  ▄        ██▁ █▁ ▄▄
  █▁▆▁▁▆▁▁▁███▆██▁██▁▁▆▁▁▁▁▆▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▆▁▁▁▁▆ ▁
  125 ms           Histogram: frequency by time          259 ms <

 Memory estimate: 158.27 MiB, allocs estimate: 274.

julia> @benchmark innerjoin($df, $df2, on=:pv2, makeunique=true)
BenchmarkTools.Trial: 34 samples with 1 evaluation.
 Range (min … max):  119.717 ms … 252.698 ms  ┊ GC (min … max): 1.92% … 44.18%
 Time  (median):     143.639 ms               ┊ GC (median):    1.02%
 Time  (mean ± σ):   147.490 ms ±  26.220 ms  ┊ GC (mean ± σ):  6.05% ± 10.18%

  ▁     █   ▃▆▁  ▁
  █▁▁▁▁▄█▇▄▁███▁▁█▇▁▄▁▄▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▄▁▁▁▁▁▁▁▁▁▄ ▁
  120 ms           Histogram: frequency by time          253 ms <

 Memory estimate: 169.72 MiB, allocs estimate: 274.
```

Apart from these differences working with pooled vectors should have no
user visible differences in comparison to working with normal vectors.

# CategoricalArrays.jl

The `CategoricalArray` type is also performing data compression, but its main
use case is when you want to represent your data as categorical in a statistical
sesnse (both ordered and unordered).

However, let me start with showing you the compression effect and grouping
and joining speedups, as they are the same as for PooledArrays.jl.

```
julia> using CategoricalArrays

julia> cv = categorical(v)
4000000-element CategoricalArray{String,1,UInt32}:
 "a"
 "b"
 "c"
 "d"
 "a"
 ⋮
 "a"
 "b"
 "c"
 "d"

julia> Base.summarysize(cv)
16000612

julia> cv2 = categorical(v, compress=true)
4000000-element CategoricalArray{String,1,UInt8}:
 "a"
 "b"
 "c"
 "d"
 "a"
 ⋮
 "a"
 "b"
 "c"
 "d"

julia> Base.summarysize(cv2)
4000564

julia> df = DataFrame(;v, cv, cv2)
4000000×3 DataFrame
     Row │ v       cv    cv2
         │ String  Cat…  Cat…
─────────┼────────────────────
       1 │ a       a       a
       2 │ b       b       b
       3 │ c       c       c
    ⋮    │   ⋮       ⋮       ⋮
 3999998 │ b       b       b
 3999999 │ c       c       c
 4000000 │ d       d       d
              3999994 rows omitted

julia> @benchmark groupby($df, :v)
BenchmarkTools.Trial: 54 samples with 1 evaluation.
 Range (min … max):  72.757 ms … 109.888 ms  ┊ GC (min … max): 0.00% … 4.32%
 Time  (median):     92.208 ms               ┊ GC (median):    3.27%
 Time  (mean ± σ):   92.917 ms ±  11.122 ms  ┊ GC (mean ± σ):  3.53% ± 2.48%

                        ▆           ▂                      ▆ █
  ▄▄▁▁▁▁▁▄▁▄▄▆▆▁▁▄▄▁▄▁▄▆█▆▁▁▁▁▁▆▁▄▆██▄▁▁▁▄▁▁▁▁▁▁▁▁▁▄▁▁▁▁▁▁▆█▁█ ▁
  72.8 ms         Histogram: frequency by time          108 ms <

 Memory estimate: 125.04 MiB, allocs estimate: 33.

julia> @benchmark groupby($df, :cv)
BenchmarkTools.Trial: 714 samples with 1 evaluation.
 Range (min … max):  3.930 ms … 15.026 ms  ┊ GC (min … max): 0.00% …  4.58%
 Time  (median):     4.474 ms              ┊ GC (median):    0.00%
 Time  (mean ± σ):   7.007 ms ±  3.824 ms  ┊ GC (mean ± σ):  5.99% ± 12.26%

  ▃ █▇              ▄▁                               ▁ ▅ ▂ ▄
  █▇██▆▇▄▁▄▁▁▁▁▄▆▅▄███▁▁▁▅▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▆▅▁▆▆▁▁▅▅▁█▁█▇█▇█ ▇
  3.93 ms      Histogram: log(frequency) by time     14.1 ms <

 Memory estimate: 30.52 MiB, allocs estimate: 57.

julia> @benchmark groupby($df, :cv2)
BenchmarkTools.Trial: 634 samples with 1 evaluation.
 Range (min … max):  3.506 ms … 22.074 ms  ┊ GC (min … max): 0.00% …  0.00%
 Time  (median):     5.651 ms              ┊ GC (median):    0.00%
 Time  (mean ± σ):   7.881 ms ±  4.817 ms  ┊ GC (mean ± σ):  6.49% ± 13.01%

  ▁█▆▂▂ ▃▃▂ ▄▂      ▁           ▄ ▄▁
  ████████████▆▇▅▆█▆█▇▅▁▁▅▅▆▄▇▅▆█████▇▄▅▅▆▅▄▆▅▅▁▇▄▅▆▆▇▅▇▆▆▄▄ ▇
  3.51 ms      Histogram: log(frequency) by time     21.2 ms <

 Memory estimate: 30.52 MiB, allocs estimate: 57.

julia> df2 = df[1:4, :]
4×3 DataFrame
 Row │ v       cv    cv2
     │ String  Cat…  Cat…
─────┼────────────────────
   1 │ a       a     a
   2 │ b       b     b
   3 │ c       c     c
   4 │ d       d     d

julia> @benchmark innerjoin($df, $df2, on=:v, makeunique=true)
BenchmarkTools.Trial: 27 samples with 1 evaluation.
 Range (min … max):  158.062 ms … 193.518 ms  ┊ GC (min … max): 0.46% … 1.62%
 Time  (median):     187.180 ms               ┊ GC (median):    1.66%
 Time  (mean ± σ):   186.249 ms ±   7.073 ms  ┊ GC (mean ± σ):  1.89% ± 0.88%

                                                   █▂
  ▄▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▄▁▁▁▁▁▁▁▁▁▄▁▁▁▁▁▁▁▁▁▁▁▁▁▁███▄▄▆▄▄▄▄▁▁▆ ▁
  158 ms           Histogram: frequency by time          194 ms <

 Memory estimate: 143.02 MiB, allocs estimate: 275.

julia> @benchmark innerjoin($df, $df2, on=:cv, makeunique=true)
BenchmarkTools.Trial: 33 samples with 1 evaluation.
 Range (min … max):  133.837 ms … 253.376 ms  ┊ GC (min … max): 0.54% … 43.65%
 Time  (median):     152.783 ms               ┊ GC (median):    1.83%
 Time  (mean ± σ):   154.100 ms ±  21.196 ms  ┊ GC (mean ± σ):  5.68% ±  8.33%

    █▂     ▅ ▅▂
  ▅▁██▅██▅▅████▁▅▁▁▁▁▁▁▁▁▁▅▁▁▅▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▅ ▁
  134 ms           Histogram: frequency by time          253 ms <

 Memory estimate: 158.27 MiB, allocs estimate: 284.

julia> @benchmark innerjoin($df, $df2, on=:cv2, makeunique=true)
BenchmarkTools.Trial: 31 samples with 1 evaluation.
 Range (min … max):  118.869 ms … 292.135 ms  ┊ GC (min … max): 0.63% … 39.85%
 Time  (median):     149.992 ms               ┊ GC (median):    2.45%
 Time  (mean ± σ):   161.881 ms ±  35.070 ms  ┊ GC (mean ± σ):  8.69% ± 10.37%

            █
  ▃▁▁▁▁▃▃▁▁▃█▆▃▃▃▃▁▃▄▃▃▃▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▃▁▁▁▁▃ ▁
  119 ms           Histogram: frequency by time          292 ms <

 Memory estimate: 169.72 MiB, allocs estimate: 284.
```

So what is the difference?

First, only a limited set of value types can be stored in categorical array:

```
julia> categorical([nothing])
ERROR: ArgumentError: CategoricalArray only supports AbstractString, AbstractChar and Number element types (got element type Nothing)
```

Second, when you retrieve values from categorical arrays they are wrapped in
a custom type:

```
julia> cv[1]
CategoricalValue{String, UInt32} "a"
```

You can extract out the underlying value using the `unwrap` function:

```
julia> unwrap(cv[1])
"a"
```

Why do we have this behavior? The reason is that values stored in the
categorical array are treated as categories, and the wrapped value is just a
label for some category.

Let us first check the order of levels in the `cv` vector and try to compare two
of its elements:

```
julia> levels(cv)
4-element Vector{String}:
 "a"
 "b"
 "c"
 "d"

julia> cv[1] < cv[2]
ERROR: ArgumentError: Unordered CategoricalValue objects cannot be tested for order using <. Use isless instead, or call the ordered! function on the parent array to change this
```

The order of levels will be reflected e.g. if you use FreqTables.jl `freqtable`
function on such a vector:

```
julia> using FreqTables

julia> freqtable(cv)
4-element Named Vector{Int64}
Dim1  │
──────┼────────
"a"   │ 1000000
"b"   │ 1000000
"c"   │ 1000000
"d"   │ 1000000
```

Let us make the `cv` vector ordered and change the order of levels:

```
julia> ordered!(cv, true);

julia> levels!(cv, ["d", "b", "a", "c"]);
```

Now try doing the same operations:

```
julia> levels(cv)
4-element Vector{String}:
 "d"
 "b"
 "a"
 "c"

julia> cv[1] < cv[2]
false

julia> cv[1], cv[2]
(CategoricalValue{String, UInt32} "a" (3/4), CategoricalValue{String, UInt32} "b" (2/4))

julia> freqtable(cv)
4-element Named Vector{Int64}
Dim1  │
──────┼────────
"d"   │ 1000000
"b"   │ 1000000
"a"   │ 1000000
"c"   │ 1000000
```

As you can see `freqtable` respects the ordering of levels. Also when we compared
`cv[1]` vs `cv[2]` the order is respected (i.e. since level `"a"` is in position
3, and level `"b"` is in position 2 the comparison returns `false`).

# Conclusions

In summary, most of the time using PooledArrays.jl is what you will need if you
only require to save memory when working with large data. However, if you want
to process data that is categorical in statistical sense use
CategoricalArrays.jl. There is some mental overhead of working with categorical
values as there is a special `CategoricalValue` type that you need to learn how
to work with. However, the benefit is that the information about levels and
their ordering is retained and can be used if you do operations on elements
of categorical vector.

[cat]: https://github.com/JuliaData/CategoricalArrays.jl
[pool]: https://github.com/JuliaData/PooledArrays.jl
[df]: https://github.com/JuliaData/DataFrames.jl
[post]: https://bkamins.github.io/julialang/2021/12/03/strings.html
