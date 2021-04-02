---
layout: post
title:  "Poor man's guide despecialization"
date:   2021-03-26 12:17:49 +0200
categories: julialang
---

# Introduction

We are just about to release [DataFrames.jl][df] 1.0. It will bring many new
features we are excited about. One of the major additions is support of multiple
threading in many core operations. These changes promise performance
improvements when processing large data frames. The cost is that code complexity
has also grown significantly.

Clearly code-base size is an issue for maintainers of the package, but should
end-users care? The answer is that unfortunately yes, as making code more
complex makes it compile longer.

In this post I summarize the experience we have when trying to reduce compile
time latency.

TLDR: if you have functions that are expensive to compile but are not
performance critical perform [argument standardization][as].

In this post I am using Julia 1.6, MethodAnalysis v0.4.4, SnoopCompile v2.6.0,
and DataFrames.jl `main` branch state at [this commit][commit] (in this case it
is relevant as changes in the code base will likely affect the results).

Before we start our experiments disable precompilation (to have a clean ground
for comparisons and simplify things; the conclusions I present hold also when
proper precompilation statements are added). To do so comment-out lines 152 and
153 in src/DataFrames.jl file:

```
#include("other/precompile.jl")
#precompile()
```

# The context

In DataFrames.jl when one performs transformation of data the following two
areas cause challenges related to run-time compilation:
* users can pass arbitrary functions as transformations;
* the ouptut of these functions can be arbitrary values (and the logic of
  processing them depends on their type).

Let us have a look at a simple example:

```
julia> using DataFrames

julia> gdf = groupby(DataFrame(a=1), :a);

julia> @time combine(gdf, x -> (a=1,));
  2.440215 seconds (8.09 M allocations: 469.718 MiB, 8.16% gc time)

julia> @time combine(gdf, x -> (a=1,));
  0.083986 seconds (550.13 k allocations: 33.481 MiB, 99.73% compilation time)

julia> @time combine(gdf, x -> (b=1,));
  0.364469 seconds (996.17 k allocations: 61.213 MiB, 4.25% gc time, 99.74% compilation time)

julia> @time combine(gdf, x -> (c=1,));
  0.337301 seconds (983.31 k allocations: 60.455 MiB, 1.91% gc time, 99.72% compilation time)
```

We can see that in each call we pass a new anonymous function to `combine`.
Additionally, the return value of this function is a `NamedTuple` that has a
fresh type each time, as the name of the column changes.

As you can see the compilation time in these examples is quite high.

Let us check how to reduce it. We will concentrate on only one method
`_combine_multicol` that is defined in line 7 of
src/groupeddataframe/complextransforms.jl. Its signature is:
```
_combine_multicol(firstres, fun::Base.Callable, gd::GroupedDataFrame,
                  incols::Union{Nothing, AbstractVector, Tuple, NamedTuple})
```

We start witch checking how many method instances were generated for it in our code:
```
julia> using MethodAnalysis

julia> methodinstances(DataFrames._combine_multicol)
9-element Vector{Core.MethodInstance}:
 MethodInstance for _combine_multicol(::NamedTuple{(:a,), Tuple{Int64}}, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:a,), Tuple{Int64}}, ::var"#1#2", ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::Any, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::Any, ::Type, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:a,), Tuple{Int64}}, ::var"#3#4", ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:b,), Tuple{Int64}}, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:b,), Tuple{Int64}}, ::var"#5#6", ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:c,), Tuple{Int64}}, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:c,), Tuple{Int64}}, ::var"#7#8", ::GroupedDataFrame{DataFrame}, ::Nothing)
```
As you can see each call like `combine(gdf, x -> (c=1,))` generates two new method instances.

Before we move forward let us check how this code would be run in 0.22 release
(precompilation is turned on here so we are not comparing apples to apples,
but if we disabled precompilation the conclusion would be similar):
```
julia> using DataFrames

julia> gdf = groupby(DataFrame(a=1), :a);

julia> @time combine(gdf, x -> (a=1,));
  1.412728 seconds (2.69 M allocations: 167.339 MiB, 3.50% gc time, 45.87% compilation time)

julia> @time combine(gdf, x -> (a=1,));
  0.041718 seconds (190.24 k allocations: 11.522 MiB, 20.74% gc time, 99.03% compilation time)

julia> @time combine(gdf, x -> (b=1,));
  0.209238 seconds (481.99 k allocations: 30.126 MiB, 99.58% compilation time)

julia> @time combine(gdf, x -> (c=1,));
  0.194280 seconds (468.45 k allocations: 29.367 MiB, 5.33% gc time, 99.53% compilation time)

julia> using MethodAnalysis

julia> methodinstances(DataFrames._combine_multicol)
13-element Vector{Core.MethodInstance}:
 MethodInstance for _combine_multicol(::DataFrame, ::Function, ::GroupedDataFrame{DataFrame}, ::Tuple{Vector{Bool}})
 MethodInstance for _combine_multicol(::DataFrame, ::Type, ::GroupedDataFrame{DataFrame}, ::Tuple{Vector{Bool}})
 MethodInstance for _combine_multicol(::Any, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::Any, ::Type, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:a,), Tuple{Int64}}, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:b,), Tuple{Int64}}, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:c,), Tuple{Int64}}, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::Any, ::Type, ::GroupedDataFrame{DataFrame}, ::NamedTuple)
 MethodInstance for _combine_multicol(::Any, ::Function, ::GroupedDataFrame{DataFrame}, ::NamedTuple)
 MethodInstance for _combine_multicol(::Any, ::Type, ::GroupedDataFrame{DataFrame}, ::Tuple)
 MethodInstance for _combine_multicol(::Any, ::Function, ::GroupedDataFrame{DataFrame}, ::Tuple)
 MethodInstance for _combine_multicol(::NamedTuple, ::Type, ::GroupedDataFrame{DataFrame}, ::Tuple{Vector{Bool}})
 MethodInstance for _combine_multicol(::NamedTuple, ::Function, ::GroupedDataFrame{DataFrame}, ::Tuple{Vector{Bool}})
```

