---
layout: post
title:  "Coin-tossing game: a numerical approach"
date:   2024-06-14 15:12:34 +0200
categories: julialang
---

# Introduction

Today I decided to follow up on my [last post][post] solving a coin-tossing game.
This time, instead of simulation I want to use numerical approach
(and so probably a bit harder).

The post was written under Julia 1.10.1 and Graphs.jl 1.11.0.

# The problem

Let me describe the setting of a game first (it is an extension of [this post][post]).

Assume Alice and Bob toss a fair coin. In each toss head (`h`) or tail (`t`) can show up with equal probability.

Alice and Bob choose some sequence of `h` and `t` they are waiting for. We assume that the chosen sequences have the same length and are different.
For example Alice could choose `htht` and Bob `tthh`.

The winner of the game is the person who saw their sequence first.

The question we ask if for a fixed sequence length `n` we can get cycles, that is, for example, that sequence `s1` beats `s2`, `s2` beats `s3`, and `s3` beats `s1`.

To answer this question we will represent the game as a Markov process.

# Step 1: non-terminating Markov chain

First we create a transition matrix of a Markov chain tracking current `n` element sequence in the game we consider.
Here is the code:

```
function markov(size::Integer)
    idx2states = vec(join.(Iterators.product([['h', 't'] for _ in 1:size]...)))
    states2idx = Dict(idx2states .=> eachindex(idx2states))
    P = zeros(2^size, 2^size)
    for state in idx2states
        for next in ("h", "t")
            nextstate = chop(state, head=1, tail=0) * next
            P[states2idx[state], states2idx[nextstate]] = 0.5
        end
    end
    return P, idx2states, states2idx
end
```

What we do in it is as follows:
1. `idx2states` vector keeps track of all `h` and `t` sequences that have length `n` (i.e. it is a mapping from state number to state signature).
2. `states2idx` is an inverse mapping - from state signature to state number.
3. `P` is transition matrix of our chain. Note that from the sequence `ab...` (where all elements are `h` or `t`) we go to sequence `b...h` or `b...t` with equal probability.

# Step 2: terminating Markov chain

We now need to create a function that is aware of Alice's and Bob's chosen sequences and make them terminating. We want to compute the probabilities of ending up in Alice's and Bob's state.
Here is the code:

```
function game(P, states2idx, alice, bob)
    P_game = copy(P)
    alice_idx, bob_idx = states2idx[alice], states2idx[bob]
    P_game[alice_idx, :] .= 0.0
    P_game[alice_idx, alice_idx] = 1.0
    P_game[bob_idx, :] .= 0.0
    P_game[bob_idx, bob_idx] = 1.0
    n = length(states2idx)
    terminal = fill(1 / n, 1, n) * P_game^(2^30)
    return terminal[states2idx[alice]], terminal[states2idx[bob]]
end
```

Note that we first update the `P_game` matrix to make `alice_idx` and `bob_idx` states terminating. Then, since I was lazy, we assume we make `2^30` steps of the process (fortunately in Julia it is fast).
Observe that initially all states are equally probably, so `terminal` matrix keeps information about long term probabilities of staying in all possible states.
We extract the probabilities of Alice's and Bob's states and return them.

# Step 3: looking for cycles

We are now ready for a final move. We can consider all possible preferred sequences of Alice and Bob and create a graph that keeps track of which sequences beat other sequences:

```
using Graphs

function analyze_game(size::Integer, details::Bool=true)
    P, idx2states, states2idx = markov(size)
    g = SimpleDiGraph(length(states2idx))
    details && println("\nWinners:")
    for alice in idx2states, bob in idx2states
        alice > bob || continue
        alice_win, bob_win = game(P, states2idx, alice, bob)
        if alice_win > 0.51
            winner = "alice"
            add_edge!(g, states2idx[alice], states2idx[bob])
        elseif bob_win > 0.51
            winner = "bob"
            add_edge!(g, states2idx[bob], states2idx[alice])
        else
            winner = "tie (or close :))"
        end
        details && println(alice, " vs ", bob, ": ", winner)
    end
    cycles = simplecycles(g)
    if !isempty(cycles)
        min_len = minimum(length, cycles)
        filter!(x -> length(x) == min_len, cycles)
    end
    println("\nCycles:")
    for cycle in cycles
        println(idx2states[cycle])
    end
end
```

Note that I used `0.51` threshold for detection of dominance of one state over the other. We could do it better, but in practice for small `n` it is enough and working this way is simpler numerically.
What this threshold means is that we want to be "sure" that one player beats the other.
In our code we do two things:
* optionally print the information which state beats which state;
* print information about cycles found in beating patterns (we keep only cycles of the shortest length).

Let us check the code. Start with sequences of length 2:

```
julia> analyze_game(2)

Winners:
th vs hh: alice
th vs ht: tie (or close :))
ht vs hh: tie (or close :))
tt vs hh: tie (or close :))
tt vs th: tie (or close :))
tt vs ht: bob

Cycles:
```

We see that only `th` beats `hh` and `ht` beats `tt` (this is a symmetric case). We did not find any cycles.

Let us check 3:

```
julia> analyze_game(3)

Winners:
thh vs hhh: alice
thh vs hth: tie (or close :))
thh vs hht: alice
thh vs htt: tie (or close :))
hth vs hhh: alice
hth vs hht: bob
tth vs hhh: alice
tth vs thh: alice
tth vs hth: alice
tth vs hht: tie (or close :))
tth vs tht: alice
tth vs htt: bob
hht vs hhh: tie (or close :))
tht vs hhh: alice
tht vs thh: tie (or close :))
tht vs hth: tie (or close :))
tht vs hht: bob
tht vs htt: tie (or close :))
htt vs hhh: alice
htt vs hth: tie (or close :))
htt vs hht: bob
ttt vs hhh: tie (or close :))
ttt vs thh: bob
ttt vs hth: bob
ttt vs tth: tie (or close :))
ttt vs hht: bob
ttt vs tht: bob
ttt vs htt: bob

Cycles:
["thh", "hht", "htt", "tth"]
```

We now have the cycle. The shortest cycle has length 4 and it is unique. Let us see what happens for patterns of length 4 (I suppress printing the details as there are too many of them):

```
julia> analyze_game(4, false)

Cycles:
["thhh", "hhth", "hthh"]
["thhh", "hhtt", "ttth"]
["hhth", "hthh", "thht"]
["hhth", "thtt", "tthh"]
["hthh", "hhtt", "thth"]
["hthh", "hhtt", "ttht"]
["hthh", "hhtt", "ttth"]
["thht", "hhtt", "ttth"]
["htht", "thtt", "tthh"]
["thtt", "tthh", "hhht"]
["thtt", "htth", "ttht"]
["thtt", "httt", "ttht"]
["tthh", "hhht", "htth"]
["tthh", "hhht", "httt"]
```

In this case we have many cycles that are even shorter as they have length three.

# Conclusions

The conclusion is that the game is slightly surprising. We can have cycles of dominance between sequences. I hope you liked this example. Happy summer!

[post]: https://bkamins.github.io/julialang/2024/06/07/probability2.html
