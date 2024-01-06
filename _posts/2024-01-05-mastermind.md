---
layout: post
title:  "Code cleanup for the New Year"
date:   2023-12-29 10:13:42 +0200
categories: julialang
---

# Introduction

I have done a small cleanup of my folders with code for the New Year.
In the process I have discovered some old Julia code that I have written and forgotten about.
One of such scripts is a simple [Mastermind][mastermind] solver.
So I decided to share it in this post.

The code was tested under Julia 1.10.0, Combinatorics.jl 1.0.2, FreqTables.jl 0.4.6, and StatsBase.jl 0.34.2.

# The Mastermind game

I assume you know how [Mastermind][mastermind] is played.
If not - I recommend you read the [Wikipedia description][mastermind] before continuing.

Let me briefly summarize the version of the game that I used as a reference
(this is a version that was popular in Poland when I was young):

- The game is played using 8 colors of pegs.
- The code that we are required to break (a solution) consists of 5 pegs of which each has to have a different color.
- You are allowed to guess the solution by showing 5 pegs (that also have to be of different colors)
  and you get two numbers: black representing number of pegs matching color and position between your guess and a solution,
  and white dots representing pegs that have a correct color, but wrong position.
- The game is finished when your guess matches the solution (i.e. you get 5 blacks and 0 whites).
  The outcome of the game is the number of guesses you have made.

# The problem

In my code I wanted to compare two strategies. Both of them use a concept of available options,
so let us define it first.

First note that we have 5 pegs and each of them has one of 8 unique colors that we have:

```
julia> binomial(8, 5) * factorial(5)
6720
```

initially available options as solution when we start a game. After each move we can check which of options
match the responses to your guesses. This matching reduces the number of available options.

The first strategy we will consider is to randomly pick a guess that is still an available option.
This strategy is in practice possible to follow (up to ensuring randomness of choice which can be hard :))
even without the help of the computer. At any moment during the game you just need to show a guess that
is consistent with the answers you already got.

The second strategy, called minimax, works as follows. Before making a move we consider every initially
available option as a possible candidate. For each candidate we check what black-white combination
we would get against all available options. From this we can get a distribution of number of black-white
combinations that a given guess would produce. We sort this distribution in a descending order and pick
the guess that produces the lexicographically smallest value.

Let me explain this by example. Assume we consider guesses `A`, `B`, and `C` and have still 20 available options.

For the guess `A` we count that we could have 10 answers of the form 2 black & 2 white, 5 answers 1 black & 2 white,
and 5 answers 2 black & 1 white. Thus for option `A` the encoding is `[10, 5, 5]`.

For the guess `B` we count that we could have 3 answers of the form 2 black & 3 white, 5 answers 1 black & 2 white,
and 12 answers 2 black & 1 white. Thus for option `B` the encoding is `[12, 5, 3]`.

For the guess `C` we count that we could have 10 answers of the form 1 black & 1 white, 8 answers 2 black & 2 white,
and 2 answers 1 black & 1 white. Thus for option `C` the encoding is `[10, 8, 2]`.

Therefore since lexicographically `[10, 5, 5] < [10, 8, 2] < [12, 5, 3]` the option `A` is best (and we would pick it)
and the option `B` is worst.

The rationale behind this strategy is that we want to minimize the maximum possible number of available options we will be
left with in the next round.

Clearly minimax strategy is much more complex. For sure it is hard to be used without computer assistance. The question is
if it is better than random strategy and, if so, by how much.

# The code

Let us start with the code that implements the game.

The first part is environment setup:

```
using Combinatorics
using FreqTables
using Random
using StatsBase

struct Position
    pos::NTuple{5, Int8}
    pegs::NTuple{8, Bool}
end

const ALL_POSITIONS = let
    combs = combinations(Int8.(1:8), 5)
    perms = reduce(vcat, (collect∘permutations).(combs))
    [Position(Tuple(pos), ntuple(i -> i in pos, 8)) for pos in perms]
end
```

In this part of code we loaded all required packages. Next we define `Position` structure.
It encodes the colors (as values from 1 to 8) passed for each peg. The `pos` field
directly encodes the position. The `pegs` field is a helper code containing information
if a given color was used in `pos` (to make it simpler to later use this data structure).
Finally we create `ALL_POSITIONS` vector which contains all 6720 available options at the start of the game.

Next define a function counting the number of black and white matches between the guess and the solution:

```
function match(p1::Position, p2::Position)
    pos1, pegs1 = p1.pos, p1.pegs
    pos2, pegs2 = p2.pos, p2.pegs
    black = sum(x -> x[1] == x[2], zip(p1.pos, p2.pos))
    matches = sum(x -> x[1] & x[2], zip(p1.pegs, p2.pegs))
    return black, matches - black
end
```

Here we see why both `pos` and `pegs` values are useful. We use `pos` to compute the number of black matches.
Then we use `pegs` to compute the number of all matches (disregarding the location). Therefore `matches - black`
gives ut the number of white matches.

