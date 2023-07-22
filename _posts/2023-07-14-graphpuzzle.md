---
layout: post
title:  "Yet another graph puzzle"
date:   2023-07-14 06:42:44 +0200
categories: julialang
---

# Introduction

Recently I have written a [post][lastpost] that presented a puzzle
that can be solved using graph analysis. I received quite nice feedback
from my readers and one of them has shared with me another interesting
puzzle that can be solved using graphs.

Today I want to discuss this puzzle. Like in the [recent post][lastpost]
I solve the problem both analytically and computationally.

The post was written under Julia 1.9.2 and Graphs.jl 1.8.0.

# The problem

The problem my friend shared with me goes as follows:

> Seven people A, B, C, E, F, G, and H visit person D.
> Upon investigation they report that they have seen
> the following persons during their visits:
>
> * A met B, C, F, and G;
> * B met A, C, E, F, and H;
> * C met A, B, and E;
> * E met B, C, and F;
> * F met A, B, E, and H;
> * G met A and H;
> * H met B, F, and G.
>
> Now, interestingly, they all also report that all of
> them visited D only once for a continuous period of time.
> The questions are:
>
> 1. Show that this is not possible,
>    i.e. that one of the visitors must have made more than one visit.
> 2. Show that it is enough to assume that
>    a single person visited D several times and identify this person.

# Analytical solution

First, note that the answers about who has seen whom are consistent
among all reporting people.
We can visualize them using the following graph:

![Visit graph](/assets/2023-07-14-graph.png)

To start let us think why it is not possible that every person visited D only once.
Consider visitors A, B, H, and G. We see that A and H have not met.
This means that their periods of visits are disjoint. Since B and G
both saw A and H, then assuming they made only one visit,
they both must have been present in the period between visits of A and H.
But this would mean that in that period B and H would have seen each other,
which is not the case.
Therefore at least one person must have visited D more than once.

We notice that we have a problem because we have a cycle of length 4 in the graph,
namely A-B-G-H, that does not contain shorter cycles inside
(i.e. an edge connecting A and H or B and G). By the reasoning
we have presented above all such cycles will be problematic. There are three such
cycles: A-B-G-H, A-G-H-F, and A-C-E-F. We notice that in all of these cycles
A is present and this is the only person that is commonly present in all of them.
This means that:

* removing A from the graph (i.e. assuming that A visited D multiple times) could
  solve the problem;
* removing any other node will not solve the problem.

So to finish the solution of the puzzle we need to show that if we remove A from
consideration all other people could indeed come only once to visit D. Such a
solution can be relatively quickly found manually to be e.g.:

* B comes;
* C comes;
* E comes;
* C leaves;
* F comes;
* E leaves;
* H comes;
* B leaves;
* F leaves;
* G comes;
* G leaves;
* H leaves.

# Computational solution

The solution was not long, but required some thought.
Now we solve the puzzle computationally using brute force.

Start by defining the graph we will work with:

```
using Graphs
g = SimpleGraph(7)
add_edge!(g, 1, 2)
add_edge!(g, 1, 3)
add_edge!(g, 1, 5)
add_edge!(g, 1, 6)
add_edge!(g, 2, 3)
add_edge!(g, 2, 4)
add_edge!(g, 2, 5)
add_edge!(g, 2, 7)
add_edge!(g, 3, 4)
add_edge!(g, 4, 5)
add_edge!(g, 5, 7)
add_edge!(g, 6, 7)
```

Note that I mapped people to numbers, A to 1, B to 2, C to 3, E to 4, F to 5, G to 6, H to 7.

Now, define a function that checks that a graph can be solved by each person visiting only once.

The logic of the code is as follows. If we n people we can, without loss of generality,
assume that they arrive or leave in time moments 1, 2, ..., 2n-1, 2n, where in a given moment
only one event (arrival or departure of some person) happens.
We will check all possible assignments
of people to pairs of moments (pairs, because a person comes and leaves in two distinct moments).
The process of checking is done iteratively:

* we try to add a next person (starting from the first and ending with the last);
* we check all possible paris of time spots still left unoccupied by people already assigned;
* for each pair we check if this assignment is consistent with the constraints defined by the graph.

I implemented this procedure recursively using [depth-first search][dfs]. The `find_assignment`
function performs the traversal of options. It has three arguments:

