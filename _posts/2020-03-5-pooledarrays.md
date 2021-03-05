---
layout: post
title:  "What is new in PooledArrays.jl?"
date:   2021-03-05 12:44:57 +0200
categories: julialang
---

# Introduction

Recently PooledArrays.jl 1.2.1 has been released. The most significant change
since 1.0 release is an improvement of performance of basic operations:
`getindex`, `copy`, `copy!`, and `copyto!`. The effect of the change is
especially significant for `PooledArray`s that have large pools. This change, is
one of the steps towards making Julia run fast in [Database-like ops
benchmark][benchmark] for joins.

Let me start with the examples and then I will comment on the internals.
The post was tested under Julia 1.6.0-rc1.

# The Benchmarks

I will just share a recordng of a Julia session doing the benchmarks. We start
with PooledArrays.jl 1.0.0:

```
(@v1.6) pkg> activate .
  Activating new environment at `~/Project.toml`

(bkamins) pkg> add PooledArrays@1.0
   Resolving package versions...
    Updating `~/Project.toml`
  [2dfb63ee] + PooledArrays v1.0.0
    Updating `~/Manifest.toml`
  [9a962f9c] + DataAPI v1.6.0
  [2dfb63ee] + PooledArrays v1.0.0
  Progress [========================================>]  1/1
1 dependency successfully precompiled in 2 seconds (1 already precompiled)

julia> using BenchmarkTools, PooledArrays

julia> x = PooledArray(string.(1:10^6));

julia> @benchmark copy($x)
BenchmarkTools.Trial:
  memory estimate:  37.44 MiB
  allocs estimate:  13
  --------------
  minimum time:     16.109 ms (0.00% GC)
  median time:      17.820 ms (0.00% GC)
  mean time:        29.851 ms (40.25% GC)
  maximum time:     175.307 ms (88.17% GC)
  --------------
  samples:          168
  evals/sample:     1

julia> @benchmark $x[1:1]
BenchmarkTools.Trial:
  memory estimate:  33.63 MiB
  allocs estimate:  12
  --------------
  minimum time:     15.334 ms (0.00% GC)
  median time:      16.929 ms (0.00% GC)
  mean time:        30.654 ms (44.06% GC)
  maximum time:     188.496 ms (89.90% GC)
  --------------
  samples:          164
  evals/sample:     1
```

In order to assess if the timings are good or bad let us do the same operations
using a plain `Vector{String}`:

```
julia> x = string.(1:10^6);

julia> @benchmark copy($x)
BenchmarkTools.Trial:
  memory estimate:  7.63 MiB
  allocs estimate:  2
  --------------
  minimum time:     395.687 μs (0.00% GC)
  median time:      441.842 μs (0.00% GC)
  mean time:        757.598 μs (38.52% GC)
  maximum time:     6.194 ms (90.51% GC)
  --------------
  samples:          6598
  evals/sample:     1

julia> @benchmark $x[1:1]
BenchmarkTools.Trial:
  memory estimate:  96 bytes
  allocs estimate:  1
  --------------
  minimum time:     25.310 ns (0.00% GC)
  median time:      26.013 ns (0.00% GC)
  mean time:        34.384 ns (8.88% GC)
  maximum time:     2.433 μs (98.05% GC)
  --------------
  samples:          10000
  evals/sample:     992
```

As you can see things are really bad in PooledArrays.jl. Now start a fresh Julia
session and install 1.2.1 release of the package.

