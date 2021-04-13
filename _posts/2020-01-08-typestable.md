---
layout: post
title:  "Why DataFrame is not type stable and when it matters"
date:   2021-01-08 16:11:35 +0200
categories: julialang
---

# Introduction

One of the most frequent performance questions related to [DataFrames.jl][df]
are caused by the fact that the `DataFrame` object is not type stable.
[Here][so] is a recent question on Stack Overflow that originated from this
issue. Experienced Julia users are aware of the trade-offs I discuss here, but
they are often surprising for people starting to use DataFrames.jl.

In this post I will want to cover the following issues: a) what does it mean
that `DataFrame` is not type stable, b) what positive consequences it has, c)
when type instability hits the user most and how to handle it.

This post was written under Julia 1.5.3, DataFrames.jl 0.22.2, and Tables 1.2.2.

# What does it mean that `DataFrame` is not type stable?

The reason here is relatively simple. Here is a stripped definition of `DataFrame`:

```
struct DataFrame <: AbstractDataFrame
    columns::Vector{AbstractVector}
    colindex::Index
end
```

The `columns` field stores the vectors that constitute the data frame, and the
`colindex` field maintains a mapping between column numbers and column names.

We can see two crucial things in this definition:
* the bad: `columns` has element type `AbstractVector`, breaking the most
  fundamental rule from [the Julia Manual][perf] about writing high-performance
  Julia code; this design means that when we extract a column from a `DataFrame`
  the Julia compiler is not able to infer its type (and in consequence is not
  able to produce the most efficient code);
* the good: the `DataFrame` type does not have parameters; this means that if
  you run some function once on a data frame it does not have to be recompiled
  later for any `DataFrame` you might pass to it (no matter what columns it
  would contain); it also means that it is possible to efficiently precompile
  many of the functions in the package with known signatures to reduce
  *the time to first result*; finally --- you are free to add or remove columns
  from a `DataFrame` in-place.

The simplest type-stable type that can work similarly to a `DataFrame` is a
`NamedTuple` of vectors. However, it should be stressed that this type-stability
is only available when which columns are extracted from such `NamedTuple` can be
inferred at compile time. Let me give two simple examples of such type
instability for the case of type-stable container like `NamedTuple`:

#### Example #1:

```
julia> nt = (a=[1, 2], b=[3.0, 4.0], c=[true, false])
(a = [1, 2], b = [3.0, 4.0], c = Bool[1, 0])

julia> f(nt, i) = nt[i]
f (generic function with 1 method)

julia> g(nt) = f(nt, 1)
g (generic function with 1 method)

julia> @code_warntype f(nt, 1)
Variables
  #self#::Core.Compiler.Const(f, false)
  nt::NamedTuple{(:a, :b, :c),Tuple{Array{Int64,1},Array{Float64,1},Array{Bool,1}}}
  i::Int64

Body::Array
1 ─ %1 = Base.getindex(nt, i)::Array
└──      return %1

julia> @code_warntype g(nt)
Variables
  #self#::Core.Compiler.Const(g, false)
  nt::NamedTuple{(:a, :b, :c),Tuple{Array{Int64,1},Array{Float64,1},Array{Bool,1}}}

Body::Array{Int64,1}
1 ─ %1 = Main.f(nt, 1)::Array{Int64,1}
└──      return %1
```

Here you can see that you achieve type stability of the result only if the
compiler is able to perform constant propagation (as in the case of the function
`g`). Otherwise, even if, in theory `NamedTuple` is type stable, the type of the
vector returned by function `f` is not known by the compiler.

This leads us to

#### Example #2:

```
julia> function h()
           nt = (a=[1, 2], b=[3.0, 4.0], c=[true, false])
           return sum(nt)
       end
h (generic function with 1 method)

julia> @code_warntype h()
Variables
  #self#::Core.Compiler.Const(h, false)
  nt::NamedTuple{(:a, :b, :c),Tuple{Array{Int64,1},Array{Float64,1},Array{Bool,1}}}

Body::Any
1 ─ %1 = (:a, :b, :c)::Core.Compiler.Const((:a, :b, :c), false)
│   %2 = Core.apply_type(Core.NamedTuple, %1)::Core.Compiler.Const(NamedTuple{(:a, :b, :c),T} where T<:Tuple, false)
│   %3 = Base.vect(1, 2)::Array{Int64,1}
│   %4 = Base.vect(3.0, 4.0)::Array{Float64,1}
│   %5 = Base.vect(true, false)::Array{Bool,1}
│   %6 = Core.tuple(%3, %4, %5)::Tuple{Array{Int64,1},Array{Float64,1},Array{Bool,1}}
│        (nt = (%2)(%6))
│   %8 = Main.sum(nt)::Any
└──      return %8
```

