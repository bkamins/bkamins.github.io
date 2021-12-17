---
layout: post
title:  "News features in DataFrames.jl 1.3: part 2"
date:   2021-12-17 08:41:53 +0200
categories: julialang
---

# Introduction

This post continues the presentation of new features added in [DataFrames.jl]
[df] 1.3.0. Last week in [this post][p1] I have discussed the changes that
improve performance of reduction operations that take wide data (e.g. taking an
average of 10,000 columns). This week I will focus on improvements of convenience
of use of the data transformation mini-language.

The post was written under Julia 1.7.0 and DataFrames.jl 1.3.0.

# The data transformation mini-language

The `select[!]`, `transform[!]`, `combine`, and `subset[!]` functions in
DataFrames.jl accept specification of column transformation's using a so called
data transformation mini-language. It has a general form:

```
[input column names] => [transformation function] => [output columns]
```

A full specification of allowed forms can be found [here][spec]. However, you
might find it a bit technical. This is unfortunately unavoidable, as the
mini-language was designed to allow maximum flexibility, so that packages like
[DataFramesMeta.jl][dfm1] or [DataFrameMacros.jl][dfm2] can rely on it and
provide a nice user-facing syntax. Therefore in [this post][b1] I have
presented several introductory examples of its usage.

# New features

One of the common advanced use-cases of the mini-language is performing the same
transformation on multiple columns of a data frame. Imagine that you have
the following data frame:

```
julia> using DataFrames

julia> df = DataFrame(name='A':'E', year2019=1:5, year2020=2:6, year2021=3:7)
5×4 DataFrame
 Row │ name  year2019  year2020  year2021
     │ Char  Int64     Int64     Int64
─────┼────────────────────────────────────
   1 │ A            1         2         3
   2 │ B            2         3         4
   3 │ C            3         4         5
   4 │ D            4         5         6
   5 │ E            5         6         7
```

Now assume that we wanted to calculate sum of each of the columns `:year2019`,
`:year2020`, and `:year2021`. The simplest way to achieve this is the
following:

```
julia> combine(df, :year2019 => sum, :year2020 => sum, :year2021 => sum)
1×3 DataFrame
 Row │ year2019_sum  year2020_sum  year2021_sum
     │ Int64         Int64         Int64
─────┼──────────────────────────────────────────
   1 │           15            20            25
```

(Note that in the call I have omitted output column name part so DataFrames.jl
automatically generated the column names consisting of the source column name
and the transformation function name that was applied to it.)

However, you might consider the above call to the `combine` function a bit
redundant. You can write the same using broadcasting like this:

```
julia> combine(df, [:year2019, :year2020, :year2021] .=> sum)
1×3 DataFrame
 Row │ year2019_sum  year2020_sum  year2021_sum
     │ Int64         Int64         Int64
─────┼──────────────────────────────────────────
   1 │           15            20            25
```

Note how the `[:year2019, :year2020, :year2021] .=> sum` is being handled by
Julia *before* it is passed to the `combine` function:

```
julia> [:year2019, :year2020, :year2021] .=> sum
3-element Vector{Pair{Symbol, typeof(sum)}}:
 :year2019 => sum
 :year2020 => sum
 :year2021 => sum
```

Now you might ask, what if I did not have three columns to process but 100 of
them? It is easy to select their names using the `names` function. Here I show
you how to select all columns in the data frame except the `:name` column:

```
julia> names(df, Not(:name))
3-element Vector{String}:
 "year2019"
 "year2020"
 "year2021"
```

Therefore the call to `combine` above can be rewritten as:

```
julia> combine(df, names(df, Not(:name)) .=> sum)
1×3 DataFrame
 Row │ year2019_sum  year2020_sum  year2021_sum
     │ Int64         Int64         Int64
─────┼──────────────────────────────────────────
   1 │           15            20            25
```

This already looks quite powerful, but there is one annoying thing. Why do we
need to call the `names` function? It should be obvious that `Not(:name)`
applies to the `df` data frame. Let us check if this would work:

```
julia> combine(df, Not(:name) .=> sum)
1×3 DataFrame
 Row │ year2019_sum  year2020_sum  year2021_sum
     │ Int64         Int64         Int64
─────┼──────────────────────────────────────────
   1 │           15            20            25
```

Yes it does! And this is the new feature in DataFrames.jl 1.3 I wanted to talk
about today.

