---
layout: post
title:  "My approach to named tuples in Julia"
date:   2023-06-30 06:21:21 +0200
categories: julialang
---

# Introduction

Named tuples are one of the most commonly used data types in Julia.
In this post I want to discuss how I think of this type and use it in practice.

This post was written under Julia 1.9.1.

# The basics

The Julia Manual in the section [Named Tuples][nt] specifies:

> The components of tuples can optionally be named, in which case a *named tuple* is constructed (...)
> Named tuples are very similar to tuples, except that fields can additionally
> be accessed by name using dot syntax (`x.a`) in addition to the regular indexing syntax (`x[1]`).

And gives the following example:

```
julia> x = (a=2, b=1+2)
(a = 2, b = 3)

julia> x[1]
2

julia> x.a
2
```

This definition highlights that `NamedTuple` is similar to a `Tuple` except that its
components can have names.

However, I find it more natural to think of a `NamedTuple` as an anonymous struct,
that optionally can be integer-indexed to get access to its components.

Let me give a few reasons why I think it is a better mental model.

# Subtyping behavior

Let us have a look at the `Tuple` type first. Construct a basic tuple:

```
julia> t = (1, "a")
(1, "a")

julia> typeof(t)
Tuple{Int64, String}
```

Note that type of `t` is a subtype of e.g. `Tuple{Integer, AbstractString}`:

```
julia> t isa Tuple{Integer, AbstractString}
true
```

This subtyping behavior is called *covariant*.

Moreover `Tuple{Integer, AbstractString}` is not concrete:

```
julia> isconcretetype(Tuple{Integer, AbstractString})
false
```

This means that you cannot create a tuple that has this type.
If you try the following call:

```
julia> t2 = Tuple{Integer, AbstractString}(t)
(1, "a")

julia> typeof(t2)
Tuple{Int64, String}
```

Nothing changes. The resulting `t2` tuple has concrete parameters `Int64` and `String`.

You might ask how much of these features a `NamedTuple` shares? The answer is none.
Let us check:

```
julia> nt = (a=1, b="a")
(a = 1, b = "a")

julia> typeof(nt)
NamedTuple{(:a, :b), Tuple{Int64, String}}
```

Type of `nt` is not a subtype of e.g. `NamedTuple{(:a, :b), Tuple{Integer, AbstractString}}`:

```
julia> nt isa NamedTuple{(:a, :b), Tuple{Integer, AbstractString}}
false
```

However, it is a subtype of `UnionAll` type `NamedTuple{(:a, :b), <:Tuple{Integer, AbstractString}}`.

```
julia> nt isa NamedTuple{(:a, :b), <:Tuple{Integer, AbstractString}}
true
```

This subtyping behavior is called *invariant* and is shared with `struct` types.

Moreover, you can construct a value that has `NamedTuple{(:a, :b), Tuple{Integer, AbstractString}}` type (so it is concrete):

```
julia> isconcretetype(NamedTuple{(:a, :b), Tuple{Integer, AbstractString}})
true

julia> nt2 = NamedTuple{(:a, :b), Tuple{Integer, AbstractString}}((1, "a"))
NamedTuple{(:a, :b), Tuple{Integer, AbstractString}}((1, "a"))

julia> typeof(nt2)
NamedTuple{(:a, :b), Tuple{Integer, AbstractString}}
```

Again, this behavior is the same as for `struct` types.

The fact that you can flexibly set parameters of a named tuple is mostly useful
if you have heterogeneous data. Here is an example.

Permissive code:

```
julia> [(a=1, b=missing), (a=missing, b="x")]
2-element Vector{NamedTuple{(:a, :b)}}:
 (a = 1, b = missing)
 (a = missing, b = "x")

julia> typeof.([(a=1, b=missing), (a=missing, b="x")])
2-element Vector{DataType}:
 NamedTuple{(:a, :b), Tuple{Int64, Missing}}
 NamedTuple{(:a, :b), Tuple{Missing, String}}
```

Here we created a vector with elements having different types.
The code using it would not be type stable. However, we could make the types the same
(which would be more compiler friendly; this would be relevant if you processed really large data):

