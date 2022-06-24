---
layout: post
title:  "Broadcasting in Julia: the Good, the Bad, and the Ugly"
date:   2022-06-24 07:43:32 +0200
categories: julialang
---

# Introduction

Recently I have written a [post about `@view` and `@views` macros][m1].
In what followed I got a feedback, that the topic I touched there was
indeed useful. Therefore for today I decided to write about another macro.
This time I will share my thoughts on the `@.` macro.

This post was written under Julia 1.7.2.

# Understanding what `@.` macro does

The `@.` macro is used to inject broadcasting into every function call
in an expression passed to it.

Let us have a look at its docstring:

>  Convert every function call or operator in expr into a "dot call"
> (e.g. convert f(x) to f.(x)), and convert every
> assignment in expr to a "dot assignment" (e.g. convert += to .+=).

> If you want to avoid adding dots for selected function calls in expr, splice
> those function calls in with $. For example,
> @. sqrt(abs($sort(x))) is equivalent to sqrt.(abs.(sort(x))) (no dot for sort).

The `@.` marco is quite useful when we work with long expressions that
involve several operations that we want to be broadcasted together. Here
is a minimal example:

```
julia> using Statistics

julia> x = 1:10
1:10

julia> @. sin(x)^2 + cos(x)^2
10-element Vector{Float64}:
 1.0
 1.0
 0.9999999999999999
 1.0
 0.9999999999999999
 0.9999999999999999
 0.9999999999999999
 1.0
 0.9999999999999999
 1.0
```

Writing `@. sin(x)^2 + cos(x)^2` is much more convenient than writing
`sin.(x).^2 .+ cos.(x).^2`.

However, what if we wanted to compute variance of `x`? A direct approach
would be to write this operation as:

```
julia> sum((x .- mean(x)) .^ 2) / (length(x) - 1)
9.166666666666666
```

As you can see I use broadcasting in only two places, while most of the
operations are not broadcasted. If we wanted to use `@.` macro we would
need to use `$` escaping and write something like:

```
julia> @. $/($sum((x - $mean(x)) ^ 2), $-($length(x), 1))
9.166666666666666
```

which is equivalent and ugly. You can check it by using `@macroexpand`:

```
julia> @macroexpand @. $/($sum((x - $mean(x))^2), $-($length(x), 1))
:(sum((^).((-).(x, mean(x)), 2)) / (length(x) - 1))
```

In this case also correct (but not equivalent) and simpler way would be:

```
julia> @. $sum((x - $mean(x))^2) / ($length(x) - 1)
9.166666666666666
```

However, in this case you need to know and be sure that by not escaping-out the
`/` and `-` function calls in the second part of the expression you will not
affect the correctness of your calculation.

In summary, I do not use `@.` in complex expressions as it is usually hard to
reason about it.

Let us now switch to some special cases of using `@.`.

# Be careful with broadcasted assignment

Other common source of bugs when using `@.` is broadcasted assignment.

Let us analyze the following code:

```
julia> x = ["a", "b"]
2-element Vector{String}:
 "a"
 "b"

julia> x = @. length(x)
2-element Vector{Int64}:
 1
 1

julia> x
2-element Vector{Int64}:
 1
 1

julia> y = ["a", "b"]
2-element Vector{String}:
 "a"
 "b"

julia> @. y = length(y)
ERROR: MethodError: Cannot `convert` an object of type Int64 to an object of type String
```

In the case of `x` variable the `@.` macro is on the right hand side of the
assignment. In this case we get a fresh binding of value to variable `x`.

In the case of `y` the `@.` encompasses the left hand side of the assignment.
In this case the operation is in-place. Therefore in this case we get an error,
because you cannot store integers in a vector of strings.

Things, of course, can be silently wrong, as in the following example of
`Vector{Char}`, as `Char` supports conversion from integer:

```
julia> z = ['a', 'b']
2-element Vector{Char}:
 'a': ASCII/Unicode U+0061 (category Ll: Letter, lowercase)
 'b': ASCII/Unicode U+0062 (category Ll: Letter, lowercase)

julia> @. z = length(z)
2-element Vector{Char}:
 '\x01': ASCII/Unicode U+0001 (category Cc: Other, control)
 '\x01': ASCII/Unicode U+0001 (category Cc: Other, control)

julia> z
2-element Vector{Char}:
 '\x01': ASCII/Unicode U+0001 (category Cc: Other, control)
 '\x01': ASCII/Unicode U+0001 (category Cc: Other, control)
```

# Incorrect handling of named tuples

Consider the following code:
```
julia> v = 1:3
1:3

julia> [x in (a=1, c=3) for x in v]
3-element Vector{Bool}:
 1
 0
 1
```
We can rewrite it using broadcasting as:
```
julia> in.(v, Ref((a=1, c=3)))
3-element BitVector:
 1
 0
 1
```
Now we think we could use the `@.` here as follows:
```
julia> @. in(v, $Ref((a=1, c=3)))
ERROR: UndefVarError: c not defined
```
However, this fails as `@.` macro incorrectly handles `=` inside `NamedTuple`
definition. We have to write:
```
julia> @. in(v, $Ref((; a=1, c=3)))
3-element BitVector:
 1
 0
 1
```
The `;` at the beginning of `NamedTuple` definition gives an equivalent object
but changes how the code expression is transformed by the Julia compiler
and it works. Here is how we can check the difference in the representation
of `(a=1, c=3)` and `(; a=1, c=3)`:
```
julia> dump(:(a=1, c=3))
Expr
  head: Symbol tuple
  args: Array{Any}((2,))
    1: Expr
      head: Symbol =
      args: Array{Any}((2,))
        1: Symbol a
        2: Int64 1
    2: Expr
      head: Symbol =
      args: Array{Any}((2,))
        1: Symbol c
        2: Int64 3

julia> dump(:(; a=1, c=3))
Expr
  head: Symbol tuple
  args: Array{Any}((1,))
    1: Expr
      head: Symbol parameters
      args: Array{Any}((2,))
        1: Expr
          head: Symbol kw
          args: Array{Any}((2,))
            1: Symbol a
            2: Int64 1
        2: Expr
          head: Symbol kw
          args: Array{Any}((2,))
            1: Symbol c
            2: Int64 3
```

Fortunately the case of `NamedTuple` is not likely to be problematic in practice
as it is extremely rare.

# Conclusions

The `@.` macro can be very convenient. However, in my experience, you
need to be careful when you use it as it is easy to get surprising results
if you work with complex expressions. Out of the possible problematic situations
I have covered in my post a most common one is forgetting to add `$` to avoid
broadcasting of some function calls in a complex expression.

I hope that you will find this post useful and it will help you to avoid
bugs in your Julia code using broadcasting!

[m1]: https://bkamins.github.io/julialang/2022/06/10/view.html
