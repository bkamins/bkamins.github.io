---
layout: post
title:  "Tutorials for DataFrames.jl release 0.21. Part I"
date:   2020-05-25 04:21:47 +0200
categories: julialang
---

# DataFrames.jl release 0.21

DataFrames.jl version 0.21 was a major release that introduced a number of
significant changes to DataFrames.jl API. The [list is long][release], so
I briefly summarize here he most significant things in terms of functionality:

* we now allow to select columns using strings (selection using `Symbol`s is
  still allowed);
* a completely new design of API for working with columns of a data frame or
  grouped data frame, covered by `select`/`select!`, `transform`/`transform!`,
  and `combine` functions; it is consistent (so you learn it once and reuse
  everywhere), more flexible, and has a better performance than the old one;
  in particular two wrappers `ByRow` and `AsTable` have been added to API;
* major enhancements to `push!` and `append!`, which allow an easy way to
  digest heterogeneous data (varying element types, varying column sets)
  into a data frame;
* `GroupedDataFrame` now supports a fast lookup by grouping columns (so making
  a `GroupedDataFrame` can be now seen as adding an index to a data frame)
* `filter`/`filter!` are now fast using `Pair`-interface;
* rules for pseudo-broadcasting (spreading single observations across multiple
  rows) have been established and are consistently applied in all methods that
  allow this operation.

All these changes combined mean that now all operations on data frames can be
expressed via function chaining (and you have a full control if you want to
make copies or perform operations in-place). There are many users who like this
style of expressing transformations made on data. If you want to go this way,
then probably you should consider learning one of the packages that makes
it easier to work with `|>` operator. There are many excellent alternatives
in the Julia ecosystem. Let me mention two [Pipe.jl][pipe] (easier) and
[Underscores.jl][underscores] (more powerful, but harder to master).

After the release I got several questions about showing how things work in
practice. Therefore in this post I list tutorials that are currently available
and have been updated to show how DataFrames.jl v0.21 works.

In the *Part II* post (that I plan to prepare next week) I will show some new
material that was prepared under DataFrames.jl v0.21.

# Tutorials for release 0.21

There are four sources of information about the functionality of DataFrames.jl
0.21 that you can check out (and I maintain them so that they should be
up to date):

1. An official [DataFrames.jl Manual][manual].
2. A notebook-based [DataFrames.jl Tutorial][tutorial].
3. Video materials at [JuliaAcademy][juliaacademy].
4. Recently updated the materials about DataFrames.jl that I have presented
   during [JuliaCon2019 workshop][juliacon2019]. You will be able to find
   there two notebooks that include worked examples how you can process
   real-life data sets.

I hope these materials will be useful for exploring the latest release
of DataFrames.jl!

[release]: https://github.com/JuliaData/DataFrames.jl/releases/tag/v0.21.0
[manual]: https://juliadata.github.io/DataFrames.jl/stable/
[tutorial]: https://github.com/bkamins/Julia-DataFrames-Tutorial/
[juliaacademy]: https://juliaacademy.com/p/introduction-to-dataframes-jl
[juliacon2019]: https://github.com/bkamins/JuliaCon2019-DataFrames-Tutorial
[underscores]: https://github.com/c42f/Underscores.jl
[pipe]: https://github.com/oxinabox/Pipe.jl