```
julia> @NamedTuple{a::Union{Int,Missing}, b::Union{String,Missing}}.([(a=1, b=missing), (a=missing, b="x")])
2-element Vector{NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}}:
 NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}((1, missing))
 NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}((missing, "x"))
```

The difference is that now the compiler will know the type of every field of our named tuple
(and Julia gracefully handles small `Union`s).

Note that I used the `@NamedTuple{a::Union{Int,Missing}, b::Union{String,Missing}}` macro call
for a convenient way to specify `NamedTuple` type. Let us check it (with one more way to write it):

```
julia> @NamedTuple{a::Union{Int,Missing}, b::Union{String,Missing}}
NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}

julia> @NamedTuple begin
           a::Union{Int,Missing}
           b::Union{String,Missing}
       end
NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}
```

Now, even on the syntax level, the similarity to `struct` definition is apparent.

Let us define two types:

```
julia> const S1 = @NamedTuple begin
           a::Union{Int,Missing}
           b::Union{String,Missing}
       end
NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}

julia> struct S2
           a::Union{Int,Missing}
           b::Union{String,Missing}
       end
```

The `S1` and `S2` types are almost undistinguishable. The only differences are:
* `S1` is a subtype of `NamedTuple` `UnionAll`, while `S2` is just a subtype of `Any`;
* `S1` accepts a tuple as a single argument, while `S2` accepts two arguments;
* `S1` is indexable, while `S2` is not;
* adding methods to foreign functions for `S1` is type piracy.

# How named tuples are different from structs

Let us create instances of `S1` and `S2` and discuss the differences mentioned above.

First the difference in the constructor:

```
julia> x1 = S1((1, "a"))
NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}((1, "a"))

julia> x2 = S2(1, "a")
S2(1, "a")
```

We can see the difference in the constructor. An extra pair of parentheses is needed for `S1`.
It would be tempting to define:

```
julia> S1(x, y) = S1((x, y))
NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}

julia> S1(1, "a")
NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}((1, "a"))
```

This would work. But if you are writing library code you should never do this.
The reason is that you have just added a method for
`NamedTuple{(:a, :b), Tuple{Union{Missing, Int64}, Union{Missing, String}}}` type
and this type is part of Base Julia. Such behavior is called type piracy
and potentially could break some third-party code (in this case the risk is minimal,
but it is worth haveing this habit).

The difference is that with your custom type `S2` you are free to add methods to
any functions you like (even third party), since you own `S2`. Therefore
(provided you make a correct method definition) you will not break any code by doing so.

Let us give an example. We have discussed the benefit of `NamedTuple` over
`struct` is that it is indexable:

```
julia> x1[1]
1

julia> x2[1]
ERROR: MethodError: no method matching getindex(::S2, ::Int64)
```

However, you can easily fix this:

```
julia> Base.getindex(x::S2, i::Int) = getfield(x, i)

julia> x2[1]
1
```

What we did by adding a method to `Base.getindex` is not type piracy since
`S2` is a type that we own.

The last difference between `S1` is `S2` is that, as we mentioned,
`S1` is subtype of `NamedTuple`, so it will get accepted by functions
that work with named tuples, as opposed to `S2`. Here is a small example:

```
julia> empty(x1)
NamedTuple()

julia> empty(x2)
ERROR: MethodError: no method matching empty(::S2)
```

Again - we could easily add a method for `Base.empty` for `S2` if we wanted, just as we did
with `Base.getindex` above.

# Conclusions

Given these examples named tuple shares some features with tuples, and some with structs.
My experience is that it is more similar to a struct in practice. There are two reasons:

1. I rarely index or iterate named tuples, most often I access their fields by name.
2. In dispatch named tuple does behave like a struct (this is quite relevant when writing your code).

Still I like to think of named tuple as an *anonymous struct*. Although we could give it an alias
like we did with `S1` definition, this is rarely done. Typically the concrete type of a named tuple
is left dynamic, and I just define my functions for `NamedTuple` `UnionAll`.

Happy hacking!

[nt]: https://docs.julialang.org/en/v1/manual/functions/#Named-Tuples
