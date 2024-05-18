---
layout: post
title:  "DataFrames.jl: avoiding compilation"
date:   2024-05-17 16:32:44 +0200
categories: julialang
---

# Introduction

Today I want to go back to a topic of performance considerations of using anonymous functions in combination with DataFrames.jl.
I have written about it in the past, but it is an issue that new users often ask about.
In the post I will explain you the problem, its causes, and how to avoid it.

The post was written under Julia 1.10.1 and DataFrames.jl 1.6.1.

# Performance issues caused by anonymous functions

Consider the following sequence of DataFrames.jl operations (I am suppressing the output of the operations with `;` as it is irrelevant):

```
julia> using DataFrames

julia> df = DataFrame(x=1:3);

julia> @time select(df, :x => (x -> 2 * x) => :x2);
  0.077134 seconds (73.33 k allocations: 5.010 MiB, 99.28% compilation time)

julia> @time select(df, :x => (x -> 2 * x) => :x2);
  0.013731 seconds (6.30 k allocations: 450.148 KiB, 98.15% compilation time)

julia> @time subset(df, :x => ByRow(x -> x > 1.5));
  0.094046 seconds (91.86 k allocations: 6.219 MiB, 99.29% compilation time)

julia> @time subset(df, :x => ByRow(x -> x > 1.5));
  0.086597 seconds (43.05 k allocations: 2.881 MiB, 42.62% gc time, 99.44% compilation time)
```

In both `select` and `subset` examples I used an anonymous function. In the first case it was `x -> 2 * x`, and in the second `x -> x > 1.5`.

What you can notice is that most of the time (even in consecutive calls) is spent in compilation. What is the reason for this?

Let me explain this by example. When we pass the `x -> 2 * x` function to `select`, the `select` function needs to be compiled. Since the `select` function
is quite complex its compilation time is long.

Why does this happen. The reason is that each time we write `x -> 2 * x` Julia defines a new anonymous function. Julia compiler does not recognize that it is in fact the same function. Have a look here:

```
julia> x -> 2 * x
#9 (generic function with 1 method)

julia> x -> 2 * x
#11 (generic function with 1 method)
```

We can see that we get a different function (one denoted `#9` and the other `#11`) although the definition of the function is identical.

# How to solve the compilation issue?

Fortunately, there is a simple way to resolve this problem. Instead of using an anonymous function, just use a named function:

```
julia> times2(x) = 2 * x
times2 (generic function with 1 method)

julia> @time select(df, :x => times2 => :x2);
  0.013728 seconds (5.63 k allocations: 401.305 KiB, 98.54% compilation time)

julia> @time select(df, :x => times2 => :x2);
  0.000142 seconds (71 allocations: 3.516 KiB)

julia> gt15(x) = x > 1.5
gt15 (generic function with 1 method)

julia> @time subset(df, :x => ByRow(gt15));
  0.041173 seconds (42.64 k allocations: 2.849 MiB, 99.01% compilation time)

julia> @time subset(df, :x => ByRow(gt15));
  0.000165 seconds (120 allocations: 5.648 KiB)
```

Now you see that consecutive calls are fast and do not cause compilation.

Actually, instead of defining the `gt15` function we could just have written `>(1.5)`:

```
julia> >(1.5)
(::Base.Fix2{typeof(>), Float64}) (generic function with 1 method)
```

Which defines a functor that works as a named function (so it requires only one compilation):

```
julia> @time subset(df, :x => ByRow(>(1.5)));
  0.075423 seconds (41.80 k allocations: 2.804 MiB, 99.32% compilation time)

julia> @time subset(df, :x => ByRow(>(1.5)));
  0.000189 seconds (124 allocations: 5.898 KiB)
```

If you want to learn how functors work in Julia, have a look [here][functor].

# Conclusions

Today I have presented some simple examples, but I hope that they are useful for new users of DataFrames.jl in helping them to improve the performance of their code.

[functor]: https://docs.julialang.org/en/v1/manual/methods/#Function-like-objects