We are now ready to implement our decision rule functions:

```
random_rule(options::AbstractVector{Position}) = rand(options)

function minimax_rule(options::AbstractVector{Position})
    length(options) == 1 && return only(options)
    best_guess = options[1]
    best_guess_lengths = [length(options)]
    for guess in shuffle(options)
        states = Dict{Tuple{Int,Int},Int}()
        for opt in options
            resp = match(guess, opt)
            states[resp] = get(states, resp, 0) + 1
        end
        current_guess_lengths = sort!(collect(values(states)), rev=true)
        if current_guess_lengths < best_guess_lengths
            best_guess = guess
            best_guess_lengths = current_guess_lengths
        end
    end
    return best_guess
end
```

The `random_rule` is simple. We just randomly pick one of the available options.

The `minimax_rule` code is more complex. We traverse `guess` taken from `options` in random order.
For each `guess` in the `states` dictionary we keep the count of black-white answers generated by
this guess. Then `current_guess_lengths` keeps the vector of combination counts sorted in a descending order.
After checking all `guess` values we return the `best_guess` value which holds the guess with the smallest value of
`current_guess_lengths`.

The final part of the code is a rule testing function:

```
function run_test(rule::Function, solution::Position)
    options = ALL_POSITIONS
    moves = 0
    while true
        moves += 1
        guess::Position = rule(options)
        guess == solution && return moves
        resp = match(guess, solution)
        options = filter(opt -> match(guess, opt) == resp, options)
        @assert solution in options
    end
end
```

In the `run_test` we get the `rule` function and the `solution` to the game. Next iteratively we ask the `rule` function what next move it picks. If it is the `solution`
then we return the number of `moves` taken. Otherwise we filter the available options, stored in the `options` variable to leave only those that match the responses
received by matching the `guess` against the `solution`.

Now the big question is if the extra complexity of `minimax_rule` is worth the effort?

# The test

Now we are ready to test the `random_rule` against the `minimax_rule`. One important observation that we can make is that it is enough to run the `run_test` function
with only one starting `solution`, which I pick to be `ALL_OPTIONS[1]`. The reason is that the problem is fully symmetric (i.e. all possible solutions are indistinguishable with respect to recoloring)
and both `random_rule` and `minimax_rule` are fully randomized. For `random_rule` it is clear. For `minimax_rule` the key line of code that ensures it is `for guess in shuffle(options)`.
In this way we ensure that we do not favor any options. E.g. if we did `for guess in options` then clearly we would favor `options[1]` option (as the game would always finish in the first move).

Now let us run the tests:

```
julia> Random.seed!(1234);

julia> @time random_res = [run_test(random_rule, ALL_POSITIONS[1]) for i in 1:1_000_000];
138.395938 seconds (7.13 M allocations: 97.072 GiB, 1.59% gc time, 0.11% compilation time)

julia> @time minimax_res = [run_test(minimax_rule, ALL_POSITIONS[1]) for i in 1:1_000];
1162.463279 seconds (65.08 M allocations: 20.119 GiB, 0.03% gc time, 0.04% compilation time)
```

As you can see I run one thousand times more `random_rule` than `minimax_rule` as the former is much faster (and still ten times more time is spent in gathering the results for `minimax_rule`).

Now let us collect some statistics about the both approaches to the solution:

```
julia> proptable(random_res)
9-element Named Vector{Float64}
Dim1  │
──────┼─────────
1     │ 0.000156
2     │ 0.002364
3     │  0.02251
4     │ 0.139851
5     │  0.39269
6     │ 0.350372
7     │  0.08736
8     │ 0.004646
9     │   5.1e-5

julia> proptable(minimax_res)
6-element Named Vector{Float64}
Dim1  │
──────┼──────
3     │ 0.015
4     │ 0.138
5     │ 0.476
6     │ 0.336
7     │ 0.034
8     │ 0.001
```

Note that the probability of solving the puzzle in one move is (we hit the solution by chance):

```
julia> 1 / (binomial(8, 5) * factorial(5))
0.00014880952380952382
```

It is pretty accurately approximated for `random_res`. For `minimax_res` we did not get even one such observation because our sample was too small.
In general, what we can see is that the probabilities of numbers of moves are similar (keeping in mind the fact that `minimax_res` has only 1000 observations),
except that random rule less frequently finishes in 5 moves, and more frequently in 7 moves. However, the difference is not very large.
This is also reflected by the fact that the mean time required to finish the game is only slightly lower for minimax strategy:

```
julia> mean(random_res)
5.346647

julia> mean(minimax_res)
5.239
```

# Conclusions

Interestingly, for the variant of Mastermind we considered a random strategy is not much worse than a much more expensive minimax approach.
A limitation of our minimax approach is that it is doing only one-step-ahead optimization. As an extension to our exercise you could consider
checking how much better would be a full-depth minimax algorithm.

Happy hacking!

[mastermind]: https://en.wikipedia.org/wiki/Mastermind_(board_game)
