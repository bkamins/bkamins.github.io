---
layout: post
title:  "Some comments on DataFrames 1.0 release"
date:   2021-04-24 12:51:55 +0200
categories: julialang
---

# Introduction

[DataFrames.jl][df] has just got a 1.0 release.
The major question we should answer following this is:

> What consequences for users does this have?

The answer is pretty boring (but significant): you can expect that we will
not introduce any breaking changes till 2.0 release. The point is that we judge
that the package is mature enough that 2.0 release will not happen soon.
In consequence it is safe to use DataFrames.jl in production code that is expected
not to be updates over longer periods.

The other aspect of calling DataFrames.jl mature enough is that we believe that
the package and its ecosystem have a good performance, so you should be able
to safely use it in your data science project without getting sudden timing
hiccups.

In consequence in this post I want to cover two things. First is what is
deprecated in DataFrames.jl 1.0 release (which means it might get removed or
changed in some 1.x release). The second is some simple performance comparisons.

This post was written under Julia 1.6 and DataFrames.jl 1.0.1.

# Deprecations

#### `indicator` keyword argument in joins

We have a new functionality of `vcat` in DataFrames.jl 1.0.
Start with an example:

```
julia> using DataFrames

julia> df1 = DataFrame(a=1:3)
3×1 DataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │     1
   2 │     2
   3 │     3

julia> df2 = DataFrame(a=4:6)
3×1 DataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │     4
   2 │     5
   3 │     6

julia> vcat(df1, df2, source=:source)
6×2 DataFrame
 Row │ a      source
     │ Int64  Int64
─────┼───────────────
   1 │     1       1
   2 │     2       1
   3 │     3       1
   4 │     4       2
   5 │     5       2
   6 │     6       2

julia> vcat(df1, df2, source=:source => ["df1", "df2"])
6×2 DataFrame
 Row │ a      source
     │ Int64  String
─────┼───────────────
   1 │     1  df1
   2 │     2  df1
   3 │     3  df1
   4 │     4  df2
   5 │     5  df2
   6 │     6  df2
```

As you can see you can pass `source` keyword argument to get an identifier of
source data frame for every row in the resulting data frame.

We have a similar functionality for joins. Here is an example:

```
julia> outerjoin(df1, df2, on=:a, source=:source)
6×2 DataFrame
 Row │ a      source
     │ Int64  String
─────┼───────────────────
   1 │     1  left_only
   2 │     2  left_only
   3 │     3  left_only
   4 │     4  right_only
   5 │     5  right_only
   6 │     6  right_only
```

As you can see the keyword argument used here is `source`. In the past this
keyword argument was called `indicator`, which was not very discoverable and now
it would be not consistent wthi `vcat` either. Therefore `indicator` keyword
argument is deprecated in favor of `source`. It will be removed in 2.0 release,
as keeping both keyword arguments is mostly harmless (so you have a lot of time
to clean up your codes).

#### Broadcasting assignment behavior

This is a super tricky deprecation as it is currently not printed
(because it cannot be under Julia 1.6).

Here is an example of deprecated functionality:

```
~$ julia --depwarn=yes
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.6.0 (2021-03-24)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

julia> using DataFrames

julia> df = DataFrame(a=1:3)
3×1 DataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │     1
   2 │     2
   3 │     3

julia> df.a .= 'x'
3-element Vector{Int64}:
 120
 120
 120

julia> df
3×1 DataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │   120
   2 │   120
   3 │   120
```

In short, as you can see, `df.col = value` syntax works in-place under Julia 1.6.
In this [post][bd] I have discussed in detail why we decided to change it.
But one of the major reasons is to allow for:

```
julia> df.b .= 1
ERROR: ArgumentError: column name :b not found in the data frame; existing most similar names are: :a
```

Let us switch to Julia nightly for a second:

```
$ ./julia --depwarn=yes
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.7.0-DEV.999 (2021-04-24)
 _/ |\__'_|_|_|\__'_|  |  Commit 1474566ffc* (0 days old master)
|__/                   |


julia> using DataFrames

julia> df = DataFrame(a=1:3)
3×1 DataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │     1
   2 │     2
   3 │     3

julia> df.a .= 'x'
┌ Warning: In the future this operation will allocate a new column instead of performing an in-place assignment.
│   caller = top-level scope at REPL[3]:1
└ @ Core REPL[3]:1
3-element Vector{Int64}:
 120
 120
 120

julia> df.b .= 1
3-element Vector{Int64}:
 1
 1
 1

julia> df
3×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │   120      1
   2 │   120      1
   3 │   120      1
```

As you can see `df.a .= 'x'` is deprecated but `df.b .= 1` is allowed.

The situation is very unfortunate, as we are unable to print deprecation
warnings under Julia 1.6. There are two deprecated cases:

* `df.a .= value` into an existing column of a `DataFrame` currently is in-place,
  and will replace a column in the future;
* `dfv.a .= value` will be disallowed if `dfv` is a `SubDataFrame`;

