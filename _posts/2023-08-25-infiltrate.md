---
layout: post
title:  "Infiltrator.jl: a no-nonsense debugging"
date:   2023-08-25 16:11:23 +0200
categories: julialang
---

# Introduction

During [JuliaCon 2023][juliacon2023] I have had several
discussions about setup of my working environment when
I develop Julia code. One of the common questions was
tools that I use to debug my code.

With Julia an important aspect of the debugging tool
one uses is to ensure that it does not have a significant
impact on the performance of code execution. Keeping this
condition in mind, I find [Infiltrator.jl][infiltrator]
quite convenient. In this post I want to show an example
how this package can be used.

The post was written using Julia 1.9.2 and Infiltrator.jl 1.6.4.
Note that you should use the examples that I present in the terminal
using the standard Julia REPL.

# An example project

Let us write a simple function that merges two sorted vectors into
a new sorted vector:

```
function mergesorted(x::T, y::T) where {T<:Vector}
    @assert issorted(x) && issorted(y)
    z = T(undef, length(x) + length(y))
    ix, iy, iz = 1, 1, 1
    for iz in eachindex(z)
        if x[ix] < y[iy]
            z[iz] = x[ix]
            ix += 1
        else
            z[iz] = y[iy]
            iy += 1
        end
    end
    return z
end
```

Now test it on some example data:

```
julia> mergesorted([1, 3], [2, 4])
ERROR: BoundsError: attempt to access 2-element Vector{Int64} at index [3]
```

We see that we get a problem when we try to get an element from a
vector with a too large index. Let us try to infiltrate this issue
but providing an appropriate condition when we want to investigate
the state of our function:

```
using Infiltrator
function mergesorted(x::T, y::T) where {T<:Vector}
    @assert issorted(x) && issorted(y)
    z = T(undef, length(x) + length(y))
    ix, iy, iz = 1, 1, 1
    for iz in eachindex(z)
        @infiltrate ix > length(x) || iy > length(y)
        if x[ix] < y[iy]
            z[iz] = x[ix]
            ix += 1
        else
            z[iz] = y[iy]
            iy += 1
        end
    end
    return z
end
```

The magic of `@infiltrate ix > length(x) || iy > length(y)`
is that the execution of the `mergesorted` function will
be interrupted at the moment when the passed condition is met.
Our condition is that either `ix` or `iy` index gets too large.

Let us run our test with an updated definition of the function:

```
julia> mergesorted([1, 3], [2, 4])
Infiltrating mergesorted(x::Vector{Int64}, y::Vector{Int64})
  at REPL[11]:6

infil> @locals
- x::Vector{Int64} = [1, 3]
- T::DataType = Vector{Int64}
- iz::Int64 = 4
- y::Vector{Int64} = [2, 4]
- z::Vector{Int64} = [1, 2, 3, 2041652468496]
- iy::Int64 = 2
- ix::Int64 = 3

infil> @continue

ERROR: BoundsError: attempt to access 2-element Vector{Int64} at index [3]
```

We see that we have a problem that we get beyond the end of the
`x` vector while the last element of the `y` vector is still not
processed. It looks that we need to check if we are beyond
the end of the `x` vector, and if it is the case jump right
to the `else` part of the code:

```
function mergesorted(x::T, y::T) where {T<:Vector}
    @assert issorted(x) && issorted(y)
    z = T(undef, length(x) + length(y))
    ix, iy, iz = 1, 1, 1
    for iz in eachindex(z)
        @infiltrate ix > length(x) || iy > length(y)
        if ix <= length(x) && x[ix] < y[iy]
            z[iz] = x[ix]
            ix += 1
        else
            z[iz] = y[iy]
            iy += 1
        end
    end
    return z
end
```

Let us run the code:

```
julia> mergesorted([1, 3], [2, 4])
Infiltrating mergesorted(x::Vector{Int64}, y::Vector{Int64})
  at REPL[13]:6

infil> @locals
- x::Vector{Int64} = [1, 3]
- T::DataType = Vector{Int64}
- iz::Int64 = 4
- y::Vector{Int64} = [2, 4]
- z::Vector{Int64} = [1, 2, 3, 8589934594]
- iy::Int64 = 2
- ix::Int64 = 3

infil> @continue

4-element Vector{Int64}:
 1
 2
 3
 4
```