So we can see that DataFrames.jl 0.22 produced even more instances, but since the
code was simpler the compilation time was lower.

# Despecialization

The first advice one gets in such cases is to use `@nospecialize` on function
arguments to avoid excessive specialization. In our case we see that problematic
are `firstres` and `fun` arguments. Now exit Julia and change the signature of
the method to:

```
_combine_multicol(@nospecialize(firstres), @nospecialize(fun::Base.Callable),
                  gd::GroupedDataFrame,
                  incols::Union{Nothing, AbstractVector, Tuple, NamedTuple})
```

Now start your Julia again and run the code we have checked above:

```
julia> using DataFrames

julia> gdf = groupby(DataFrame(a=1), :a);

julia> @time combine(gdf, x -> (a=1,));
  2.238896 seconds (8.10 M allocations: 470.113 MiB, 5.34% gc time)

julia> @time combine(gdf, x -> (a=1,));
  0.095581 seconds (550.16 k allocations: 33.482 MiB, 9.94% gc time, 99.75% compilation time)

julia> @time combine(gdf, x -> (b=1,));
  0.304849 seconds (982.79 k allocations: 60.359 MiB, 2.95% gc time, 99.71% compilation time)

julia> @time combine(gdf, x -> (c=1,));
  0.297039 seconds (969.93 k allocations: 59.620 MiB, 6.12% gc time, 99.72% compilation time)

julia> using MethodAnalysis

julia> methodinstances(DataFrames._combine_multicol)
7-element Vector{Core.MethodInstance}:
 MethodInstance for _combine_multicol(::NamedTuple{(:a,), Tuple{Int64}}, ::var"#1#2", ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::Any, ::Function, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::Any, ::Type, ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:a,), Tuple{Int64}}, ::var"#3#4", ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:b,), Tuple{Int64}}, ::var"#5#6", ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::NamedTuple{(:c,), Tuple{Int64}}, ::var"#7#8", ::GroupedDataFrame{DataFrame}, ::Nothing)
 MethodInstance for _combine_multicol(::Any, ::Union{Function, Type}, ::GroupedDataFrame{DataFrame}, ::Nothing)
```

As you can see things got slightly better. GC time is distorting the comparison
a bit, but correcting for it we see an improvement. Also we have reduced the
number of method instances from 9 to 7. But wait - we wanted to disable specialization,
and we still get 7 method instances to be compiled. How to reduce it?

# Poor man's despecialization

The approach I found to work reliably is to aggresively perform
[argument standardization][as]. A standard trick to be sure that we break
method specialization for some value is to wrap it in `Ref{Any}`, and then
immediately unwrap.

Let us try it. Now we change our method signature to:
```
_combine_multicol((firstres,)::Ref{Any}, (fun,)::Ref{Any},
                  gd::GroupedDataFrame,
                  incols::Union{Nothing, AbstractVector, Tuple, NamedTuple})
```
we also need to change two lines of code where `_combine_multicol` is called,
namely lines 268:
```
idx, outcols, nms = _combine_multicol(Ref{Any}(firstres), Ref{Any}(cs_i), gd, nothing)
```
and 395:
```
idx, outcols, nms = _combine_multicol(Ref{Any}(firstres), Ref{Any}(fun), gd, incols)
```
in file src/groupeddataframe/splitapplycombine.jl.

