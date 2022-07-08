---
layout: post
title:  "Efficiency of data frame row iteration"
date:   2022-07-08 08:06:42 +0200
categories: julialang
---

# Introduction

Last week I have received several questions about efficiency of iteration
over rows of data frames in DataFrames.jl. In this post I summarize the
most important recommendations on this topic.

The code I use was written under Julia 1.7.2, DataFrames.jl 1.3.4,
DataFramesMeta.jl 0.11.0.

# A basic approach

Assume we have a data frame that has two numeric columns `:a` and `:b` and we
want to check if for all rows the value in column `:a` is less than the value in
column `:b`. We want to compare several approaches to this task and check their
performance. (I have chosen an easy task on purpose to concentrate on the issue
of iteration.)

Here is a basic approach to this problem (in all the examples I will run
the operation twice and show the `@time` output to capture the compilation
time and reflect interactive experience of the user):

```
julia> using DataFrames

julia> df = DataFrame(a=1:100_000_000, b=2:100_000_001)
100000000×2 DataFrame
       Row │ a          b
           │ Int64      Int64
───────────┼──────────────────────
         1 │         1          2
         2 │         2          3
     ⋮     │     ⋮          ⋮
  99999999 │  99999999  100000000
 100000000 │ 100000000  100000001
             99999996 rows omitted

julia> @time all(df.a .< df.b)
  0.204820 seconds (326.44 k allocations: 28.022 MiB, 53.94% compilation time)
true

julia> @time all(df.a .< df.b)
  0.093742 seconds (6 allocations: 11.925 MiB)
true
```

I have used broadcasting to present a reference performance of the operation.

# Using data frame row iteration

The first take on our problem is to use the `eachrow` iterator:

```
julia> function f1(df)
           for row in eachrow(df)
               row.a < row.b || return false
           end
           return true
       end
f1 (generic function with 1 method)

julia> @time f1(df)
 19.280229 seconds (700.15 M allocations: 10.438 GiB, 6.00% gc time, 0.20% compilation time)
true

julia> @time f1(df)
 19.465430 seconds (700.00 M allocations: 10.431 GiB, 5.46% gc time)
true
```

As you can see using `eachrow` is slow. This approach is easy, but
should be used only for data frames that have few rows. The reason why it
is slow is that it is not type stable.

# Using named tuples

Here is the approach that is type stable:

```
julia> function f2(nti)
           for row in nti
               row.a < row.b || return false
           end
           return true
       end
f2 (generic function with 1 method)

julia> @time f2(Tables.namedtupleiterator(df))
  0.104822 seconds (7.64 k allocations: 438.955 KiB, 7.41% compilation time)
true

julia> @time f2(Tables.namedtupleiterator(df))
  0.090318 seconds (9 allocations: 336 bytes)
true
```

This time the operation is fast. Note two things though:
* using `Tables.namedtupleiterator` can be slow if data frame has many columns
  (it can have high compilation cost);
* we need to pass `Tables.namedtupleiterator(df)` as an argument to `f2` to
  make this function type stable.

# Using vectors

The next approach that is typically used is to pass the vectors that
we want to compare to the function:

```
julia> function f3(a, b)
           for i in eachindex(a, b)
               @inbounds a[i] < b[i] || return false
           end
           return true
       end
f3 (generic function with 1 method)

julia> @time f3(df.a, df.b)
  0.113009 seconds (6.14 k allocations: 323.976 KiB, 6.03% compilation time)
true

julia> @time f3(df.a, df.b)
  0.082265 seconds
true
```

As you can see the operation is fast this time. I know I can safely use
`@inbounds` because the `i` index is taken from `eachindex(a, b)` that
guarantees that only valid indices are passed.

# Using DataFramesMeta.jl

The last option we consider is using the `@eachrow!` macro from DataFramesMeta.jl:

```
julia> function f4(df)
           flag = true
           @eachrow! df begin
               if !(:a < :b)
                   flag = false
               end
           end
           return flag
       end
f4 (generic function with 1 method)

julia> @time f4(df)
  0.133970 seconds (80.46 k allocations: 4.634 MiB, 16.30% compilation time)
true

julia> @time f4(df)
  0.090125 seconds (27 allocations: 1.578 KiB)
true
```

The operation is also fast. Here it is worth to note two things:
* we use `@eachrow!` not `@eachrow` as the latter would copy `df` which in our
  case is not needed;
* we need to use the `flag` helper variable and we cannot use `break` to stop
  the iteration early (in the example I use this does not affect the result
  since `:a` is always less than `:b` so we iterate all rows anyway, but for
  other data it could matter);

# Conclusions

The take-aways from these examples are as follows:
* `eachrow` is easy to use, but it will be slow if you work with data frame
  that has many rows;
* you can use `Tables.namedtupleiterator` wrapper instead of `eachrow`; it will
  be fast but it can have large compilation time for wide tables (note, however,
  that you can always pass only a narrow data frame to it if not all source
  columns are needed in your operation, for example
  `Tables.namedtupleiterator(df[!, [:a, :b]])` in the code we used in this post);
* you can directly pass columns you want to work with to a function - this
  approach will be fast and gives you most control (at the expense of having to
  write a more low-level code);
* There is the `@eachrow!` macro (and `@eachrow` if you want to copy data) in
  DataFramesMeta.jl that will be also fast. When you use it remember that it
  is designed to always iterate all rows of a data frame.
