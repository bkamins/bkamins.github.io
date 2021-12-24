---
layout: post
title:  "News features in DataFrames.jl 1.3: part 3"
date:   2021-12-24 06:13:31 +0200
categories: julialang
---

# Introduction

This post continues the presentation of new features added in [DataFrames.jl]
[df] 1.3.0. This time I will discuss what is new in indexing syntax.

The post was written under Julia 1.7.0 and DataFrames.jl 1.3.1.
When running the examples use the `--depwarn=yes` option when starting Julia.

# Adding columns in views

Since DataFrames.jl 1.3 a long requested feature to allow adding columns to
views has been added. As, in general, in a view you can reorder and/or drop
columns this feature is only allowed if a view was created with `:` as column
selector (remember, that when using `:` as column selector a view will *always*
reflect the list of columns of its parent `DataFrame`). Here is an example:

```
julia> using DataFrames

julia> df = DataFrame(a=1:3)
3×1 DataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │     1
   2 │     2
   3 │     3

julia> dfv = @view df[[1,3], :]
2×1 SubDataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │     1
   2 │     3

julia> dfv[:, :b] = 4:5
4:5

julia> dfv
2×2 SubDataFrame
 Row │ a      b
     │ Int64  Int64?
─────┼───────────────
   1 │     1       4
   2 │     3       5

julia> df
3×2 DataFrame
 Row │ a      b
     │ Int64  Int64?
─────┼────────────────
   1 │     1        4
   2 │     2  missing
   3 │     3        5
```

Note that in column `:b` in `df` in filtered out rows `missing` value was
placed.

As noted creating new columns is not allowed if other column selector than `:`
is passed when creating a view:

```
julia> dfv = @view df[[1,3], 1:2]
2×2 SubDataFrame
 Row │ a      b
     │ Int64  Int64?
─────┼───────────────
   1 │     1       4
   2 │     3       5

julia> dfv[:, :c] = 4:5
ERROR: ArgumentError: creating new columns in a SubDataFrame that subsets columns of its parent data frame is disallowed
```

Additionally it is allowed to replace columns in a view when `!` selector is
used (here it works for any view as we are not creating new columns):

```
julia> dfv.a = ["111", "113"]
2-element Vector{String}:
 "111"
 "113"

julia> df
3×2 DataFrame
 Row │ a    b
     │ Any  Int64?
─────┼──────────────
   1 │ 111        4
   2 │ 2    missing
   3 │ 113        5

```

As you can see the values that were present in filtered-out rows are retained.
If the new values have type not allowed in the current element type of the
column an appropriate type promotion is performed. Here is another example:

```
julia> df = DataFrame(a=1:3, b=4:6)
3×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      4
   2 │     2      5
   3 │     3      6

julia> dfv = @view df[[1,3], :]
2×2 SubDataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      4
   2 │     3      6

julia> dfv.a = 11.0:12.0
11.0:1.0:12.0

julia> dfv.b = 'a':'b'
'a':1:'b'

julia> df
3×2 DataFrame
 Row │ a        b
     │ Float64  Any
─────┼──────────────
   1 │    11.0  a
   2 │     2.0  5
   3 │    12.0  b
```

# Why does adding columns in views matter?

The huge benefit of allowing adding columns in views is as follows: we can make
all standard functions like `insertols!`, `select!`, `transform!` work on
`SubDataFrames`. This is very useful if you want to perform some operation only
under some condition. Here is a simple example:

```
julia> df = DataFrame(x = -1:0.2:1)
11×1 DataFrame
 Row │ x
     │ Float64
─────┼─────────
   1 │    -1.0
   2 │    -0.8
   3 │    -0.6
   4 │    -0.4
   5 │    -0.2
   6 │     0.0
   7 │     0.2
   8 │     0.4
   9 │     0.6
  10 │     0.8
  11 │     1.0

julia> transform!(subset(df, :x => ByRow(>(0)), view=true), :x => ByRow(log))
5×2 SubDataFrame
 Row │ x        x_log
     │ Float64  Float64?
─────┼────────────────────
   1 │     0.2  -1.60944
   2 │     0.4  -0.916291
   3 │     0.6  -0.510826
   4 │     0.8  -0.223144
   5 │     1.0   0.0

julia> df
11×2 DataFrame
 Row │ x        x_log
     │ Float64  Float64?
─────┼─────────────────────────
   1 │    -1.0  missing
   2 │    -0.8  missing
   3 │    -0.6  missing
   4 │    -0.4  missing
   5 │    -0.2  missing
   6 │     0.0  missing
   7 │     0.2       -1.60944
   8 │     0.4       -0.916291
   9 │     0.6       -0.510826
  10 │     0.8       -0.223144
  11 │     1.0        0.0
```

