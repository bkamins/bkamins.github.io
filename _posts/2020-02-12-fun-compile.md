---
layout: post
title:  "Reducing compilation cost in DataFrames.jl"
date:   2021-02-12 07:44:43 +0200
categories: julialang
---

# Introduction

[DataFrames.jl][df] is a general purpose package. This means that it is maximally
flexible, at the cost of having to handle a very wide variety of use-cases.

This situation leads to quite high complexity of internal code of DataFrames.jl.
In turn, this complexity causes a relatively high time-to-first-result times.
Fortunately [Milan Bouchet-Valat][mbv] in [this PR][pr1] made it much better in
0.22 release due to taking advantage of precompilation, and the hints from [Tim
Holy][th] in [this PR][pr2] and [this PR][pr3] are going to be included in the
1.0 release when we finish changing the core functionality of the package (so
expect even more compiler friendly DataFrames.jl soon).

The things listed above are on DataFrames.jl developer side. However, there is
one simple rule that can help you reduce compilation time on user side and at
the same time make your code more readable (at least this is my preference).

The rule is: avoid using anonymous functions in top-level code as they trigger
compilation each time they are defined (even if the body of the function is not
changed).

Let me jump straight to the examples.

The code was run under Julia 1.5.3 and DataFrames.jl 0.22.5.

# Anonymous function compilation latency

Consider the following code:
```
julia> using DataFrames

julia> df = DataFrame(x=[1,2])
2×1 DataFrame
 Row │ x
     │ Int64
─────┼───────
   1 │     1
   2 │     2

julia> @time transform(df, :x => (x -> x) => :x2)
  0.649323 seconds (1.79 M allocations: 94.619 MiB, 5.09% gc time)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2

julia> @time transform(df, :x => (x -> x) => :x2)
  0.096049 seconds (182.88 k allocations: 9.707 MiB)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2

julia> @time transform(df, :x => (x -> x) => :x2)
  0.092431 seconds (182.88 k allocations: 9.708 MiB)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2
```

As you can see compilation gets triggered each time the `transform` function is
executed (on the first run more compilation has to be done, but in consecutive
runs this cost is still visible). Let us compare it to the following code (I use
a fresh Julia session):

```
julia> using DataFrames

julia> df = DataFrame(x=[1,2])
2×1 DataFrame
 Row │ x
     │ Int64
─────┼───────
   1 │     1
   2 │     2

julia> f(x) = x
f (generic function with 1 method)

julia> @time transform(df, :x => f => :x2)
  0.640971 seconds (1.78 M allocations: 94.119 MiB, 5.26% gc time)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2

julia> @time transform(df, :x => f => :x2)
  0.000077 seconds (96 allocations: 5.641 KiB)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2

julia> @time transform(df, :x => f => :x2)
  0.000083 seconds (96 allocations: 5.641 KiB)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2
```

As you can see Julia needs to compile the code only once.

I have mentioned that I consider the method with a transformation function
pre-defined as more readable. The reason is that, unless the function is very
simple, the code starts being not very readable when an anonymous function is
passed in-line to e.g. `transform`. Also having an informative name for a
transformation function usually helps going back to the code after some time and
understanding what it does.

# When the anonymous function is not a problem

There are cases when defining a fresh anonymous function is not a problem.

The first, and quite common one, is when we use a type-stable [functor][functor]
that is passed a predefined function. Here is a basic example (fresh Julia
session again):

```
julia> using DataFrames

julia> df = DataFrame(x=[1,2])
2×1 DataFrame
 Row │ x
     │ Int64
─────┼───────
   1 │     1
   2 │     2

julia> @time transform(df, :x => ByRow(>(1.5)) => :x2)
  0.747826 seconds (2.18 M allocations: 112.703 MiB, 2.84% gc time)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Bool
─────┼──────────────
   1 │     1  false
   2 │     2   true

julia> @time transform(df, :x => ByRow(>(1.5)) => :x2)
  0.000081 seconds (99 allocations: 5.734 KiB)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Bool
─────┼──────────────
   1 │     1  false
   2 │     2   true

julia> @time transform(df, :x => ByRow(>(1.5)) => :x2)
  0.000083 seconds (99 allocations: 5.734 KiB)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Bool
─────┼──────────────
   1 │     1  false
   2 │     2   true
```

