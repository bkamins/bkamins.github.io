---
layout: post
title:  "Values' mutability in Julia"
date:   2022-04-29 08:21:51 +0200
categories: julialang
---

# Introduction

This week chapter 4 of my upcoming [Julia for Data Analysis][jda] book has been
released by Manning in MEAP. Therefore, following the plan I have announced
in [this post][post], I will discuss a topic related to the material I cover
in chapter 4, but that was not included in the book.

In chapter 4 I give an introduction to working with collections in Julia.
A fundamental topic that is related with this subject is understanding
that values in Julia can be either mutable (like `Vector` or `Dict`) or
immutable (like `Tuple` or `NamedTuple`).

In this post I want to present an example showing the relevance of this
distinction.

The codes I use were tested under Julia 1.7.2 and BenchmarkTools.jl 1.3.1.

# How to check if some value is mutable?

If you have some value you can check if it is mutable using the `ismutable`
function. Let us check it on some example:

```
julia> x = big(1)
1

julia> typeof(x)
BigInt

julia> ismutable(x)
true
```

As you can see the value having `BigInt` type is mutable. This means that
it can be changed in-place. Therefore, you now know that you must be
careful when passing `BigInt` values to functions as you cannot assume
that such functions will not change the passed value.

Let us check that this is indeed the case:

```
julia> x
1

julia> Base.GMP.flipsign!(x, -1)
-1

julia> x
-1
```

In this case the `Base.GMP.flipsign!` function mutated the value bound to the
`x` variable name.

# Is mutability of `BigInt` useful?

You might ask why `BigInt` values are made mutable. This is a bit surprising
as other standard numeric types like `Int64`, `Bool`, or `Float64` are
immutable. The reason is that in certain cases it allows to perform operations
on `BigInt` values in a faster way. Here is an example of computing a sum
of elements of an array storing `BigInt` values:

```
julia> using BenchmarkTools

julia> v = collect(big(1):big(1_000_000));

julia> function sum1(v::AbstractArray{BigInt})
           s = big(0)
           for x in v
               s += x
           end
           return s
       end
sum1 (generic function with 1 method)

julia> function sum2(v::AbstractArray{BigInt})
           s = big(0)
           for x in v
               Base.GMP.MPZ.add!(s, x)
           end
           return s
       end
sum2 (generic function with 1 method)

julia> @btime sum1($v)
  97.111 ms (2000002 allocations: 45.78 MiB)
500000500000

julia> @btime sum2($v)
  27.560 ms (3 allocations: 48 bytes)
500000500000

julia> @btime sum($v)
  27.557 ms (3 allocations: 48 bytes)
500000500000
```

The difference between `sum1` and `sum2` is that the former uses `+` to make
addition and the later uses `Base.GMP.MPZ.add!`, which updates its first
argument in-place. As you can see `sum2` allocates much less memory and is
faster. Additionally I have shown you that `sum` has essentially the same
performance. This suggests that `sum` has a special method for performing
summation of `BigInt` values. Indeed this is the case, the implementation of
this method can be found in *base/gmp.jl* file and is as follows:

```
sum(arr::Union{AbstractArray{BigInt}, Tuple{BigInt, Vararg{BigInt}}}) =
    foldl(MPZ.add!, arr; init=BigInt(0))
```

We can see that it uses an in-place `add!` function.

Note that both in `sum2` and `sum` functions it was crucial to use a new
`BigInt` value as an accumulator. The point is that since `add!` mutates
it in-place we must not use any value stored in the source array for this
purpose.

# Conclusions

I hope that you will find the presented example useful. As a final comment let
me highlight one convention.

In our codes the `add!` function has a `!` suffix. This is a convention that
signals that it might mutate its arguments. This is indeed what happens both in
`sum2` and `sum` functions to the accumulator value. However, `sum2` and `sum`
functions do not need a `!` in their names as in their implementation the passed
array is not mutated (only accumulator value that is created inside these
functions is mutated).

[jda]: https://www.manning.com/books/julia-for-data-analysis
[post]: https://bkamins.github.io/julialang/2022/04/15/dispatch.html
