---
layout: post
title:  "Construction vs conversion in Julia"
date:   2021-03-26 12:17:49 +0200
categories: julialang
---

# Introduction

In the recent [0.22.6 release][release] of the DataFrames.jl package we have
deprecated a bunch of `convert` methods for objects defined in it.

In this post I want to comment on when `convert` is used and when it is
appropriate to define a `convert` method for custom types.

# The history

DataFrames.jl is an old package, with its 0.0.0 release made on Feb 14, 2013.

During this time the Julia language has evolved. Even in Julia 0.6 `convert`
method was a default fallback for construction as you can see [here][j06]:

> However, in some cases you could consider adding methods to `Base.convert`
> instead of defining a constructor, because Julia falls back to calling
> `convert()` if no matching constructor is found. For example, if no constructor
> `T(args...) = ...` exists `Base.convert(::Type{T}, args...) = ...` is called.

This is a past of pre-1.0 Julia design (if you are interested in some more
history you can start with [this issue][issue]). The things have changed now,
but until DataFrames.jl 0.22.6 release we had some convert methods that were
inspired by this rule, and we even had constructors that fell back to `convert`
explicitly.

So what is the current rule for using `convert`?

# The present

Fortunately, as of Julia 1.6 [the manual][conversion] is quite clear that
`convert` should be only defined in cases when it is safe to perform a
conversion even when the user does not ask for it explicitly. Before we move
forward let us see it in action:

```
julia> struct Example
       a::Int
       b::Complex{Int}
       end

julia> Example(1.0, 1.0)
Example(1, 1 + 0im)
```

You can see that `1.0` (of type `Float64`) got implicitly converted to `1` (of
type `Int`). Similarly `1.0` got converted to `1 + 0im` (`Complex{Int}`).
These conversions are performed implicitly to ensure programmer convenience.

However, the following fails:
```
julia> example(a::Int, b::Complex{Int}) = (a, b)
example (generic function with 1 method)

julia> example(1.0, 1.0)
ERROR: MethodError: no method matching example(::Float64, ::Float64)
```

So when does the implicit conversion happen? Again the maunal explains it [here][called]:

> * Assigning to an array converts to the array's element type.
> * Assigning to a field of an object converts to the declared type of the field.
> * Constructing an object with `new` converts to the object's declared field types.
> * Assigning to a variable with a declared type (e.g. `local x::T`) converts to that type.
> * A function with a declared return type converts its return value to that type.
> * Passing a value to `ccall` converts it to the corresponding argument type.

So a short, simplifying, rule is that the conversion happens when you want to make
an assignment to a value that has a pre-specified type.

Let us see it at work:

```
julia> d1 = Set{Int}()
Set{Int64}()

julia> push!(d1, 1.0)
Set{Int64} with 1 element:
  1

julia> d2 = BitSet()
BitSet([])

julia> push!(d2, 1.0)
ERROR: MethodError: no method matching push!(::BitSet, ::Float64)
```

You can push `1.0` to `Set{Int}`, because its `push!` method makes no
restriction on the type of value that is pushed to the `Set{Int}`. Then,
internally, `Set{Int}` converts the passed value to `Int` before storing it. You
can make sure that this happens by writing:

```
julia> push!(d1, "a")
ERROR: MethodError: Cannot `convert` an object of type String to an object of type Int64
Closest candidates are:
  convert(::Type{T}, ::Ptr) where T<:Integer at pointer.jl:23
  convert(::Type{T}, ::T) where T<:Number at number.jl:6
  convert(::Type{T}, ::Number) where T<:Number at number.jl:7
  ...
Stacktrace:
 [1] setindex!(h::Dict{Int64, Nothing}, v0::Nothing, key0::String)
   @ Base ./dict.jl:374
 [2] push!(s::Set{Int64}, x::String)
   @ Base ./set.jl:57
 [3] top-level scope
   @ REPL[32]:1
```

and if you check the `setindex!` implementation you will see the conversion:

```
function setindex!(h::Dict{K,V}, v0, key0) where V where K
    key = convert(K, key0)
    if !isequal(key, key0)
        throw(ArgumentError("$(limitrepr(key0)) is not a valid key for type $K"))
    end
    setindex!(h, v0, key)
end
```

So why you are not allowed to `push!` a float to a `BitSet`? The reason is that
its `push!` method is defined more restrictively like this:

```
@inline push!(s::BitSet, n::Integer) = _setint!(s, _check_bitset_bounds(n), true)
```

and since no conversion happens when passing parameters to functions an error is
thrown.

# Conclusion

To wrap up let us go back to the manual [here][last]:

> since `convert` can be called implicitly, its methods are restricted to cases
> that are considered "safe" or "unsurprising". `convert` will only convert between
> types that represent the same basic kind of thing (e.g. different
> representations of numbers, or different string encodings). It is also usually
> lossless; converting a value to a different type and back again should result in
> the exact same value.

In essence this means that, unless you are developing a low level infrastructure
package (like defining new numeric type), you are not likely to need to define
`convert` methods at all. Just define proper constructors for your types.

In DataFrames.jl we left only three conversions.

The first is from `SubDataFrame` to `DataFrame`. It is clearly a lose-less
conversion that sometimes might be useful (e.g. you have a container of
`DataFrame` objects and want to be able to implicitly add `SubDataFrame` to it).

The second and third conversions are from `DataFrameRow` and `GroupKey` to
`NamedTuple`. The reason behind allowing them is that we try to make both these
types to work like `NamedTuple` (except for the fact that they are views).
Again in this case the conversion is lose-less.

As a final note --- I think Julia 1.6 manual is really a great resource
(half-way of writing this post I considered to stop it as the manual
is so clear about the rules).

[release]: https://github.com/JuliaData/DataFrames.jl/releases/tag/v0.22.6
[j06]: https://docs.julialang.org/en/v0.6/manual/constructors/#constructors-and-conversion-1
[issue]: https://github.com/JuliaLang/julia/pull/23273
[conversion]: https://docs.julialang.org/en/v1/manual/conversion-and-promotion/#Conversion
[called]: https://docs.julialang.org/en/v1/manual/conversion-and-promotion/#When-is-convert-called?
[last]: https://docs.julialang.org/en/v1/manual/conversion-and-promotion/#Conversion-vs.-Construction
