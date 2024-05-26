---
layout: post
title:  "A simple coin tossing came"
date:   2024-05-24 06:32:44 +0200
categories: julialang
---

# Introduction

I have been writing my blog for over 4 years now (without missing a single week).
My first post was on May 10, 2020, you can find it [here][firstpost].

There is a small change in how I distribute my content. Starting from last week I made the repository of my blog public,
so if you find any mistake please do not hesitate to open a Pull Request [here][repo].

To celebrate this I decided to go back to my favorite topic - mathematical puzzles.
Today I use a classic coin tossing game example

The post was written under Julia 1.10.1 and StatsBase.jl 0.34.4, and FreqTables.jl 0.4.6, BenchmarkTools.jl 1.5.0.

# The problem

Assume Alice and Bob toss a fair coin. Alice wins if after tossing a head (`H`) tail (`T`) is tossed, that is we see a `HT` sequence.
Bob wins if two consecutive heads are tossed, that is we see a `HH` sequence.

The questions are:

* Who is more likely to win this game?
* If only Alice played, how long, on the average, would she wait till `HT` was tossed?
* If only Bob played, how long, on the average, would she wait till `HH` was tossed?

Let us try to answer this questions using Julia.

# Both players play

This code simulates the situation when Alice and Bob play together:

```
function both()
    a = rand(('H', 'T'))
    while true
        b = rand(('H', 'T'))
        if a == 'T'
            a = b
        else
            return b == 'H' ? "Bob" : "Alice"
        end
    end
end
```

The `both` function returns `"Bob"` if Bob wins, and `"Alice"` otherwise.
From the code it should be already clear that both players have the same probability of winning.
The only way to terminate the simulation is `return b == 'H' ? "Bob" : "Alice"` and this condition is symmetric
with respect to Alice and Bob. Let us confirm this by running a simulation:

```
julia> using FreqTables, Random

julia> Random.seed!(1234);

julia> freqtable([both() for _ in 1:100_000_000])
2-element Named Vector{Int64}
Dim1  │
──────┼─────────
A     │ 50000012
B     │ 49999988
```

Indeed, the number of times Alice and Bob win seem to be the same.

# Alice's waiting time

Now let us check how long, on the average, Alice has to wait to see the `HT` sequence. Here is Alice's simulator:

```
function alice()
    a = rand(('H', 'T'))
    i = 1
    while true
        b = rand(('H', 'T'))
        i += 1
        a == 'H' && b == 'T' && return i
        a = b
    end
end
```

Let us check it:

```
julia> using StatsBase

julia> describe([alice() for _ in 1:100_000_000])
Summary Stats:
Length:         100000000
Missing Count:  0
Mean:           3.999890
Std. Deviation: 2.000032
Minimum:        2.000000
1st Quartile:   2.000000
Median:         3.000000
3rd Quartile:   5.000000
Maximum:        31.000000
Type:           Int64
```

So it seems that, in expectation, Alice finishes her game in 4 tosses.
Can we expect the same for Ben (as we remember - if they play together they have the same chances of finishing first)?
Let us see.

# Alice's waiting time

Now let us check how long, on the average, Bob has to wait to see the `HH` sequence. Here is Bob's simulator:

```
function bob()
    a = rand(('H', 'T'))
    i = 1
    while true
        b = rand(('H', 'T'))
        i += 1
        a == 'H' && b == 'H' && return i
        a = b
    end
end
```

Let us check it:

```
julia> describe([bob() for _ in 1:100_000_000])
Summary Stats:
Length:         100000000
Missing Count:  0
Mean:           5.999915
Std. Deviation: 4.690177
Minimum:        2.000000
1st Quartile:   2.000000
Median:         5.000000
3rd Quartile:   8.000000
Maximum:        87.000000
Type:           Int64
```

To our surprise, Bob needs 6 coin tosses, on the average, to see `HH`.

What is the reason of this difference? Assume we have just tossed `H`. Start with Bob. If we hit `H` we finish. If we hit `T` we then need to wait till we see `H` again to be able to consider finishing.
However, if we are Alice if we hit `T` we finish, but if we hit `H` we do not have to wait for anything - we are already in a state that gives us a chance to finish the game in the next step.

# Conclusions

The difference between joint games and separate games is a bit surprising and I hope you found it interesting if you have not seen this puzzle before.
Today I have approached this problem using simulation. However, it is easy to write down a [Markov chain][mc] representation of all three scenarios and solve them analytically.
I encourage you to try doing this exercise.

PS.

In the code I use the `rand(('H', 'T'))` form to generate randomness. It is much faster than e.g. writing `rand(["H", "T"])` (which would be a first instinct), for two reasons:

* using `Char` instead of `String` is a more lightweight option;
* using `Tuple` instead of `Vector` avoids allocations.

Let us see a comparison of timing (I cut out the histograms from the output):

```
julia> using BenchmarkTools

julia> @benchmark rand(('H', 'T'))
BenchmarkTools.Trial: 10000 samples with 1000 evaluations.
 Range (min … max):  1.900 ns … 233.700 ns  ┊ GC (min … max): 0.00% … 0.00%
 Time  (median):     2.500 ns               ┊ GC (median):    0.00%
 Time  (mean ± σ):   3.316 ns ±   2.772 ns  ┊ GC (mean ± σ):  0.00% ± 0.00%

 Memory estimate: 0 bytes, allocs estimate: 0.

julia> @benchmark rand(["H", "T"])
BenchmarkTools.Trial: 10000 samples with 999 evaluations.
 Range (min … max):  15.816 ns …  2.086 μs  ┊ GC (min … max): 0.00% … 96.30%
 Time  (median):     19.019 ns              ┊ GC (median):    0.00%
 Time  (mean ± σ):   23.041 ns ± 60.335 ns  ┊ GC (mean ± σ):  9.44% ±  3.63%

 Memory estimate: 64 bytes, allocs estimate: 1.
```

In this case we could also just use `rand(Bool)` (as the coin is fair and has only two states):

```
julia> @benchmark rand(Bool)
BenchmarkTools.Trial: 10000 samples with 1000 evaluations.
 Range (min … max):  1.700 ns … 131.000 ns  ┊ GC (min … max): 0.00% … 0.00%
 Time  (median):     3.200 ns               ┊ GC (median):    0.00%
 Time  (mean ± σ):   3.218 ns ±   2.112 ns  ┊ GC (mean ± σ):  0.00% ± 0.00%

 Memory estimate: 0 bytes, allocs estimate: 0.
```

but as you can see `rand(('H', 'T'))` has a similar speed and leads to a much more readable code.

[firstpost]: https://bkamins.github.io/julialang/2020/05/10/julia-project-environments.html
[repo]: https://github.com/bkamins/bkamins.github.io
[mc]: https://en.wikipedia.org/wiki/Markov_chain