In this example both `>(1.5)` and `ByRow` are parametric struts. So the expression
`ByRow(>(1.5))` has a constant type across a single Julia session.

Of course the function being the argument of the functor must not be anonymous,
as then all is bad again (fresh Julia session):

```
julia> using DataFrames

julia> df = DataFrame(x=[1,2])
2×1 DataFrame
 Row │ x
     │ Int64
─────┼───────
   1 │     1
   2 │     2

julia> @time transform(df, :x => ByRow(x -> x > 1.5) => :x2)
  0.752139 seconds (2.18 M allocations: 112.890 MiB, 4.51% gc time)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Bool
─────┼──────────────
   1 │     1  false
   2 │     2   true

julia> @time transform(df, :x => ByRow(x -> x > 1.5) => :x2)
  0.184907 seconds (538.13 k allocations: 26.211 MiB, 5.16% gc time)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Bool
─────┼──────────────
   1 │     1  false
   2 │     2   true

julia> @time transform(df, :x => ByRow(x -> x > 1.5) => :x2)
  0.169902 seconds (538.12 k allocations: 26.213 MiB)
2×2 DataFrame
 Row │ x      x2
     │ Int64  Bool
─────┼──────────────
   1 │     1  false
   2 │     2   true
```

The second case when having an anonymous function is not a problem is when we
introduce local scope in the code via a loop (this is a pattern that I see quite
often; fresh Julia session):

```
julia> using DataFrames

julia> df = DataFrame(x=[1,2])
2×1 DataFrame
 Row │ x
     │ Int64
─────┼───────
   1 │     1
   2 │     2

julia> for i in 1:3
           @time transform(df, :x => ByRow(x -> x > 1.5) => :x2)
       end
  0.759848 seconds (2.18 M allocations: 112.888 MiB, 4.29% gc time)
  0.000131 seconds (102 allocations: 5.844 KiB)
  0.000102 seconds (102 allocations: 5.844 KiB)
  ```

As you can see this time `x -> x > 1.5` is defined only once within the body of
the loop so `transform` gets compiled only once.

# Conclusions

I hope this post will be useful to reduce latency of your DataFrames.jl code.

I would like to highlight that these recommendations are not just DataFrames.jl
specific. They apply to any function that takes a function as one of its
arguments, e.g. `map`, `filter`, `sum`, ...

Another point to note is that, as I have mentioned above, DataFrames.jl is a
general purpose package that supports [Tables.jl][tables] table interface.
If you have a specific use case it might be better to go for a specialized
package that is tailored to your needs. See e.g. a
[recent discussion][timeseries]
about the design of the TimeSeries.jl package. Still I hope that users
will find DataFrames.jl a good one-stop shop for majority of their data
wrangling needs.

[post]: https://bkamins.github.io/julialang/2021/01/30/bang.html
[df]: https://github.com/JuliaData/DataFrames.jl
[timeseries]: https://github.com/JuliaStats/TimeSeries.jl/issues/482#issuecomment-777379241
[pr1]: https://github.com/JuliaData/DataFrames.jl/pull/2456
[mbv]: https://github.com/nalimilan
[th]: https://github.com/timholy
[pr2]: https://github.com/JuliaData/DataFrames.jl/pull/2597
[pr3]: https://github.com/JuliaData/DataFrames.jl/pull/2563
[functor]: https://docs.julialang.org/en/v1/manual/methods/#Function-like-objects
[tables]: https://github.com/JuliaData/Tables.jl
