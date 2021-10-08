---
layout: post
title:  "Julia beginner's corner: mastering comparison operators"
date:   2021-10-08 07:01:41 +0200
categories: julialang
---

# Introduction

In every programming language performing comparisons is one of the most
fundamental operations. Many people starting to learn Julia find it surprising
that it provides two sets of comparison operators. Today I want to summarize
how each of them works and discuss the practical consequences.

In this the post I use Julia 1.6.3.

# The standard comparison operators

Normally one uses `==` and `!=` to test for equality, and `<`, `>`, `<=`, and
`>=` to test for ordering of values.

Here we can see how it works:

```
julia> 1 == 2
false

julia> "ab" < "cd"
true

julia> (1, "ab") > (1, "cd")
false

julia> (1 => 2) < (3 => 4)
true
```

An important distinction is that `==` and `!=` are always defined for values of
any type, while the ordering comparisons are defined only when the type designer
decided that such comparisons make sense.

Another general rule that is worth remembering, and it was shown in the examples
above, is that comparisons can be applied to collections (like e.g. arrays or
tuples) and they normally are implemented by recursively comparing elements
contained in the collection using the lexicographic ordering.

The standard operators are typically used in practice and are, in particular,
easy to type and read. However, they exhibit behavior that might not be
desirable in certain cases. The three most common situations are as follows.

*Case 1*: numeric `-0.0` and `0.0` are considered equal:

```
julia> -0.0 == 0.0
true

julia> -0.0 < 0.0
false
```

*Case 2*: comparisons with `NaN` always produce `false`:

```
julia> NaN == NaN
false

julia> NaN < NaN
false

julia> NaN > NaN
false

julia> NaN == 1.0
false

julia> NaN < 1.0
false

julia> NaN > 1.0
false
```

*Case 3*: comparisons with `missing` always produce `missing`:

```
julia> missing == missing
missing

julia> missing < missing
missing

julia> missing > missing
missing

julia> missing == 1
missing

julia> missing < 1
missing

julia> missing > 1
missing
```

These properties make sense in certain situations, but when e.g. we want
to sort values or store them in a dictionary or set they are not desirable.
Therefore Julia introduces another set of comparison operators.

# The special comparison operators

There are two special comparison functions: `isequal` and `isless`. The major
difference here is that the user can expect that these comparisons always return
a `Bool` value. Additionally `isequal` distinguishes `0.0` and `-0.0` and
considers all `NaN` values as equal. Therefore:

```
julia> isequal(NaN, NaN)
true

julia> isless(NaN, NaN)
false

julia> isequal(-0.0, 0.0)
false

julia> isless(-0.0, 0.0)
true

julia> isequal(missing, missing)
true

julia> isless(missing, missing)
false
```

Here is an example of `isequal` at work:

```
julia> unique([-1.0, -0.0, 0.0, missing, NaN, NaN])
5-element Vector{Union{Missing, Float64}}:
  -1.0
  -0.0
   0.0
    missing
 NaN
```

The `unique` function uses `isequal` to test for equality thus it de-duplicated
`NaN`s, but retained both `-0.0` and `0.0`. Also, even though we had `missing`
in the vector it was not a problem and the function worked without an error.

Similarly, `sort` uses `isless` by default so the following works:

```
julia> sort([-1.0, -0.0, 0.0, missing, NaN, NaN])
6-element Vector{Union{Missing, Float64}}:
  -1.0
  -0.0
   0.0
 NaN
 NaN
    missing
```

If we switched the comparison operator to `<` we would get an error:

```
julia> sort([-1.0, -0.0, 0.0, missing, NaN, NaN], lt=<)
ERROR: TypeError: non-boolean (Missing) used in boolean context
```

or have an undefined result in corner cases:

```
julia> sort([-0.0, 0.0], lt=<)
2-element Vector{Float64}:
 -0.0
  0.0

julia> sort([0.0, -0.0], lt=<)
2-element Vector{Float64}:
  0.0
 -0.0
```

In practice the most common use of `isequal` and `isless` is in cases when
we want to avoid `missing` result from a comparison.

# The egal equality

