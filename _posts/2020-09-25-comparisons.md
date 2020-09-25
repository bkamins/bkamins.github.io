---
layout: post
title:  "DataFrames.jl vs Pandas, dplyr, and Stata"
date:   2020-09-25 15:13:23 +0200
categories: julialang
---

# New content in DataFrames.jl documentation

Many people moving to DataFrames.jl from other data-management ecosystems are
interested in learning how to map their favorite code patterns to Julia.

It was a long standing issue. Fortunately recently thanks to the efforts of
[Matthieu Gomez][mg] and [Tom Kwong][tk] (with the usual major support from
[Peter Deffebach][pd] and [Milan Bouchet-Valat][mbv], and a few other
contributors) we finally have a section in the manual on [comparisons][loc]
against Pandas, dplyr, and Stata.

In parallel [Tom Kwong][tk] also prepared DataFrames.jl [cheat sheet][cs] which
excellently shows key functionalities that we currently provide.

We all hope that these materials will be useful for people to get started with
DataFrames.jl. If you would like to see some additional content in the
[comparisons][loc] section of the DataFrames.jl manual -- please do not hesitate
to open an issue or pull request.

# Lessons learned

As an after-word let me comment that getting dplyr and Stata material was much
smoother than Pandas. It is also reflected in the volume of the material covered
(though probably dplyr and Stata coverage could be improved). The main reason is
that Pandas differs many more ways from DataFrames.jl than dplyr or Stata.
A few of the notable differences are:
- the type of return value from `loc` function in Pandas depends on the value
  (not only the type) of its arguments;
- `0` based indexing (Pandas) vs `1` based indexing (DataFrames.jl);
- `NaN` in Pandas is treated as `missing` in Julia, but is skipped by default
  as opposed to Julia, where you have to be explicit;
- Pandas has `inplace` argument to functions while in Julia we have functions
  with and without `!` to distinguish between non-mutating and mutating operations;
- Pandas provides row index, while in DataFrames.jl you need a separate column
  (or columns) in a `DataFrame` to hold it and later run a `groupby` function on
  them to get an efficient row-lookup functionality through `GroupedDataFrame`
  object (note, in particular, that in this way you can have many different row
  indexing column sets to for the same data frame).

[loc]: https://juliadata.github.io/DataFrames.jl/latest/man/comparisons/
[mg]: https://github.com/matthieugomez
[tk]: https://github.com/tk3369
[pd]: https://github.com/pdeffebach
[mbv]: https://github.com/nalimilan
[cs]: https://ahsmart.com/assets/pages/data-wrangling-with-data-frames-jl-cheat-sheet/DataFramesCheatSheet_v0.21_rev3.pdf
