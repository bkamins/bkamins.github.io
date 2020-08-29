---
layout: post
title:  "Subsetting strings in Julia using character indexing"
date:   2020-08-29 10:15:21 +0200
categories: julialang
---

# Introduction

Two weeks ago I have written [a blog post][post] about comparison of byte and
character indexing of strings in Julia Base.

In the mean time I have answered several questions when users had to subset
a `String` in Julia using character indices. In this post I show a macro that
allows to do this.

This code is tested to work under Julia 1.5.

# The implementation of `@char` macro

The `@char` macro is shown below (I hope I got all hygene right --- if not
please let me know)

{% highlight julia %}
macro char(ex)
    if Meta.isexpr(ex, :ref) &&
        isa(ex.args[1], Union{String, Symbol}) &&
        length(ex.args) == 2

        S, i = ex.args
        i, _ = Base.replace_ref_begin_end_!(i,
            (:($Base.firstindex($S)),:($Base.length($S))))
        ex.args[2] = Expr(:(.), Base.nextind, Expr(:tuple, S, 0, i))
        return esc(ex)
    else
        throw(ArgumentError("@char macro argument must be an expression S[i]."))
    end
end
{% endhighlight %}

What this macro does is turning `str[idx]` expression from using byte indexing
to use character indexing by writing `@char str[idx]`.
I think it is simplest to explain it using an example:

```
julia> "∀∃12😄🍕"[2:5]
ERROR: StringIndexError("∀∃12😄🍕", 2)
Stacktrace:
 [1] string_index_err(::String, ::Int64) at ./strings/string.jl:12
 [2] getindex(::String, ::UnitRange{Int64}) at ./strings/string.jl:249
 [3] top-level scope at REPL[5]:1

julia> @char "∀∃12😄🍕"[2:5]
"∃12😄"

julia> str = "∀∃12😄🍕"
"∀∃12😄🍕"

julia> str[2:5]
ERROR: StringIndexError("∀∃12😄🍕", 2)
Stacktrace:
 [1] string_index_err(::String, ::Int64) at ./strings/string.jl:12
 [2] getindex(::String, ::UnitRange{Int64}) at ./strings/string.jl:249
 [3] top-level scope at REPL[17]:1

julia> @char str[2:5]
"∃12😄"
```

Let us also check that we correctly handle `begin` and `end` when indexing:
```
julia> idx = [2, 4]
2-element Array{Int64,1}:
 2
 4

julia> @char str[[begin, idx[begin], idx[end], end]]
"∀∃2🍕"

julia> @macroexpand @char str[[begin, idx[begin], idx[end], end]]
:(str[(nextind).(str, 0, [(Base).firstindex(str), idx[(firstindex)(idx)],
  idx[(lastindex)(idx)], (Base).length(str)])])
```
and all seems to work correctly.

# Concluding comments

I have limited this macro to expect that in `str[idx]` expression `str` is a
variable name or a string literal to simplify the logic of the code (allowing
`str` to be a general expression would lead to a much more complex code).
I assume that in practice this should not be a severe limitation.

In terms of performance this macro does not do any optimizations of the lookup
of character index as `nextind` is called for each byte index separately,
so in some special cases this could be optimized.

Finally it is worth to remember that for most common cases of string subsetting
the `first`, `last` and `chop` functions defined in Julia Base are available
and they use character indexing.

[post]: https://bkamins.github.io/julialang/2020/08/13/strings.html