In DataFrames.jl you have to do this in two steps: `subset` to a view and then
`transform!`. However, I hope that [DataFramesMeta.jl][dfm1] and
[DataFrameMacros][dfm2] packages in the coming releases will provide a nicer
syntax for this, allowing to combine transformation and filtering in one step.

# Hard deprecation period for broadcasted assignment

Since Julia 1.7 is out a long missing feature is now available.
The feature is that it is allowed to add new columns to a data frame using
the broadcasting assignment with `setproperty`:

```
julia> df = DataFrame(a=1:3)
3×1 DataFrame
 Row │ a
     │ Int64
─────┼───────
   1 │     1
   2 │     2
   3 │     3

julia> df.b .= 1
3-element Vector{Int64}:
 1
 1
 1

julia> df
3×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      1
   3 │     3      1
```

Also views are supported the way we have described earlier:

```
julia> dfv = view(df, [1, 3], :)
2×2 SubDataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     3      1

julia> dfv.c .= 2
2-element Vector{Int64}:
 2
 2

julia> df
3×3 DataFrame
 Row │ a      b      c
     │ Int64  Int64  Int64?
─────┼───────────────────────
   1 │     1      1        2
   2 │     2      1  missing
   3 │     3      1        2
```

In essence using `setproperty` is made almost the same as using the `!` row
selector and assignment. Why do I say *almost*? The reason is that the only
place where it is inconsistent is broadcasted assignment to an existing column:

```
julia> df.a .= 10
┌ Warning: In the 1.4 release of DataFrames.jl this operation will allocate a new column instead of performing an in-place assignment. To perform an in-place assignment use `df[:, col] .= ...` instead.
│   caller = top-level scope at REPL[8]:1
└ @ Core REPL[8]:1
3-element Vector{Int64}:
 10
 10
 10
```

As you can see in the warning message this inconsistency (that was known and
discussed for some time already) will be fixed in DataFrames.jl 1.4.
We have been waiting with this change for several releases in order to clearly
inform users about this fix in advance.

This change is not only about consistency, but also to make sure we do not
perform accidental conversions where users most likely do not expect them:

```
julia> df2 = DataFrame(x = 'a':'c')
3×1 DataFrame
 Row │ x
     │ Char
─────┼──────
   1 │ a
   2 │ b
   3 │ c

julia> df2.x .= 104
┌ Warning: In the 1.4 release of DataFrames.jl this operation will allocate a new column instead of performing an in-place assignment. To perform an in-place assignment use `df[:, col] .= ...` instead.
│   caller = top-level scope at REPL[18]:1
└ @ Core REPL[18]:1
3-element Vector{Char}:
 'h': ASCII/Unicode U+0068 (category Ll: Letter, lowercase)
 'h': ASCII/Unicode U+0068 (category Ll: Letter, lowercase)
 'h': ASCII/Unicode U+0068 (category Ll: Letter, lowercase)

julia> df2
3×1 DataFrame
 Row │ x
     │ Char
─────┼──────
   1 │ h
   2 │ h
   3 │ h
```

In the future, when DataFrames.jl 1.4 is released, instead you will get a data
frame with column `:x` having element type `Int` and storing three `104`
values.

# Conclusions

The changes I have described today are not something that a new person starts to
use on the first day of working with DataFrames.jl. However, after one learns
the basics more and more advanced queries are needed in practice. Improvements
in functionality and consistency of the design of the core of indexing
mechanisms in DataFrames.jl are hopefully going to make these complex
requirements easier to meet.

[df]: https://github.com/JuliaData/DataFrames.jl
[dfm1]: https://github.com/JuliaData/DataFramesMeta.jl
[dfm2]: https://github.com/jkrumbiegel/DataFrameMacros.jl