The `select[!]`, `transform[!]`, `combine`, and `subset[!]` functions when they
get any of the selectors `Not`, `Between`, `Cols`, `All` in a broadcasting
expression are now able to resolve them with respect to the context of the data
frame that is being processed by them.

Let me give two more examples of this feature to show you how it works:

```
julia> combine(df, Not(:name) .=> [minimum maximum])
1×6 DataFrame
 Row │ year2019_minimum  year2020_minimum  year2021_minimum  year2019_maximum  year2020_maximum  year2021_maximum
     │ Int64             Int64             Int64             Int64             Int64             Int64
─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1 │                1                 2                 3                 5                 6                 7

julia> combine(df, Not(:name) .=> sum .=> Not(:name))
1×3 DataFrame
 Row │ year2019  year2020  year2021
     │ Int64     Int64     Int64
─────┼──────────────────────────────
   1 │       15        20        25
```

In the first one you can see that broadcasting is properly applied even in two
dimensional case (note that `[minimum maximum]` is a `Matrix`).

In the second example you see that broadcasting is properly handled both in
specification of source as well as for target column names.

# Behind the scenes

The way things work are in my opinion intuitive and expected. However, let me
show you that they are not as easy as you might think. The reason is that
broadcasting is resolved *before* the data transformation mini-language
expression is passed to `combine` (or other transformation functions I have
listed). Let us check how the expressions I have used above get resolved
*before* they got passed to `combine`:

```
julia> Not(:name) .=> sum
InvertedIndices.BroadcastedInvertedIndex(InvertedIndex{Symbol}(:name)) => sum

julia> Not(:name) .=> [minimum maximum]
1×2 Matrix{Pair{InvertedIndices.BroadcastedInvertedIndex}}:
 BroadcastedInvertedIndex(InvertedIndex{Symbol}(:name))=>minimum  BroadcastedInvertedIndex(InvertedIndex{Symbol}(:name))=>maximum

julia> Not(:name) .=> sum .=> Not(:name)
InvertedIndices.BroadcastedInvertedIndex(InvertedIndex{Symbol}(:name)) => (sum => InvertedIndices.BroadcastedInvertedIndex(InvertedIndex{Symbol}(:name)))
```

They look quite messy. What is the problem? All these three data transformation
mini-language expressions do not include `df` in them. Therefore when Julia
executes the broadcasting operation *it is unaware* of the `df` context. The
workaround is to create a special `BroadcastedInvertedIndex` object (in the
case of `Not` operation; for `Cols`, `Between`, and `All` also a special
wrapper object is created) that signals `combine` that broadcasting was used on
`Not(:name)` selector. Then `combine` internally has implemented its own
broadcasting machinery that matches the Julia Base broadcasting rules and
resolves the expression within the `df` context as required.

As you can see things that seem simple end up quite complex. In particular
this means that DataFrames.jl must closely monitor changes in Julia Base
broadcasting implementation to make sure it matches its rules.

# Conclusions

I have two conclusions for today.

The first one is user facing. In DataFrames.jl 1.3 we have added a long
requested convenience functionality of broadcasting `Not`, `Cols`, `Between`,
and `All` calls in data transformation mini-language within the context of a
data frame that they apply to. Therefore, hopefully, our users will be more
happy now.

The second is for DataFrames.jl maintenance. Some of the users might have noted
that JuliaData members always ask for a strong justification before new
features are added. The reason is twofold. Firstly, having increasingly more
features makes learning of DataFrames.jl harder. Secondly, as you can see in
the example given today, adding new features makes the code base of
DataFrames.jl quite complex and implicitly strongly linked to Julia Base
design. This means that it becomes increasingly harder for new contributors to
get involved in the package development (and we would love to see more of them
so we prefer to keep things simple if possible).

Finally, in the coming weeks I will continue the discussion of the new features
in DataFrames.jl 1.3, so stay tuned.

[df]: https://github.com/JuliaData/DataFrames.jl
[p1]: https://bkamins.github.io/julialang/2021/12/10/rowreduce.html
[spec]: https://dataframes.juliadata.org/stable/man/split_apply_combine/
[b1]: https://bkamins.github.io/julialang/2020/12/24/minilanguage.html
[dfm1]: https://github.com/JuliaData/DataFramesMeta.jl
[dfm2]: https://github.com/jkrumbiegel/DataFrameMacros.jl