```
(bkamins) pkg> activate .
  Activating environment at `~/Project.toml`

(bkamins) pkg> add PooledArrays@1.2
   Resolving package versions...
    Updating `~/Project.toml`
  [2dfb63ee] ↑ PooledArrays v1.0.0 ⇒ v1.2.1
    Updating `~/Manifest.toml`
  [2dfb63ee] ↑ PooledArrays v1.0.0 ⇒ v1.2.1
  [9fa8497b] + Future
  [9a3f8284] + Random
  [9e88b42a] + Serialization

julia> using BenchmarkTools, PooledArrays

julia> x = PooledArray(string.(1:10^6));

julia> @benchmark copy($x)
BenchmarkTools.Trial:
  memory estimate:  3.81 MiB
  allocs estimate:  4
  --------------
  minimum time:     893.391 μs (0.00% GC)
  median time:      948.740 μs (0.00% GC)
  mean time:        1.009 ms (5.08% GC)
  maximum time:     103.288 ms (98.55% GC)
  --------------
  samples:          4953
  evals/sample:     1

julia> @benchmark $x[1:1]
BenchmarkTools.Trial:
  memory estimate:  160 bytes
  allocs estimate:  3
  --------------
  minimum time:     47.806 ns (0.00% GC)
  median time:      53.053 ns (0.00% GC)
  mean time:        184.150 ns (44.04% GC)
  maximum time:     55.779 μs (68.58% GC)
  --------------
  samples:          10000
  evals/sample:     976
```

This looks better. Still you have to pay some cost over a `Vector{Sting}` but it
is much smaller (the cost is due to the fact that `PooledArray` constructor
performs some consistency checks of passed data to ensure extra safety).

You can expect that other operations that take a `PooledArray` or its view and
produce a `PooledArray` (like or `copyto!`) to experience similar
speedups.

So, what has changed between 1.0 and 1.2 release of PooledArrays.jl?

# The Internals

In order to understand why the speedups were possible one needs to understand
first why the original code was slow. The reason is that `PooledArray` struct
contains three key fields:

* `refs`: a vector of integer references (levels) of a `PooledArray`;
* `pool`: a vector giving a mapping from references to actual values;
* `invpool`: a dictionary providing a reverse mapping - from values to references.

As you can see this data structure is quite heavy if the size of `pool` relative
to the size of the `PooledArray` is large. In the example above I have shown an
extreme case where they were equal. But even if `pool` has e.g. 10% of size of
the `PooledArray` the cost is noticeable.

In PooledArrays.jl 1.0 all thee fields `refs`, `pool` and `invpool` were always
copied when a new `PooledArray` was created. This was expensive. What
PooeldArrays.jl 1.2 introduces is a well known from R and often asked about in
Julia *copy on write* behavior. What we now do is that we only copy `refs`. The
`pool` and `invpool` fields are shared across `PooledArrays` as this is a safe
thing to do as long as you are not changing the set of levels in the `pool`.

So where does the *copy on write* happen? PooledArrays.jl is now aware if you
share `pool` and `invpool` across several arrays and if this is the case and you
add levels to `PooledArray` then `pool` and `invpool` get copied. So essentially
we have a lazy mechanism that copies them only if needed.

One could ask why do we copy `pool` and `invpool` at all. One could just keep
sharing them without having to pay the cost of copying at all. The decision was
guided here by two considerations:

* in practice new levels are added to `PooledArray` quite rarely (you mostly do
  it when constructing an initial source `PooledArray`);
* it is safer to copy the `pool` and `invpool` in consideration of potential
  multi-threaded usages of `PooledArray` (where many tricky corner cases can
  happen).

# Conclusions

The main take away is that you can expect your code using PooledArrays.jl to be
much faster since 1.2 release.

Before I finish let me comment on one important feature of `PooledArray` object
to keep in mind in the multi-threaded applications I have mentioned above.

It is currently not thread safe to add new levels to the `pool`
of `PooledArray`. So the rule is: if you add levels to `PooledArray` make sure
you are not performing any other operations on it in other threads.

However, you can safely perform any operations in multi-threaded context that do
not change the `pool`. So, in particular (barring standard considerations of
correct multi-threaded code), you are allowed to use `setindex!` on
`PooledArray` as long as you do not add new levels.

[benchmark]: https://h2oai.github.io/db-benchmark/