We can see that even if the type of `nt` is known at compile-time the compiler
is not able to determine the return type of the `sum` function as this function
iterates over elements of `nt` (which is not type stable in general; some
functions for small number of iterations might perform loop unrolling in which
case the result would be type stable).

So the first take-away is that it is not enough to have a type-stable data
structure but also you have to write your code in a way that is type stable.

# Why making `DataFrame` type stable would be a problem?

In order to understand why type-stability might be a problem consider the
following examples.

#### Example #3:

Here we will show what happens if we have many data frames to process and they
have a different schema (start a new Julia session):

```
julia> using DataFrames

julia> @time dfs = [DataFrame("a$i" => [isodd(i) ? 1 : 1.0]) for i in 1:10000];
  0.333070 seconds (792.20 k allocations: 49.514 MiB, 6.14% gc time)

julia> @time dfs = [DataFrame("a$i" => [isodd(i) ? 1 : 1.0]) for i in 1:10000];
  0.077577 seconds (298.23 k allocations: 23.319 MiB, 14.77% gc time)

julia> f(df) = df[1, 1]
f (generic function with 1 method)

julia> @time f.(dfs);
  0.109797 seconds (206.24 k allocations: 10.630 MiB)

julia> @time f.(dfs);
  0.001640 seconds (19.50 k allocations: 461.141 KiB)
```
now start a new Julia session again:
```
julia> @time nts = [(;Symbol("a$i") => [isodd(i) ? 1 : 1.0]) for i in 1:10000];
    280.058361 seconds (14.82 M allocations: 886.891 MiB, 0.12% gc time)

julia> @time nts = [(;Symbol("a$i") => [isodd(i) ? 1 : 1.0]) for i in 1:10000];
 18.461082 seconds (193.24 k allocations: 8.826 MiB)

julia> f(nt) = nt[1][1]
f (generic function with 1 method)

julia> @time f.(nts);
 41.306839 seconds (18.02 M allocations: 1.085 GiB, 1.36% gc time)

julia> @time f.(nts);
  0.006529 seconds (19.50 k allocations: 461.141 KiB)
```

As you can see having `DataFrame` non-parametric is much more compiler friendly.
Of course the above examples are artificial, but they show what happens if you
would need to work with data frames constantly changing schemas (e.g. having
repeatedly added/removed columns).

#### Example #4:

Now we switch to consider a single data frame, that is wide (again start a fresh
Julia session):

```
julia> using DataFrames

julia> @time DataFrame(["a$i" => [isodd(i) ? 1 : 1.0] for i in 1:10000]);
  0.495980 seconds (1.22 M allocations: 63.541 MiB, 4.83% gc time)

julia> @time DataFrame(["a$i" => [isodd(i) ? 1 : 1.0] for i in 1:10000]);
  0.077374 seconds (205.31 k allocations: 11.107 MiB)

julia> @time (;[Symbol("a$i") => [isodd(i) ? 1 : 1.0] for i in 1:10000]...);
 23.039182 seconds (2.09 M allocations: 82.427 MiB, 0.12% gc time)

julia> @time (;[Symbol("a$i") => [isodd(i) ? 1 : 1.0] for i in 1:10000]...);
  0.070873 seconds (176.05 k allocations: 9.829 MiB)
```

As you can see we have a situation in which the compiler is very severely
burdened, this time by the fact that our `NamedTuple` is very wide.

In conclusion --- because DataFrames.jl is intended to be a general purpose
package it uses a type-unstable design. In this way we are sure that we are not
going to be hit by compilation-related issues even when having very many tables
having different schemas or very wide tables (and both scenarios are encountered
in practice).

There is one additional consideration that favors making the `DataFrame` type
type-unstable, and it is related to allowing to change the schema of the data
frame in-place. What does *changing schema* mean? Well --- many things users
typically want to do in-place:
* renaming columns of a data frame;
* adding/replacing/removing columns of a data frame;
* using a long list of functions ending with `!` in DataFrames.jl (that
  essentially do one or both of the operations listed above).

# When it really matters that `DataFrame` is not type stable?

Maybe let me start with commenting when it does not matter that `DataFrame` is
not type stable. In general (except for extreme situations) when you are working
on whole columns of a data frame you are not going to be severely punished by
type instability. The only cost you have to pay is the cost of one dynamic
dispatch to the function to which you are passing the extracted column (this
strategy is used in DataFrames.jl e.g. in the `select` function).