Apply the changes and run our code in a fresh Julia session again:
```
julia> using DataFrames

julia> gdf = groupby(DataFrame(a=1), :a);

julia> @time combine(gdf, x -> (a=1,));
  2.262996 seconds (7.70 M allocations: 445.018 MiB, 8.53% gc time)

julia> @time combine(gdf, x -> (a=1,));
  0.047010 seconds (299.33 k allocations: 18.038 MiB, 99.52% compilation time)

julia> @time combine(gdf, x -> (b=1,));
  0.278649 seconds (733.47 k allocations: 45.020 MiB, 2.44% gc time, 99.62% compilation time)

julia> @time combine(gdf, x -> (c=1,));
  0.245550 seconds (720.62 k allocations: 44.284 MiB, 2.73% gc time, 99.61% compilation time)

julia> using MethodAnalysis

julia> methodinstances(DataFrames._combine_multicol)
1-element Vector{Core.MethodInstance}:
 MethodInstance for _combine_multicol(::Base.RefValue{Any}, ::Base.RefValue{Any}, ::GroupedDataFrame{DataFrame}, ::Nothing)
```

Now that is much better. We compiled only one method instance for
`_combine_multicol` and the timings have significantly improved.

# Conclusions

Let us now check what timings the above code has after applying such techniques
to all relevant functions in source code, as proposed in [this PR][pr].
I still keep precompilation disabled so the comparison is again a bit unfair
in reference to 0.22 release:

```
julia> using DataFrames

julia> gdf = groupby(DataFrame(a=1), :a);

julia> @time combine(gdf, x -> (a=1,));
  2.478645 seconds (8.43 M allocations: 489.011 MiB, 9.96% gc time)

julia> @time combine(gdf, x -> (a=1,));
  0.005444 seconds (7.43 k allocations: 486.211 KiB, 96.44% compilation time)

julia> @time combine(gdf, x -> (b=1,));
  0.199044 seconds (371.68 k allocations: 22.773 MiB, 99.39% compilation time)

julia> @time combine(gdf, x -> (c=1,));
  0.173116 seconds (358.82 k allocations: 22.033 MiB, 4.05% gc time, 99.50% compilation time)

julia> using MethodAnalysis

julia> methodinstances(DataFrames._combine_multicol)
1-element Vector{Core.MethodInstance}:
 MethodInstance for _combine_multicol(::Base.RefValue{Any}, ::Base.RefValue{Any}, ::GroupedDataFrame{DataFrame}, ::Base.RefValue{Any})
```

And now we are under 0.22 timings. Also note that the whole compilation cost
is now related to generation of methods related to the new type of the return
value (the `NamedTuple` issue). Let us check:

```
julia> using SnoopCompile

julia> t = @snoopi_deep combine(gdf, x -> (d=1,));

julia> sort(SnoopCompile.flatten(t), by = x -> x.exclusive_time)
271-element Vector{SnoopCompileCore.InferenceTiming}:
 InferenceTiming: 0.000019/0.000019 on InferenceFrameInfo for convert(::Type{Int64}, 1::Int64)
 InferenceTiming: 0.000019/0.000019 on InferenceFrameInfo for convert(::Type{Int64}, 0::Int64)
 InferenceTiming: 0.000020/0.000020 on InferenceFrameInfo for convert(::Type{Int64}, 2::Int64)
 ⋮
 InferenceTiming: 0.007164/0.022407 on InferenceFrameInfo for DataFrames._combine_rows_with_first!(::NamedTuple{(:d,), Tuple{Int64}}, ::Tuple{Vector{Int64}}, ::Function, ::GroupedDataFrame{DataFrame}, nothing::Nothing, ::Tuple{Symbol}, Val{true}()::Val{true})
 InferenceTiming: 0.096098/0.096305 on InferenceFrameInfo for map(::Type{Tuple}, ::Tuple{NamedTuple{(:d,), Tuple{Int64}}})
 InferenceTiming: 0.136345/0.279824 on InferenceFrameInfo for Core.Compiler.Timings.ROOT()
```
and we see that the biggest cost is paid by `map` function applied to `NamedTuple`,
which is a function in Julia Base.

In summary the approaches I discuss here are most useful when you expect to get
values of very heterogeneous types as arguments to your functions. In DataFrames.jl
the two most common cases of these situations are:
* anonymous transformation functions;
* `NamedTuple`s as produced values from transformations.

If you would have any comments on the best strategies to avoid code
specialization please contact me as reducing compilation latency is one
of the priorities of DataFrames.jl 1.0 release.

Before I finish let me add one thing. If you are interested in true performance
guided tips to reduce compilation latency (not just poor man's ones I have given
in this post) I highly recommend you to check out [this][p1], [this][p2], and
[this][p3] post.

[df]: https://github.com/JuliaData/DataFrames.jl
[as]: https://timholy.github.io/SnoopCompile.jl/stable/pgdsgui/#Argument-standardization
[commit]: https://github.com/JuliaData/DataFrames.jl/commit/7300d97053699b573063e6c0980a53042e705f22
[pr]: https://github.com/JuliaData/DataFrames.jl/pull/2691
[p1]: https://julialang.org/blog/2020/08/invalidations/
[p2]: https://julialang.org/blog/2021/01/precompile_tutorial/
[p3]: https://julialang.org/blog/2021/01/snoopi_deep/