The good thing is that the first change (replace instead of in-place) will not
affect almost any user workflows (except for the case I have given above, when
by doing `df.a .= 'x'` you got `120` in the column; in the future you will get
`'x'`, as most likely you wanted). The second change (error for `SubDataFrame`)
will be easy to fix, as it will throw an error.

In the future (when Julia 1.7 is commonly used), either in 1.x or 2.0 release
of DataFrames.jl these deprecations will be turned into the target functionality.

# Performance

To show performance changes I decided to look for some examples of `data.table`
performance tests prepared by [Jan Gorecki][jg] (Jan is a `data.table` guru and
`data.table` is a golden standard for in-memory data wrangling) that I have not
checked earlier when developing the 1.0 release (to have a fair comparison).

For this I have found [this post][post1] that gave a link to these two Stack
Overflow questions answered by Jan using `data.table`: [question 1][so1] and
[question 2][so2] (one is for split-apply-combine, and the other is for joins).
I have changed the size of the data a bit to make a single run of the test
on `data.table` be around 1 second. Both for `data.table` and DataFrames.jl
I use 4 threads.

I decided to rewrite these two examples under DataFrames.jl and compare timings
of:
* DataFrames.jl 1.0.1
* `data.table` 1.14.0 (to see a competitive comparison)
* DataFrames.jl 0.22.7 (to see the difference in the performance)

(we are on R 4.0.5 and back on Julia 1.6)

Below I share reproducible details but the short conclusion is:
* we have improved;
* we are competitive with `data.table` (at least on the tests that Jan Goercki
  used in the Stack Overflow examples).

(still we have a lot of work to do to improve performance post 1.0 release)

#### Split-apply-combine tests

We start with `data.table` timing:
```
> library(data.table)
data.table 1.14.0 using 4 threads (see ?getDTthreads).  Latest news: r-datatable.com
> library(microbenchmark)
> n = 5e7
> k = 5e5
> x = runif(n)
> grp = sample(k, n, TRUE)
> dt = setnames(setDT(list(x, grp)), c("x","grp"))
> microbenchmark(dt[, .(sum(x), .N), grp], times=10)
Unit: milliseconds
                     expr      min       lq     mean   median       uq      max
 dt[, .(sum(x), .N), grp] 928.9709 976.8374 1087.029 1095.204 1199.951 1220.086
 neval
    10
>
```

Now DataFrames.jl 1.0.1:
```
julia> using DataFrames

julia> using BenchmarkTools

julia> n = 50_000_000;

julia> k = 500_000;

julia> x = rand(n);

julia> grp = rand(1:k, n);

julia> df = DataFrame(x=x, grp=grp);

julia> @benchmark combine(groupby($df, :grp), :x => sum, nrow)
BenchmarkTools.Trial:
  memory estimate:  397.73 MiB
  allocs estimate:  634
  --------------
  minimum time:     593.295 ms (3.19% GC)
  median time:      609.667 ms (2.59% GC)
  mean time:        622.208 ms (2.04% GC)
  maximum time:     668.141 ms (6.77% GC)
  --------------
  samples:          9
  evals/sample:     1
```

Finally DataFrames.jl 0.22.7:
```
julia> using DataFrames

julia> using BenchmarkTools

julia> n = 50_000_000;

julia> k = 500_000;

julia> x = rand(n);

julia> grp = rand(1:k, n);

julia> df = DataFrame(x=x, grp=grp);

julia> @benchmark combine(groupby($df, :grp), :x => sum, nrow)
BenchmarkTools.Trial:
  memory estimate:  1.26 GiB
  allocs estimate:  225
  --------------
  minimum time:     7.936 s (0.00% GC)
  median time:      7.936 s (0.00% GC)
  mean time:        7.936 s (0.00% GC)
  maximum time:     7.936 s (0.00% GC)
  --------------
  samples:          1
  evals/sample:     1
```

#### Join tests

First goes `data.table`:
```
> library(microbenchmark)
> library(data.table)
data.table 1.14.0 using 4 threads (see ?getDTthreads).  Latest news: r-datatable.com
> n = 5e6
> df1 = data.frame(x=sample(n,n), y1=rnorm(n))
> df2 = data.frame(x=sample(n,n), y2=rnorm(n))
> dt1 = as.data.table(df1)
> dt2 = as.data.table(df2)
> microbenchmark(dt1[dt2, nomatch=NULL, on = "x"], times=10)
Unit: milliseconds
                               expr      min       lq    mean   median       uq
 dt1[dt2, nomatch = NULL, on = "x"] 893.9817 905.4534 924.142 914.2142 940.4294
      max neval
 993.6209    10
> microbenchmark(dt2[dt1, on = "x"], times=10)
Unit: milliseconds
               expr      min      lq     mean   median      uq      max neval
 dt2[dt1, on = "x"] 845.7094 884.172 885.5079 889.8894 891.272 903.6629    10
> microbenchmark(dt1[dt2, on = "x"], times=10)
Unit: milliseconds
               expr      min      lq     mean   median       uq      max neval
 dt1[dt2, on = "x"] 848.3348 874.032 878.4387 880.2858 885.1598 894.9579    10
> microbenchmark(merge(dt1, dt2, by = "x", all = TRUE), times=10)
Unit: seconds
                                  expr      min       lq     mean   median
 merge(dt1, dt2, by = "x", all = TRUE) 1.924328 1.931986 1.957051 1.954956
       uq      max neval
 1.981632 2.002862    10
```

