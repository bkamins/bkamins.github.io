---
layout: post
title:  "DataFrames.jl user's corner: filtering performance"
date:   2021-08-20 06:14:31 +0200
categories: julialang
---

# Introduction

Today I write about a topic that was proposed by [DataFrames.jl][df] user.
The question was related the performance of the `filter` function. I hope
this will be useful to people who start using Julia and [DataFrames.jl][df].

In this post I will show several ways to perform the same row sub-setting
operation using the `filter` function on a data frame and compare their performance.

The post was written under Julia 1.6.1 and DataFrames.jl 1.2.2.

# The `filter` performance test

Assume that we have some data on financial transactions. The data frame
storing them has two columns: `:from` and `:to` storing account numbers between
which the money transfer was performed.

Let us first create some random data frame with this structure and 100,000 rows:

```
julia> using DataFrames

julia> using Random

julia> Random.seed!(1);

julia> transfers = DataFrame(from=rand(1:10^5, 10^5), to=rand(1:10^5, 10^5))
100000×2 DataFrame
    Row │ from   to
        │ Int64  Int64
────────┼──────────────
      1 │ 65190  19016
      2 │ 84951  66344
   ⋮    │   ⋮      ⋮
  99999 │ 21480  54671
 100000 │ 86787  30151
     99996 rows omitted

```

Now the task is to find all transactions from accounts that received some
transfer to them.

Here is a first attempt to do it (I am showing everywhere the timing of a second
run of a specific `filter` operation, so the compilation time is this related
to only this specific call; also I am showing the output of the call to makes
sure that the result is the same):

```
julia> @time filter(x -> x.from in transfers.to, transfers)
  3.627735 seconds (621.32 k allocations: 23.318 MiB, 3.03% compilation time)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

The code above is the most natural `filter` call following the Julia Base syntax.
However, we can read in the documentation of `filter` in [DataFrames.jl][df] that
it is faster to pass the column we want to operate on (in this case `:filter`)
using the `=>` syntax. In this way the operation can be made more efficient:

```
julia> @time filter(:from => in(transfers.to), transfers)
  2.543524 seconds (28 allocations: 1.463 MiB)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

As you can see it is somewhat faster. There is no compilation as `in(transfers.to)`
has to be compiled only once, see:

```
julia> in(transfers.to) # this creates a callable type
(::Base.Fix2{typeof(in), Vector{Int64}}) (generic function with 1 method)
```

and we have a small number of allocations as the code is type stable.

Note that the code below uses the same idea:

```
julia> @time filter(:from => x -> x in transfers.to, transfers)
  2.663027 seconds (458.82 k allocations: 22.803 MiB, 5.28% compilation time)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

but the difference is that as opposed to `in(transfers.to)` the the code
`x -> x in transfers.to` creates a new anonymous function each time it is called,
which triggers compilation.

Can we do better? An experienced Julia programmer willre immediately comment
that it would be more efficient to perform a lookup in a `Set` and not in a `Vector`.

Therefore our next attempt is:

```
julia> @time filter(:from => in(Set(transfers.to)), transfers)
  0.007884 seconds (36 allocations: 3.714 MiB)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

This time it is super fast. Let us dissect the timing:

```
julia> @time s = Set(transfers.to)
  0.004668 seconds (8 allocations: 2.251 MiB)
Set{Int64} with 63246 elements:
  92533
  76914
  45120
  1703
  37100
  ⋮

julia> @time filter(:from => in(s), transfers) # remember it is a second run timing
  0.004394 seconds (28 allocations: 1.463 MiB)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

and we can see that roughly similar time is spent in constructing of the `Set`
as in the filtering later.

Note, however, that it is essential that we use the `in(Set(transfers.to))`
construct, as it is evaluated only once.

Here is what happens if we tried to use the `Set` in our original `filter` call:

```
julia> @time filter(x -> x.from in Set(transfers.to), transfers)
272.379325 seconds (1.82 M allocations: 219.832 GiB, 1.12% gc time, 0.08% compilation time)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

This is very bad, as we call `Set` on each row of our `transfers` data set.
Let us try reusing `s` variable we have created above:

```
julia> @time filter(x -> x.from in s, transfers)
  0.140256 seconds (620.53 k allocations: 23.268 MiB, 90.38% compilation time)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

Now it is already reasonably good. As a final test we switch `s` to be a constant:

```
julia> const cs = s;

julia> @time filter(x -> x.from in cs, transfers) # remember it is a second run timing
  0.122081 seconds (619.03 k allocations: 23.060 MiB, 86.92% compilation time)
63154×2 DataFrame
   Row │ from   to
       │ Int64  Int64
───────┼──────────────
     1 │ 84951  66344
     2 │ 11514   5399
   ⋮   │   ⋮      ⋮
 63153 │ 91605  34813
 63154 │ 86787  30151
    63150 rows omitted
```

And we see that it is a bit better, but not much, as we are in type unstable mode anyway.

# Conclusions

I would summarize the key general observations as follows:
* when using `in` it is recommended to pass a `Set` as a collection in which
  we perform test if the test is executed many times;
* the `filter(predicate, data_frame)` style is type unstable, and typically a faster
  alternative is `filter(column => predicate, data_frame)`;
* `in(collection)` a callable object that can be used later to test
  for inclusion of its argument in `collection`; as a side benefit it is compiled
  only once per type of `collection` which reduces compilation latency;
* one needs to understand how Julia would execute code; following general
  recommendations blindly as in the `filter(x -> x.from in Set(transfers.to), transfers)`
  example can lead to extremely bad performance.

[df]: https://github.com/JuliaData/DataFrames.jl