This time things seem to work as expected. Let us thus turn-off
infiltration and run some randomized tests:

```
function mergesorted(x::T, y::T) where {T<:Vector}
    @assert issorted(x) && issorted(y)
    z = T(undef, length(x) + length(y))
    ix, iy, iz = 1, 1, 1
    for iz in eachindex(z)
        if ix <= length(x) && x[ix] < y[iy]
            z[iz] = x[ix]
            ix += 1
        else
            z[iz] = y[iy]
            iy += 1
        end
    end
    return z
end
```

And run the tests:

```
julia> using Random

julia> using Test

julia> Random.seed!(1234);

julia> for i in 1:10
           x = sort!(rand(rand(1:10)))
               y = sort!(rand(rand(1:10)))
           @assert mergesorted(x, y) == sort!([x; y])
       end
ERROR: BoundsError: attempt to access 3-element Vector{Float64} at index [4]
```

As a side note, the `rand(rand(1:10))` is a convenient pattern for generating
random vectors of random length.

Going back to our main topic we see that we still get a problem.
How to diagnose it? This time, as an example,
let me show how to turn infiltration on when an error happens:

```
function mergesorted(x::T, y::T) where {T<:Vector}
    @assert issorted(x) && issorted(y)
    z = T(undef, length(x) + length(y))
    ix, iy, iz = 1, 1, 1
    for iz in eachindex(z)
        try
            if ix <= length(x) && x[ix] < y[iy]
                z[iz] = x[ix]
                ix += 1
            else
                z[iz] = y[iy]
                iy += 1
            end
        catch e
            @infiltrate
            rethrow(e)
        end
    end
    return z
end
```

Now run the same test code:

```
julia> Random.seed!(1234);

julia> for i in 1:10
           x = sort!(rand(rand(1:10)))
               y = sort!(rand(rand(1:10)))
           @assert mergesorted(x, y) == sort!([x; y])
       end
Infiltrating mergesorted(x::Vector{Float64}, y::Vector{Float64})
  at REPL[31]:15

infil> @locals
- x::Vector{Float64} = [0.6932923170086805, 0.7600131804670265]
- T::DataType = Vector{Float64}
- iz::Int64 = 4
- y::Vector{Float64} = [0.11679226454435099, 0.20295936651757684, 0.43552097477865936]
- e::BoundsError = BoundsError([0.11679226454435099, 0.20295936651757684, 0.43552097477865936], (4,))
- z::Vector{Float64} = [0.11679226454435099, 0.20295936651757684, 0.43552097477865936, 5.0e-324, 5.0e-324]
- iy::Int64 = 4
- ix::Int64 = 1

infil> @continue

ERROR: BoundsError: attempt to access 3-element Vector{Float64} at index [4]
```

Ah - we can now see where the problem is. We have not covered the case when `iy` is greater than
the length of `y`. It seems we are close. Let us make a final attempt:

```
function mergesorted(x::T, y::T) where {T<:Vector}
    @assert issorted(x) && issorted(y)
    z = T(undef, length(x) + length(y))
    ix, iy, iz = 1, 1, 1
    for iz in eachindex(z)
        if ix <= length(x) && (iy > length(y) || x[ix] < y[iy])
            z[iz] = x[ix]
            ix += 1
        else
            z[iz] = y[iy]
            iy += 1
        end
    end
    return z
end
```

We are ready for an even more comprehensive test:

```
julia> Random.seed!(1234);

julia> for i in 1:10_000
           x = sort!(rand(rand(0:10)))
               y = sort!(rand(rand(0:10)))
           @assert mergesorted(x, y) == sort!([x; y])
       end
```

This time we got no errors, so we can be relatively confident that our code works well.

# Conclusions

I find [Infiltrator.jl][infiltrator] useful because it is a lightweight solution.
I did not discuss all of its features. However, even these minimal examples
I have shown are quite nice in practice:

1. You can use `@infiltrate` with a condition.
2. You can put `@infiltrate` in a `try-catch-end` block
   to be able to infiltrate into the state of your computations
   at the moment an exception is thrown (a thing that I quite often need in practice).

Happy debugging!

[juliacon2023]: https://juliacon.org/2023/
[infiltrator]: https://github.com/JuliaDebug/Infiltrator.jl
