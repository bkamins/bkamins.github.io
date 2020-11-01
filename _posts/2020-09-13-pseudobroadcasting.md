---
layout: post
title:  "Pseudo broadcasting in DataFrames.jl"
date:   2020-09-13 07:10:42 +0200
categories: julialang
---

# Introduction

The descriptions I present in this post relate to Julia 1.5 and DataFrames.jl 0.21.

In Julia base the standard rule to do [broadcasting][broadcasting] is to use `.`.
For example, as opposed to R this operation fails:

```
julia> x = [1, 2, 3, 4]
4-element Array{Int64,1}:
 1
 2
 3
 4

julia> x[3:4] = 0
ERROR: ArgumentError: indexed assignment with a single value to many locations is not supported; perhaps use broadcasting `.=` instead?
```

instead you have to write:

```
julia> x[3:4] .= 0
2-element view(::Array{Int64,1}, 3:4) with eltype Int64:
 0
 0

julia> x
4-element Array{Int64,1}:
 1
 2
 0
 0
```

Similar syntax is fully supported in DataFrames.jl, e.g.:
```
julia> using DataFrames

julia> df = DataFrame(zeros(4,5))
4×5 DataFrame
│ Row │ x1      │ x2      │ x3      │ x4      │ x5      │
│     │ Float64 │ Float64 │ Float64 │ Float64 │ Float64 │
├─────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ 1   │ 0.0     │ 0.0     │ 0.0     │ 0.0     │ 0.0     │
│ 2   │ 0.0     │ 0.0     │ 0.0     │ 0.0     │ 0.0     │
│ 3   │ 0.0     │ 0.0     │ 0.0     │ 0.0     │ 0.0     │
│ 4   │ 0.0     │ 0.0     │ 0.0     │ 0.0     │ 0.0     │

julia> df[2:3, 2:4] .= 1
2×3 SubDataFrame
│ Row │ x2      │ x3      │ x4      │
│     │ Float64 │ Float64 │ Float64 │
├─────┼─────────┼─────────┼─────────┤
│ 1   │ 1.0     │ 1.0     │ 1.0     │
│ 2   │ 1.0     │ 1.0     │ 1.0     │

julia> df
4×5 DataFrame
│ Row │ x1      │ x2      │ x3      │ x4      │ x5      │
│     │ Float64 │ Float64 │ Float64 │ Float64 │ Float64 │
├─────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ 1   │ 0.0     │ 0.0     │ 0.0     │ 0.0     │ 0.0     │
│ 2   │ 0.0     │ 1.0     │ 1.0     │ 1.0     │ 0.0     │
│ 3   │ 0.0     │ 1.0     │ 1.0     │ 1.0     │ 0.0     │
│ 4   │ 0.0     │ 0.0     │ 0.0     │ 0.0     │ 0.0     │
```

However, there are some scenarios in DataFrames.jl, when you naturally want
a broadcasting-like behavior, but do not allow for the use of `.` operation.
These operations are based on `=>` syntax (recently I described [here][pairsyntax]
where it can be used in general).

Here is a list of relevant cases:

* creation of data frames:
```
julia> df = DataFrame(x=1:3, y=0)
3×2 DataFrame
│ Row │ x     │ y     │
│     │ Int64 │ Int64 │
├─────┼───────┼───────┤
│ 1   │ 1     │ 0     │
│ 2   │ 2     │ 0     │
│ 3   │ 3     │ 0     │
```
* adding columns to data frames:
```
julia> insertcols!(df, :z => "z")
3×3 DataFrame
│ Row │ x     │ y     │ z      │
│     │ Int64 │ Int64 │ String │
├─────┼───────┼───────┼────────┤
│ 1   │ 1     │ 0     │ z      │
│ 2   │ 2     │ 0     │ z      │
│ 3   │ 3     │ 0     │ z      │
```
* transforming data frames using `select`/`select!`/`transform`/`transform!`/`combine`:
```
julia> combine(df, :x, :x => sum)
3×2 DataFrame
│ Row │ x     │ x_sum │
│     │ Int64 │ Int64 │
├─────┼───────┼───────┤
│ 1   │ 1     │ 6     │
│ 2   │ 2     │ 6     │
│ 3   │ 3     │ 6     │
```

In all three examples above we created a column with a scalar;
`0` for `DataFrame` constructor, `"z"` in `insertcols!`,
and `6` (result of aggregation using `sum`) in `combine`;
and this scalar was then broadcasted to all rows in the output data frame.

So a natural question is how does DataFrames.jl decide what and how should be
broadcasted? This behavior is what we call *pseudo broadcasting* and in this
post I explain its rules.

The reason why it is called *pseudo* is because it is not 100% compliant with
broadcasting rules in Julia Base (incidentally DataFrames.jl was there before
broadcasting came to life in Julia), but the rules are the same most of the time
(in particular in a majority of typical scenarios when doing data science work).

# The rules for `DataFrame` and `insertcols!`

I start with the rules for `DataFrame` and `insertcols!` as they are simpler.

