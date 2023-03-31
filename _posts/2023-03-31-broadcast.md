---
layout: post
title:  "Broadcast fusion in Julia: all you need to know to avoid pitfalls"
date:   2023-03-31 10:11:22 +0200
categories: julialang
---

# Introduction

Broadcasting is a powerful feature of Julia and quickly becomes a tool of choice
of developers because it is convenient to use.

However, in more complicated scenarios it can be tricky. The issue is that because
of its power broadcasting has a complex design and even the Julia Manual has it covered in four
different parts to discuss different aspects of the functionality:
[here][jm1], [here][jm2], [here][jm3], and [here][jm4].

For this reason, some time ago I have written [a post about `@.`][blog] to
clarify its usage. A recent [discussion on Julia Discourse][post] prompted me to write
another post about this topic.

I will cover two things related to broadcast fusion today:
broadcasting of containers having different shape
and aliasing in broadcasted assignment.

The presented examples were tested under Julia 1.9.0-rc1.

# Broadcasting of containers having different shape

Let me start with an example:

```
julia> x = string.([1, 2, 3], ",", ["a" "b"], ":")
3×2 Matrix{String}:
 "1,a:"  "1,b:"
 "2,a:"  "2,b:"
 "3,a:"  "3,b:"

julia> y = rand.(Int8, 1:3)
3-element Vector{Vector{Int8}}:
 [-95]
 [65, -119]
 [-77, -78, -5]

julia> string.(x, y)
3×2 Matrix{String}:
 "1,a:Int8[-95]"           "1,b:Int8[-95]"
 "2,a:Int8[65, -119]"      "2,b:Int8[65, -119]"
 "3,a:Int8[-77, -78, -5]"  "3,b:Int8[-77, -78, -5]"

julia> string.([1, 2, 3], ",", ["a" "b"], ":", rand.(Int8, 1:3))
3×2 Matrix{String}:
 "1,a:Int8[-127]"            "1,b:Int8[-93]"
 "2,a:Int8[92, -91]"         "2,b:Int8[-88, 118]"
 "3,a:Int8[-104, -29, -38]"  "3,b:Int8[23, 109, 76]"
```

What we can see here is that the `x = string.([1, 2, 3], ",", ["a" "b"], ":")` operation creates a 3×2 matrix,
while `y = rand.(Int8, 1:3)` creates a 3-element vector. Since in Julia vectors are treated as columnar
the matrix and vector have matching dimensions and can be used in broadcasting.
The call `string.(x, y)` reuses elements of `y` in each row. As a result the suffix of every string
in the resulting matrix is the same for each row.

Therefore, you might be surprised that when you combine the expressions into one call
`string.([1, 2, 3], ",", ["a" "b"], ":", rand.(Int8, 1:3))` you get a different result.
Now each suffix is different (I used random numbers to show you that the suffix is different indeed).

What is the reason for this behavior? As is explained in the Julia Manual entries I linked to in the introduction
Julia performs broadcast fusion. This means that it behaves as if it created a single loop over two
dimensions of the output matrix and evaluates the expression:
`string(p, ",", q, ":", rand.(Int8, r))` for values of `p`, `q` and `r` appropriately determined
from the source data `[1, 2, 3]`, `["a" "b"]`, and `1:3` without caching them when doing the expansion
of the `1:3` vector over the second dimension. This means that we get different suffix in each cell.
Sometimes it is indeed desired, in other cases it can be surprising and not wanted.

First, let me explain how to resolve this issue. You can use the `identity` function (not-broacasted)
to break broadcast fusion behavior. Here is how you can do it:

```
julia> string.([1, 2, 3], ",", ["a" "b"], ":", identity(rand.(Int8, 1:3)))
3×2 Matrix{String}:
 "1,a:Int8[45]"           "1,b:Int8[45]"
 "2,a:Int8[-121, 60]"     "2,b:Int8[-121, 60]"
 "3,a:Int8[47, -25, 42]"  "3,b:Int8[47, -25, 42]"
```

The part of the expression wrapped in `identity` gets evaluated and then is fed into the enclosing
broadcasting expression.

In our example this changes the result of the operation, because we generated random numbers.
However, even if the result would not be impacted it can affect the performance significantly.
Have a look at this example (timings are after compilation):

```
julia> @time sin.(1:1000) .+ cos.((1:1000)');
  0.032546 seconds (2 allocations: 7.629 MiB)

julia> @time identity(sin.(1:1000)) .+ identity(cos.((1:1000)'));
  0.005688 seconds (4 allocations: 7.645 MiB)
```

What is the reason of the difference? In the first case both `sin` and `cos`
are evaluated 1,000,000 times (for each cell separately).
In the second example we have only 1000 calls of `sin` and `cos`.

You might ask when the default behavior might be desirable? It is useful when for example
you want to avoid aliasing. Take a look:

```
julia> m1 = tuple.([1 2], vcat.(1:3, 4:6))
3×2 Matrix{Tuple{Int64, Vector{Int64}}}:
 (1, [1, 4])  (2, [1, 4])
 (1, [2, 5])  (2, [2, 5])
 (1, [3, 6])  (2, [3, 6])

julia> push!(m1[1, 1][2], 100)
3-element Vector{Int64}:
   1
   4
 100

julia> m1
3×2 Matrix{Tuple{Int64, Vector{Int64}}}:
 (1, [1, 4, 100])  (2, [1, 4])
 (1, [2, 5])       (2, [2, 5])
 (1, [3, 6])       (2, [3, 6])

julia> m2 = tuple.([1 2], identity(vcat.(1:3, 4:6)))
3×2 Matrix{Tuple{Int64, Vector{Int64}}}:
 (1, [1, 4])  (2, [1, 4])
 (1, [2, 5])  (2, [2, 5])
 (1, [3, 6])  (2, [3, 6])

julia> push!(m2[1, 1][2], 100)
3-element Vector{Int64}:
   1
   4
 100

julia> m2
3×2 Matrix{Tuple{Int64, Vector{Int64}}}:
 (1, [1, 4, 100])  (2, [1, 4, 100])
 (1, [2, 5])       (2, [2, 5])
 (1, [3, 6])       (2, [3, 6])
```

As you can see, in this case we typically would want `vcat` to be called separately for every cell.
When we broke broadcast fusion with `identity(vcat.(1:3, 4:6))` we get the same vector in every cell in a single row,
which could lead to hard-to-catch bugs.

Another question is when broadcast fusion is useful from the performance perspective?
The answer is that in simple calls like (timings are after compilation):

```
julia> @time cot.(sin.(cos.(tan.(1:10^6))));
  0.057595 seconds (2 allocations: 7.629 MiB)
```

we avoid unnecessary allocation of intermediate objects. We can simulate non-fused performance
by injecting `identity` to see the difference:

```
julia> @time cot.(identity(sin.(identity(cos.(identity(tan.(1:10^6)))))));
  0.085532 seconds (8 allocations: 30.518 MiB)
```

# Aliasing in broadcasted assignment

Another potential issue is aliasing in broadcasted assignment `.=`. Have a look at this example:

```
julia> x = [1 2; 3 4]
2×2 Matrix{Int64}:
 1  2
 3  4

julia> x .= sum.(Ref(x))
2×2 Matrix{Int64}:
 10  35
 19  68

julia> x = [1 2; 3 4]
2×2 Matrix{Int64}:
 1  2
 3  4

julia> x .= identity(sum.(Ref(x)))
2×2 Matrix{Int64}:
 10  10
 10  10
```

In the first case of `x .= sum.(Ref(x))`, as we already discussed, `sum.(Ref(x))`
gets executed for each cell of `x` matrix. Now, since we use `.=` broadcasted assignment
the operation happens in-place, which means that `x` gets updated during the process
and consecutive `sum.(Ref(x))` calls use changed `x`. Again, breaking broadcasting
fusion with `identity(sum.(Ref(x)))` forces Julia to materialize the sum before
doing the outer broadcasted assignment and we get `10` in every cell.

To give another example let us have a look how we can fill a vector with consecutive
powers of 2 (of course there are better ways to do it):

```
julia> x = [1, 0, 0, 0, 0, 0, 0]
7-element Vector{Int64}:
 1
 0
 0
 0
 0
 0
 0

julia> x .= sum.(Ref(x))
7-element Vector{Int64}:
  1
  1
  2
  4
  8
 16
 32
```

# Conclusions

In summary, it is important to keep in mind that Julia performs broadcast fusion when
operating on several broadcasted function calls that are chained together.

This broadcast fusion in general improves performance and reduces allocations, but in some
cases it is not desirable. The most common scenarios are:

* when we broadcast operations over containers of different dimensions
  (when it can degrade performance, or lead to different results).
* when we perform broadcasted assignment to a container that is also used on right hand side of
  an expression (when it can lead to unexpectedly incorrect results).

As I have shown, in such cases one of the ways to fix the problem is to break broadcast fusion
by injecting a non-broadcasted function call forcing materialization of intermediate results
of computation. The `identity` function can be used to achieve this effect.

[post]: https://discourse.julialang.org/t/unexpected-broadcasting-behavior-involving-eachrow/96781
[blog]: https://bkamins.github.io/julialang/2022/06/24/broadcasting.html
[jm1]: https://docs.julialang.org/en/v1/manual/mathematical-operations/#man-dot-operators
[jm2]: https://docs.julialang.org/en/v1/manual/arrays/#Broadcasting
[jm3]: https://docs.julialang.org/en/v1/manual/functions/#man-vectorized
[jm4]: https://docs.julialang.org/en/v1/manual/interfaces/#man-interfaces-broadcasting