Now DataFrames.jl 1.0.1:
```
julia> using DataFrames, Random

julia> using BenchmarkTools

julia> n = 5_000_000;

julia> df1 = DataFrame(x=shuffle(1:n), y1=randn(n));

julia> df2 = DataFrame(x=shuffle(1:n), y2=randn(n));

julia> @benchmark innerjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  228.90 MiB
  allocs estimate:  248
  --------------
  minimum time:     957.133 ms (0.00% GC)
  median time:      974.750 ms (0.13% GC)
  mean time:        975.144 ms (0.65% GC)
  maximum time:     992.020 ms (2.63% GC)
  --------------
  samples:          6
  evals/sample:     1

julia> @benchmark leftjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  234.26 MiB
  allocs estimate:  312
  --------------
  minimum time:     1.061 s (0.00% GC)
  median time:      1.071 s (0.00% GC)
  mean time:        1.073 s (0.41% GC)
  maximum time:     1.084 s (0.00% GC)
  --------------
  samples:          5
  evals/sample:     1

julia> @benchmark rightjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  234.26 MiB
  allocs estimate:  312
  --------------
  minimum time:     980.800 ms (0.00% GC)
  median time:      1.003 s (0.36% GC)
  mean time:        1.001 s (0.80% GC)
  maximum time:     1.013 s (2.63% GC)
  --------------
  samples:          6
  evals/sample:     1

julia> @benchmark outerjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  239.63 MiB
  allocs estimate:  332
  --------------
  minimum time:     1.081 s (0.00% GC)
  median time:      1.096 s (0.00% GC)
  mean time:        1.097 s (0.41% GC)
  maximum time:     1.106 s (0.66% GC)
  --------------
  samples:          5
  evals/sample:     1
```

Finally DataFrames.jl 0.22.7:
```
julia> using DataFrames, Random

julia> using BenchmarkTools

julia> n = 5_000_000;

julia> df1 = DataFrame(x=shuffle(1:n), y1=randn(n));

julia> df2 = DataFrame(x=shuffle(1:n), y2=randn(n));

julia> @benchmark innerjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  674.36 MiB
  allocs estimate:  218
  --------------
  minimum time:     4.788 s (0.55% GC)
  median time:      4.795 s (0.44% GC)
  mean time:        4.795 s (0.44% GC)
  maximum time:     4.803 s (0.32% GC)
  --------------
  samples:          2
  evals/sample:     1

julia> @benchmark leftjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  755.43 MiB
  allocs estimate:  223
  --------------
  minimum time:     4.975 s (0.36% GC)
  median time:      4.976 s (0.27% GC)
  mean time:        4.976 s (0.27% GC)
  maximum time:     4.978 s (0.18% GC)
  --------------
  samples:          2
  evals/sample:     1

julia> @benchmark rightjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  760.20 MiB
  allocs estimate:  237
  --------------
  minimum time:     5.077 s (0.39% GC)
  median time:      5.077 s (0.39% GC)
  mean time:        5.077 s (0.39% GC)
  maximum time:     5.077 s (0.39% GC)
  --------------
  samples:          1
  evals/sample:     1

julia> @benchmark outerjoin($df1, $df2, on=:x)
BenchmarkTools.Trial:
  memory estimate:  769.73 MiB
  allocs estimate:  228
  --------------
  minimum time:     5.148 s (0.18% GC)
  median time:      5.148 s (0.18% GC)
  mean time:        5.148 s (0.18% GC)
  maximum time:     5.148 s (0.18% GC)
  --------------
  samples:          1
  evals/sample:     1
```

# Conclusions

In summary let me highlight some of the development objectives after 1.0 release:
* API improvements (e.g. better `stack`/`unstack` functionality, having in-place
  joins, extensions of transformation minilanguage).
* Review of the whole package for performance bottlenecks and using
  multi-threading in more places.
* Resolving ecosystem integration performance bottlenecks (especially against
  CSV.jl and Arrow.jl). Here one big challenge is thinking of something that
  would reduce GC strain of having millions of strings stored in the `DataFrame`
  (which is a typical situation in many data science workflows).
* Documentation improvements.

Happy data wrangling with DataFrames.jl 1.0!

[df]: https://github.com/JuliaData/DataFrames.jl
[bd]: https://discourse.julialang.org/t/release-announcements-for-dataframes-jl/18258/145
[jg]: https://github.com/jangorecki
[post1]: https://jangorecki.github.io/blog/2015-12-11/Solve-common-R-problems-efficiently-with-data.table.html
[so1]: https://stackoverflow.com/questions/3505701/grouping-functions-tapply-by-aggregate-and-the-apply-family/34167477#34167477
[so2]: https://stackoverflow.com/questions/1299871/how-to-join-merge-data-frames-inner-outer-left-right/34219998#34219998