*Rule 1.* `AbstractVector`s are accepted as is. This seemingly natural rule
is different than what  broadcasting in Julia Base does, so it is good to be
aware of this difference:
```
julia> df = DataFrame(x=[1,2], y=[1])
ERROR: DimensionMismatch("column :x has length 2 and column :y has length 1")
```
errors, while in Julia Base:
```
julia> tuple.([1, 2], [1])
2-element Array{Tuple{Int64,Int64},1}:
 (1, 1)
 (2, 1)
```
works fine as a dimension of length one gets expanded.

*Rule 2.* Values of type `Ref` or `AbstractArray{<:Any, 0}` (zero dimensional arrays)
are unwrapped and what is stored inside them is recycled. Here is an example
showing that it is useful when you want to protect an `AbstractVector` and treat
it as a cell entry:
```
julia> df = DataFrame(x=[1,2], y=Ref([1,2,3]), z=fill(1:3))
2×3 DataFrame
│ Row │ x     │ y         │ z        │
│     │ Int64 │ Array…    │ UnitRan… │
├─────┼───────┼───────────┼──────────┤
│ 1   │ 1     │ [1, 2, 3] │ 1:3      │
│ 2   │ 2     │ [1, 2, 3] │ 1:3      │
```
This behavior matches what Julia Base Does:
```
julia> tuple.(1:2, Ref([1,2,3]), fill(1:3))
2-element Array{Tuple{Int64,Array{Int64,1},UnitRange{Int64}},1}:
 (1, [1, 2, 3], 1:3)
 (2, [1, 2, 3], 1:3)
```

For curious users most typically you get a zero dimensional array in three
scenarios (all essentially related to cases involving array operations where
all dimensions are dropped):
```
julia> fill(1) # fill with a single argument
0-dimensional Array{Int64,0}:
1

julia> [1 for i in 1] # comprehension iterating over a scalar
0-dimensional Array{Int64,0}:
1

julia> view(1:1, 1) # view dropping all dimensions
0-dimensional view(::UnitRange{Int64}, 1) with eltype Int64:
1
```

*Rule 3.* `AbstractArray`s of dimension higher than one throw an error:
```
julia> DataFrame(x=ones(2,2))
ERROR: ArgumentError: adding AbstractArray other than AbstractVector as a column of a data frame is not allowed
```
This is clearly different than broadcasting in Julia Base, but the rule in DataFrames.jl
is that we do not automatically expand e.g. matrices into several columns except
a constructor that only takes a matrix:
```
julia> DataFrame(ones(2,2))
2×2 DataFrame
│ Row │ x1      │ x2      │
│     │ Float64 │ Float64 │
├─────┼─────────┼─────────┤
│ 1   │ 1.0     │ 1.0     │
│ 2   │ 1.0     │ 1.0     │
```

*Rule 4.* All else is recycled (as in *Rule 2*). This is more flexible than what
Julia Base does, as there you have to opt-in to have a support for broadcasting,
e.g.:
```
julia> DataFrame(x=1:2, y=(a=1, b=2))
2×2 DataFrame
│ Row │ x     │ y              │
│     │ Int64 │ NamedTuple…    │
├─────┼───────┼────────────────┤
│ 1   │ 1     │ (a = 1, b = 2) │
│ 2   │ 2     │ (a = 1, b = 2) │
```
is OK, but in Julia Base:
```
julia> tuple.(1:2, (a=1, b=2))
ERROR: ArgumentError: broadcasting over dictionaries and `NamedTuple`s is reserved
```
errors.

# The rules for aggregation and transformation functions

For `select`/`select!`/`transform`/`transform!`/`combine` the rules above also
apply but with a twist related to the fact that
```
combine(::Union{Function, Pair}, ::Union{DataFrame, ::GroupedDataFrame})
```
(call it *legacy `combine`*)
allows to create multiple columns from one call, and all the other signatures
currently do not allow it, but we plan to support it in the future.

The consequences of this are the following:
* in `ByRow` transformations it is currently disallowed to return
  `NamedTuple` and `DataFrameRow` as a value (in preparation that such values
  might get expanded into multiple columns in the future);
* in *legacy `combine`* it is allowed to return values of types
  `AbstractDataFrame`, `NamedTuple`, `DataFrameRow`, `AbstractMatrix`
  and they get expanded into multiple columns;
* in all other transformation functions it is currently disallowed to return values
  of types `AbstractDataFrame`, `NamedTuple`, `DataFrameRow`, `AbstractMatrix`
  as in the future probably they will get expanded into multiple columns.

The extra rules essentially boil down to the fact that currently *legacy `combine`*
is the only aggregation function that allows for producing multiple columns from
a single transformation function specification (you have to use a separate
transformation specification for every column created in all other
aggregation/transformation functions).

# The future

The decision how to handle returning multiple columns from a single transformation
function specification is one of the last blocking things for a final design
towards DataFrames.jl 1.0 (as likely it will lead to some changes that would be
mildly breaking). Therefore if you would like to voice your opinion how
it should be resolved please comment in [this issue][issue].

[broadcasting]: https://docs.julialang.org/en/v1/manual/mathematical-operations/#man-dot-operators
[pairsyntax]: https://bkamins.github.io/julialang/2020/07/17/pair.html
[issue]: https://github.com/JuliaData/DataFrames.jl/issues/2410