Before we move forward it is worth to know that in Julia there is yet a third
notion of equality. It is invoked using the `===` comparison. This comparison
always returns a `Bool` value and tests if the passed arguments are identical,
in the sense that no program could distinguish them.

This distinction is most relevant for mutable types:

```
julia> x = [1]
1-element Vector{Int64}:
 1

julia> y = [1]
1-element Vector{Int64}:
 1

julia> x === x
true

julia> x === y
false

julia> x == x
true

julia> x == y
true
```

As you can see above `x` and `y` vectors are considered equal by `==` as they
have the same contents, but are not equal by `===` as they have a different
location in memory.

It is important to note here that immutable types are compared with `===` by
their contents on bit level, so we have the following:

```
julia> x = (1,)
(1,)

julia> y = (1,)
(1,)

julia> x === y
true
```

# Rules for designing custom types

In Julia it is very easy to define custom types. Therefore it is crucially
important, when doing so, to understand what are the default implementations
of the comparison operators. Here are the rules:
* `==` falls back to `===` by default;
* `isequal` falls back to `==` by default and additionally requires that the
  `hash` function is consistently defined;
* `<` falls back to `isless`.

As you can see, maybe somewhat surprisingly, the fallback implementations work
in different ways for `==` and `isequal` vs `<` and `isless` pairs. Also,
although `==` is not directly linked with `hash` it is indirectly linked to it
because `isequal` falls back to it.

The simple practical rule then is the following. If you define a new type then:
* always design `==`, `isequal` and `hash` functions jointly if you implement
  them (if you do not implement any you are safe as the default fallbacks for
  `===` and `hash` are designed in a consistent way);
* if you want your type to support ordering, always design `<` and `isless`
  jointly, and then also define the equality operators discussed in the bullet
  above.

# The `in` case study

As a special application of the above examples let me discuss the `in` function
here. It is very useful for testing if some value is found in some collection.
I am mentioning it because the implementation of `in` is quite tricky in Julia
1.6. Normally it uses `==` to test for equality except for certain collections,
like `Set` or `Dict`, which use `isequal` instead. In consequence we have the
following:

```
julia> 1 in [1, missing]
true

julia> 1 in [2, missing]
missing

julia> 1 in Set([1, missing])
true

julia> 1 in Set([2, missing])
false

julia> NaN in [NaN]
false

julia> NaN in Set([NaN])
true

julia> 0.0 in [-0.0]
true

julia> 0.0 in Set([-0.0])
false
```

These differences can be surprising so it is important to remember them. Let me
note that this is a quite relevant practical consideration because using of a
`Set` wrapper is a very common pattern for improving the performance of the
`in` test:

```
julia> x = rand(Int, 10^5);

julia> y = rand(Int, 10^5);

julia> in.(x, Ref(y)); # precompile

julia> @time in.(x, Ref(y)); # this is slow
  5.528883 seconds (6 allocations: 16.672 KiB)

julia> in.(x, Ref(Set(y))); # precompile

julia> @time in.(x, Ref(Set(y))); # this is fast
  0.006156 seconds (14 allocations: 2.267 MiB)
```

# The `isapprox` case study

Sometimes when comparing numeric values we are interested in checking if
they are approximately equal. The reason is that in some cases due to round-off
errors the sharp `==` equality is not what one might expect:

```
julia> 0.1 + 0.2 == 0.3
false
```

In such cases, when we are interested in testing of approximate equality the
`isapprox` function can be used:

```
julia> isapprox(0.1 + 0.2, 0.3)
true
```

You might ask how the approximate equality is defined. The rules are a bit
involved so I refer you to [the documentation][approx] for the details. Here
let me just note that you can control absolute tolerance, relative tolerance
and how `NaN` values are handled via `atol`, `reltol`, and `nans` keyword
arguments respectively.

# Conclusions

Designing comparison operators properly is one of the hardest tasks in every
programming language. In Julia the design covers a wide range of possible
scenarios that the user might want in practice. The cost of this flexible
design is that it takes some time to master it. I hope that after reading this
post you have enough understanding of the details to be able to confidently
work with comparison operators in Julia.

[approx]: https://docs.julialang.org/en/v1/base/math/#Base.isapprox
