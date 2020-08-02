---
layout: post
title:  "JuliaCon2020: conclusions for DataFrames.jl"
date:   2020-08-02 10:13:23 +0200
categories: julialang
---

# Introduction

[JuliaCon 2020][juliacon] was a great event. It has opened my eyes to many
fantastic things that happen in the ecosystem and for sure I will write at least
one more post with the summary of my take-aways.

In this post I want to summarize my conclusions from the discussions around
DataFrames.jl and related ecosystem. In particular
[Julia & Data: An Evolving Ecosystem][bof] BOF was a great gathering to discuss
the future directions. Thank you all who participated.

# The survey

Before the BOF I have made a [quick survey][survey] to check with the community
where the development effort of the DataFrames.jl should focus on. While many
topics are were found important the top issue is performance, with a particular
emphasis on adding treading support and improving the performance of joins
(which are not sub-par in comparison to aggregation).

Therefore this will be the area that I plan to focus most of the development
effort in the short term (of course all contributors are encouraged to open
issues/PRs in all potential areas of improvement and they will be handled).

In particular, regarding the performance, I have opened an [issue][joins]
related to joins. Everyone is welcome to comment there with thoughts how things
could be improved. I believe that the current major reason of bad performance
we have is that we have only one join algorithm that treats left and right
joined data frame differently which in some cases leads to severe performance
bottlenecks.

To give an example of the problem consider the following timings in
DataFrames.jl 0.21.4:
```
julia> using DataFrames, BenchmarkTools

julia> df1 = DataFrame(id=1:10^6, x1=1:10^6);

julia> df2 = DataFrame(id=1:10^3, x2=1:10^3);

julia> @benchmark innerjoin($df1, $df2, on=:id)
BenchmarkTools.Trial:
  memory estimate:  76.41 MiB
  allocs estimate:  1999686
  --------------
  minimum time:     215.176 ms (0.00% GC)
  median time:      229.573 ms (0.00% GC)
  mean time:        228.554 ms (2.15% GC)
  maximum time:     241.558 ms (6.11% GC)
  --------------
  samples:          22
  evals/sample:     1

julia> @benchmark innerjoin($df2, $df1, on=:id)
BenchmarkTools.Trial:
  memory estimate:  61.54 MiB
  allocs estimate:  1692
  --------------
  minimum time:     115.506 ms (0.00% GC)
  median time:      122.250 ms (0.00% GC)
  mean time:        123.309 ms (0.29% GC)
  maximum time:     133.132 ms (0.63% GC)
  --------------
  samples:          41
  evals/sample:     1

julia> df2 = DataFrame(id=1:10, x2=1:10);

julia> @benchmark innerjoin($df1, $df2, on=:id)
BenchmarkTools.Trial:
  memory estimate:  76.30 MiB
  allocs estimate:  1999673
  --------------
  minimum time:     55.207 ms (0.00% GC)
  median time:      69.426 ms (0.00% GC)
  mean time:        68.312 ms (7.17% GC)
  maximum time:     81.201 ms (17.13% GC)
  --------------
  samples:          74
  evals/sample:     1

julia> @benchmark innerjoin($df2, $df1, on=:id)
BenchmarkTools.Trial:
  memory estimate:  61.42 MiB
  allocs estimate:  201
  --------------
  minimum time:     117.681 ms (0.00% GC)
  median time:      121.471 ms (0.00% GC)
  mean time:        122.413 ms (0.26% GC)
  maximum time:     131.358 ms (0.64% GC)
  --------------
  samples:          41
  evals/sample:     1
```
As you can see the order of arguments matters and influences the performance
in a non-trivial way. Also a challenge for managing deprecation process when
we change the implementation is that the row order of the result of joins
depends on the order in which we passed data frames for joining (and it is
possible that faster algorithms will produce different row orderings of the
resulting joined table).

# The ecosystem

For the things that happen around DataFrames.jl I would like to highlight two
out of many interesting efforts:
* It can be expected that soon Apache Arrow will have a full support in Julia.
  This is a super important thing I think and when we have it it will be much
  easier to use Julia in enterprise applications.
* There is a significant amount of work done to make DataFramesMeta.jl even more
  user friendly than it is now. I am really looking forward to it, as then
  in DataFrames.jl we will be able to concentrate on the internals and making
  things fast, and the *bells and whistles* that make daily work with data
  frames smooth will be provided in DataFramesMeta.jl.

The last point relates to the tension around how much DataFrames.jl should
follow a Unix convention *do one thing, and do it right* vs the approach where
we would like to see it as a *Swiss Army knife* for all tabular data
manipulation tasks. There are pros and cons of both approaches and soon I will
write a separate post explaining my current thinking about this issue.

# What is next?

In the conclusion I would like to write what to expect in DataFrames.jl
development in the coming months. Please consider it as my personal view as the
community might disagree:
* In 1-2 months we shall have a 0.22 release that will introduce new breaking
  changes.
* The 1.0 release will probably happen in the early 2021 with a major target
  that it would incorporate performance improvement fixes.

Now what is the rationale behind this:
* In 0.21 there were found several corner cases of functionality that we should
  change (like making sure `transform` does not reorder existing columns and
  properly handles data frames with zero rows, see this [PR][pr] for details).
  So we need a minor release relatively soon.
* When introducing performance fixes we might need to change how rows of the
  the requested operations are ordered (e.g. in joins). This means that making
  performance improvements might introduce changes that will be breaking.
  And we should not expect to fix all performance issues (e.g. providing a
  decent threading support) sooner than in the end of 2020 and then such things
  require detailed tests, as usually the algorithms that are fast are complex.

Having said that I am committed to the contract we have stated when releasing
0.21 version that **we do not want to be breaking after this release**.
Therefore, as users you can expect that this promise is taken very seriously and
if we break something there is a strong reason for it. In particular I very
strongly want to avoid API breakage (we rather can extend it, but not break
things that already worked). However, things that might be broken, as you see
from this post, is what is the column or row order of the result of some
operations (so in a sense --- from a data base perspective these things mostly
would not be considered as breaking, but as DataFrames.jl is seen as a
matrix-like structure by some operations in user's code row and column order
matters).

[juliacon]: https://juliacon.org/2020/
[bof]: https://live.juliacon.org/talk/CA3SET
[survey]: https://discourse.julialang.org/t/dataframes-jl-development-survey/44022
[joins]: https://github.com/JuliaData/DataFrames.jl/issues/2340
[pr]: https://github.com/JuliaData/DataFrames.jl/pull/2324
