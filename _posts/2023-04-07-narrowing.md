---
layout: post
title:  "Element type surprises when processing collections in Julia"
date:   2023-04-07 14:01:54 +0200
categories: julialang
---

# Introduction

Today I want to write about a topic that is a quite tricky
element of design of Julia. The issue is that it is sometimes hard
to predict the element type of the output collection produced by
an operation that transforms an input collection.

The description above looks complicated, but the problem is
encountered in practice, so let me explain it by example.

The post was written under Julia 1.9.0-rc1.

# A basic example

Assume you have some input collection `[1, 2, 3]` and you
want to compute square root of all its elements.

Let us consider three standard ways how you can do it:

```
julia> x = [1, 2, 3]
3-element Vector{Int64}:
 1
 2
 3

julia> sqrt.(x)
3-element Vector{Float64}:
 1.0
 1.4142135623730951
 1.7320508075688772

julia> map(sqrt, x)
3-element Vector{Float64}:
 1.0
 1.4142135623730951
 1.7320508075688772

julia> [sqrt(v) for v in x]
3-element Vector{Float64}:
 1.0
 1.4142135623730951
 1.7320508075688772
```

As you can see in each case a proper element type, that is
`Float64`, was determined for the returned collection.

This behavior is useful, as the user does not have to think
about specifying the output element type. In fact,
in combination with the transformation using the `identity` function,
this behavior can be used to conveniently narrow
down element type of some collection:

```
julia> y = Any[1, 2, 3]
3-element Vector{Any}:
 1
 2
 3

julia> identity.(y)
3-element Vector{Int64}:
 1
 2
 3

julia> map(identity, y)
3-element Vector{Int64}:
 1
 2
 3

julia> [v for v in y]
3-element Vector{Int64}:
 1
 2
 3
```

This pattern comes handy if we have input data that does not have a known
element type, but later we want to perform element type narrowing
when processing it (one of the major benefits of such narrowing
is that processing vectors of `Any` values is slow so we typically want to avoid it).

# The hard case

Automatic output element type detection works nice most of the time.
Unfortunately, when we work with empty collections, it becomes hard to predict.
Here is a simple example:

```
julia> String.([])
Any[]

julia> map(String, [])
String[]

julia> [String(v) for v in []]
Any[]

julia> string.([])
AbstractString[]

julia> map(string, [])
Any[]

julia> [string(v) for v in []]
Any[]
```

As you can see from it broadcating, `map`, and comprehension use a different set of
rules to automatically determine the produced element type. These rules of course
exist and could be learned, but the point is that the issue is non-trivial.

The problem is that when you are writing production code
(e.g. you are developing a package) you want to be sure
what the element type of the collection you produce will be, as often
you cannot know upfront if the input collection the user is going to provide
will be empty or not.

# The solution I use

In situations when it matters what the element type of the collection
produced by some transformation is going to be I use comprehensions
with output element type annotation:

```
julia> [string(v) for v in []]
Any[]

julia> String[string(v) for v in []]
String[]
```

Such annotation has an additional consequence that it is going to perform
conversion of the produced elements to the target type if needed:

```
julia> using Test

julia> s = ["a", GenericString("a")]
2-element Vector{AbstractString}:
 "a"
 "a"

julia> [string(v) for v in s]
2-element Vector{AbstractString}:
 "a"
 "a"

julia> typeof.([string(v) for v in s])
2-element Vector{DataType}:
 String
 GenericString

julia> String[string(v) for v in s]
2-element Vector{String}:
 "a"
 "a"

julia> typeof.(String[string(v) for v in s])
2-element Vector{DataType}:
 String
 String
```

Note that in the example prefixing the comprehension with
`String` made sure that the result of the operation has
`String` element type and all produced values have this type.

# Element type widening

Let me comment on one common related operation. What if we
want to initialize some container with a given value but
we want its element type to be wider? This is not an artificial
case - it often happens with `missing` (where we initialize
some container with this value only to later replace `missing` with proper
values).

Using the `fill` function is a first thing we might try:

```
julia> fill(missing, 3)
3-element Vector{Missing}:
 missing
 missing
 missing
```

However, the produced container has `Missing` element type which
is not useful if we e.g. wanted to later also store integers in it.

One can use a comprehension annotated with a proper output element type
instead:

```
julia> Union{Int, Missing}[missing for _ in 1:3]
3-element Vector{Union{Missing, Int64}}:
 missing
 missing
 missing
```

The pattern with `missing` is needed often enough that we have
a custom function in the Missings.jl package that can be used
to get the desired result more conveniently:

```
julia> using Missings

julia> missings(Int, 3)
3-element Vector{Union{Missing, Int64}}:
 missing
 missing
 missing
```

# Conclusions

Fortunately, in interactive use the problem with setting
of the proper element type for some collection does not
occur often. However, when I write production programs I make
sure to always think if I need to use comprehension with
element type specification to ensure type stability of my code.