Let us have a look at some simple example:

```
julia> using Statistics

julia> using DataFrames

julia> mx = rand(10000, 100);

julia> df = DataFrame(mx, :auto);

julia> mycor(df) = [cor(x, y) for x in eachcol(df), y in eachcol(df)]
mycor (generic function with 1 method)

julia> mycor(mx); @time mycor(mx);
  0.062812 seconds (2 allocations: 78.203 KiB)

julia> mycor(df); @time mycor(df);
  0.067229 seconds (60.10 k allocations: 1.911 MiB)
```

As you can see the performance is pretty similar.

However if you really need to go type-stable starting from a data frame it is
very easy just use the `Tables.columntable` function:

```
julia> using DataFrames

julia> df = DataFrame(a=[1,2], b=[3,4])
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4

julia> tbl = Tables.columntable(df)
(a = [1, 2], b = [3, 4])
```

and you move to a type-stable `NamedTuple` (no copying of columns is performed),
and assuming the table is not very wide this conversion is cheap. Of course it
is equally easy to go back:

```
julia> DataFrame(tbl, copycols=false)
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4
```

I used `copycols=false` to avoid copying of the columns --- so again the
process is cheap; note, however, that by default the `DataFrame` constructor
copies columns passed to it as it is a safer approach.

So when does type stability matter most? The answer is when you need to iterate
rows of a data frame (especially as in a typical data frame there are many times
more rows than columns). Note that iterating a data frame cell by cell (i.e.
extracting single values from it) is essentially the same case.

Why it is a problem? Well --- the issue is that when you extract a row from a
`DataFrame` it is still type unstable `DataFrameRow` object (the reason is
similar to what we have discussed above --- if you would have a very wide data
frame you would have huge compilation cost if you wanted to make a row of a data
frame type-stable). However, this time we are paying type-instability cost for
each row of a data frame --- and this can indeed get expensive.

Here is an example:

```
julia> using DataFrames

julia> df = DataFrame(rand(10^6, 2), :auto);

julia> @time [row[1] for row in eachrow(df)];
  0.227021 seconds (4.19 M allocations: 78.453 MiB, 24.12% gc time)

julia> @time [row[1] for row in eachrow(df)];
  0.279041 seconds (4.11 M allocations: 73.963 MiB, 49.28% gc time)
```

Is this timing bad? Well, compare it to:

```
julia> tbl = Tables.columntable(df);

julia> @time [row[1] for row in Tables.rows(tbl)];
  0.049375 seconds (66.81 k allocations: 11.309 MiB, 21.53% gc time)

julia> @time [row[1] for row in Tables.rows(tbl)];
  0.029195 seconds (44.86 k allocations: 9.987 MiB)
```

so indeed it looks bad. Actually there is a shorthand for the operation we just
did:

```
julia> @time [row[1] for row in Tables.namedtupleiterator(df)];
  0.232059 seconds (1.25 M allocations: 68.209 MiB, 6.59% gc time)

julia> @time [row[1] for row in Tables.namedtupleiterator(df)];
  0.042947 seconds (45.12 k allocations: 9.925 MiB)
```

So the sort conclusion is --- you should expect to be most significantly
influenced by the performance degradation due to type-instability of `DataFrame`
if you want to iterate rows (or access single cells). However, in such cases you
can use e.g. `Tables.columntable` or `Tables.namedtupleiterator` to easily
switch to type-stable mode (there is also `@eachrow` macro in
[DataFramesMeta.jl][dfm]).

As you saw above this was the approach that allowed to resolve the
[question][so] asked on Stack Overflow.

# Conclusions

The crucial take-away from this discussion is that it is easy to switch between
type-stable and type-unstable modes when using DataFrames.jl.

Therefore, especially in generic code that is designed to work with arbitrary
data we do not know in advance (that e.g. can be wide or can constantly change
its schema), a practical pattern is to:
* work with `DataFrame` most of the time (and accept paying a small cost of its
  type instability)
* when you really need performance (and especially when you need to iterate rows
  of the data frame) temporarily switch to type-stable mode (this is what
  `select` does internally when you use the `ByRow` wrapper on a function)

[df]: https://github.com/JuliaData/DataFrames.jl
[so]: https://stackoverflow.com/questions/65584387/julia-csv-write-very-memory-inefficient
[perf]: https://docs.julialang.org/en/v1/manual/performance-tips/#man-performance-abstract-container
[dfm]: https://github.com/JuliaData/DataFramesMeta.jl