* `g` is graph of constraints (who has seen whom);
* `v` is a vector of length 2n holding information what visitors already have their
   arrival and departure time allocated (`0` indicates a free spot)
* `level` is information about the number of visitor that we are currently considering.

The `check_g_v` checks is adding a given person (with `level` number) in moments specified by
the `v` vector is consistent with information given in graph `g`.

 The code is as follows:

 ```
function check_g_v(g, v, level)
    lev_v = findall(==(level), v)
    for i in 1:level-1 # need to check only against previous visitors
        lev_i = findall(==(i), v)
        if has_edge(g, i, level)
            if lev_v[2] < lev_i[1] || lev_v[1] > lev_i[2]
                return false
            end
        else
            if !(lev_v[2] < lev_i[1] || lev_v[1] > lev_i[2])
                return false
            end
        end
    end
    return true
end

function find_assignment(g, level=1, v=zeros(Int, 2*nv(g)))
    n = nv(g)
    if level == n # we check last person
        loc = findall(==(0), v) # no choice of free spots
        @assert length(loc) == 2
        v[loc] .= n
        if check_g_v(g, v, level)
            @info "Graph is good! Solution $v"
            return true # signal success up the recursion
        end
        v[loc] .= 0 # clean up on failure
    else
        for i in 1:2*n-1 # i indicates arrival time
            v[i] == 0 || continue
            v[i] = level
            for j in i+1:2*n # j indicates departure time
                v[j] == 0 || continue
                v[j] = level
                if check_g_v(g, v, level)
                    if find_assignment(g, level+1, v)
                        return true # signal success up the recursion
                    end
                end
                v[j] = 0 # clean up on failure
            end
            v[i] = 0 # clean up on failure
        end
    end
    if level == 1 # all attempts to find good allocations failed
        @info "Graph is not good!"
    end
    return false # signal failure up the recursion
end
```

We can use it to check if the original graph is feasible:

```
julia> find_assignment(g);
[ Info: Graph is not good!
```

As we already knew - the graph is not good and the computational
procedure confirmed it.

How can we check graphs when we would remove one of the persons from it?
It is easy with Graphs.jl, as we can just subset our graph
by removing some node from it. Let us see:

```
julia> for i in 1:nv(g)
           @info "removing $i"
           find_assignment(g[[1:i-1; i+1:7]])
       end
[ Info: removing 1
[ Info: Graph is good! Solution [1, 2, 3, 2, 4, 3, 6, 1, 4, 5, 5, 6]
[ Info: removing 2
[ Info: Graph is not good!
[ Info: removing 3
[ Info: Graph is not good!
[ Info: removing 4
[ Info: Graph is not good!
[ Info: removing 5
[ Info: Graph is not good!
[ Info: removing 6
[ Info: Graph is not good!
[ Info: removing 7
[ Info: Graph is not good!
```

As we can see indeed removing node `1` (representing person A) solves the problem
and this is the only solution.

Let us do the mapping of numbers to letters to make the solution more readable.
Note that since we removed node 1 (corresponding to person A)
from the graph numbers of all nodes got decreased by 1.
This can be done by comprehension or by broadcasting (I show both approaches
for a reference):

```
julia> mapping = ["B", "C", "E", "F", "G", "H"]
6-element Vector{String}:
 "B"
 "C"
 "E"
 "F"
 "G"
 "H"

julia> [mapping[i] for i in [1, 2, 3, 2, 4, 3, 6, 1, 4, 5, 5, 6]]
12-element Vector{String}:
 "B"
 "C"
 "E"
 "C"
 "F"
 "E"
 "H"
 "B"
 "F"
 "G"
 "G"
 "H"

julia> getindex.(Ref(mapping), [1, 2, 3, 2, 4, 3, 6, 1, 4, 5, 5, 6])
12-element Vector{String}:
 "B"
 "C"
 "E"
 "C"
 "F"
 "E"
 "H"
 "B"
 "F"
 "G"
 "G"
 "H"
 ```

Indeed, this is the same order that we had in the analytical solution.

# Conclusions

I hope you find this puzzle and a solution interesting.

Next week I will post about features introduced in
a 1.6 release of [DataFrames.jl][df] that we have recently made.

[lastpost]: https://bkamins.github.io/julialang/2023/06/23/graphspuzzle.html
[dfs]: https://en.wikipedia.org/wiki/Depth-first_search
[df]: https://github.com/JuliaData/DataFrames.jl
