---
layout: post
title:  "The return of the graphs (and an interesting puzzle)"
date:   2023-06-23 05:13:13 +0200
categories: julialang
---

# Introduction

This week I come back to graphs. The reason is that I have participated
in an inspiring [Social Networks & Complex Systems Workshop][workshop].

During this workshop [Paweł Prałat][pp] asked the following puzzle:

> You are given eight batteries, out of which four are good and four
> are depleted, but visually they do not differ.
> You have a flashlight that needs two good batteries to work.
> You want your flashlight to work. What is the minimum number of times
> you need to put two batteries into your flashlight to be sure it works
> in the worst case.

In this post I want to discuss how it can be solved.
We will start with an analytical solution and then give a computational
one. You can judge for yourself which is harder.

The post was written under Julia 1.9.1,
Combinatorics v1.0.2, DataFrames v1.5.0,
SimpleGraphs v0.8.4, and SimpleGraphAlgorithms v0.4.21.

# Analytical solution

Before we begin we need to make one observation. If I say that I want to use
a given number of trials in the worst case I can provide them upfront (before
making any tests). The reason is that if I put some batteries into a
flashlight it will work (and then we are done) or not work (and we need to continue).
So the situation is simpler than in many other puzzles when you might want
adjust your strategy conditional on the answer you see to your previous queries.

What we will want to show is that we need 7 trials. Start with showing
that 7 tests is enough. To see this notice that we have 4 good batteries.
Therefore if we split our 8 batteries into three groups there will be one
group that has two batteries (by the [pigeonhole principle][pig]).
What we need to do is to show that using 7
comparisons we can find a pair of good batteries within one of these
three groups.

This is the way how you can do it.
Assume we number the batteries from 1 to 8. Split them into three groups:
`{1, 2, 3}`, `{4, 5, 6}`, and `{7, 8}`. Now we assume that we make all
possible comparisons within groups. We can see that in the first two groups
there are three comparisons possible, and in the last group only one. Seven
in total. This finishes the proof that 7 comparisons are enough.
We can visualize this solution as follows (line represents the comparison
we make):

```
  1      4    7
 / \    / \   |
2---3  5---6  8
```

We are left with showing that 6 comparisons are not enough. To see
this note the following. In the plot above we have a graph on 8 nodes
and having 7 edges. We could claim that we have a solution because in
any four element subset of its nodes there existed at least two connected
by an edge (within one group).

So to show that 6 comparisons are not enough we must show that no matter
how we make them there will always be 4 nodes that are mutually not
connected. We will prove that by contradiction. Assume we have some assignment
of edges under which maximally three nodes are not connected. Without loss
of generality take that their numbers are 1, 2, and 3. But this means that each
of the nodes 4, 5, 6, 7, and 8 must be connected to one of the 1, 2, or 3 nodes
by at least one edge. So we must use-up 5 edges to make these connections.
We are left with one edge (recall we assume that we can use 6 edges in total).
If this one last edge is connected to node 1, 2, or 3. Then nodes 4, 5, 6, 7, and 8
are not connected directly and we just found a 5-element set that is not connected by an edge.
So assume that this edge connects one pair of nodes from the set `{4, 5, 6, 7, 8}`.
However, since we only have one edge we still will have 4 nodes that are not connected.
E.g. if 4 and 5 are connected then the set of nodes `{4, 6, 7, 8}` are not connected,
so we have a 4 element set of nodes that are not connected. A contradiction to the assumption
that there were maximally three nodes that are not connected. In conclusion - 6 comparisons
are not enough.

(If you would like to see an alternative proof that uses the [probabilistic method][pm]
you can reach out to [Paweł Prałat][pp] who has shown it to me.)

# Computational solution

Now let us move to the brute force and computation (and in the process hopefully learn some Julia tricks).

First load the required packages and do some setup:

```
julia> using Combinatorics

julia> using DataFrames

julia> using SimpleGraphs

julia> using SimpleGraphAlgorithms

julia> use_Cbc()
[ Info: Solver Cbc verbose is set to false
```

As we saw in the analytical solution we can represent our queries as graphs on 8 nodes
and some edges.

The problem is that there are potentially many such graphs. Therefore we will want to limit
our search to the graphs that are not [isomorphic][ig]. Two graphs are isomorphic if you can
get one from the other by re-labelling the nodes. Clearly such two graphs are undistinguishable
for our purposes.

What we want to show is that all graphs on 8 nodes and 6 edges contain at least 4 nodes that
are not connected by an edge. And that there exist graphs on 8 nodes and 7 edges in which
the maximum number of unconnected nodes is 3.

So how do we create a list of all non-isomorphic graphs having 6 and 7 edges respectively?

Let us start with a simpler case of graphs on 8 nodes and 3 edges and list non-isomorphic graphs:

```
julia> function create_graph(es)
           g = IntGraph(8)
           for e in es
               add!(g, e...)
           end
           return g
       end
create_graph (generic function with 1 method)

julia> g3 = map(create_graph, combinations([(i, j) for i in 1:7 for j in i+1:8], 3));

julia> hg3 = uhash.(g3);

julia> g3df = DataFrame(hg=hg3, g=g3);

julia> g3gdf = groupby(g3df, :hg);

julia> redirect_stdout(devnull) do
           for sdf in g3gdf
               for i in 2:nrow(sdf)
                   @assert is_iso(sdf.g[1], sdf.g[i])
               end
           end
       end

julia> noniso3 = combine(g3gdf, first).g;

julia> elist.(noniso3)
5-element Vector{Vector{Tuple{Int64, Int64}}}:
 [(1, 2), (1, 3), (1, 4)]
 [(1, 2), (1, 3), (2, 3)]
 [(1, 2), (1, 3), (2, 4)]
 [(1, 2), (1, 3), (4, 5)]
 [(1, 2), (3, 4), (5, 6)]
```

Let us explain what we do step by step. The `g3` object contains all
graphs on three edges. Let us check how many of them we have:

```
julia> length(g3)
3276
```

There is a lot of graphs. But most of them are isomorphic. How do we prune them?
Using the `uhash` function we compute the hash of each graph.
`uhash` guarantees us that graphs having a different hash are not isomorphic.
The `g3gdf` is a `GroupedDataFrame` that keeps these graphs grouped by their
hash value. We have 5 such groups as can be checked in:

```
julia> length(g3gdf)
5
```

However, maybe this is the case that we have non-isomorphic graphs that
have the same hash value (this is unlikely but possible). We check it with
the `is_iso` function. If they would not be isomorphic `@assert` would error.
Since it does not we are good. Note that I use the `redirect_stdout(devnull)`
trick to avoid printing any output that `is_iso` produces. The reason
is that it calls CBC solver which prints to the screen its status (and since
we do over 3000 calls the screen would be flooded by not very useful output).

With the `elist.(noniso3)` we can see what are the edges of the five non-isomorphic
graphs that exist on 3 edges.
(and since we have only 5 graphs you can probably convince yourself using pen
and paper that we have found all possible options)

How do we do tis process for larger number of edges.
The same approach would work, but would be much more time consuming (there are
over 1 million graphs on 7 edges). So we use the following trick, we take the
non-isomorphic graphs with 3 edges and add one edge to them. We now get graphs
on 4 edges. Some of them are isomorphic. But we already know how to prune them
to be left only with non-isomorphic ones.

The procedure that iteratively performs this task up to 7 edges is as follows:

```
julia> function add_possible_edges(g::T) where T
           res = T[]
           for i in 1:7, j in i+1:8
               if !has(g, i, j)
                   newg = deepcopy(g)
                   add!(newg, i, j)
                   @assert NE(newg) == NE(g) + 1
                   push!(res, newg)
               end
                   end
           return res
       end
add_possible_edges (generic function with 1 method)

julia> function grow_graphs(noniso)
           g = reduce(vcat, add_possible_edges.(noniso))
           hg = uhash.(g)
           gdf = DataFrame(; hg, g)
           ggdf = groupby(gdf, :hg)
           redirect_stdout(devnull) do
               for sdf in ggdf
                   for i in 2:nrow(sdf)
                       @assert is_iso(sdf.g[1], sdf.g[i])
                   end
               end
           end
           return combine(ggdf, first).g
       end
grow_graphs (generic function with 1 method)

julia> noniso4 = grow_graphs(noniso3)
11-element Vector{UndirectedGraph{Int64}}:
 UndirectedGraph{Int64} (n=8, m=4)
 ⋮
 UndirectedGraph{Int64} (n=8, m=4)

julia> noniso5 = grow_graphs(noniso4)
24-element Vector{UndirectedGraph{Int64}}:
 UndirectedGraph{Int64} (n=8, m=5)
 ⋮
 UndirectedGraph{Int64} (n=8, m=5)

julia> noniso6 = grow_graphs(noniso5)
56-element Vector{UndirectedGraph{Int64}}:
 UndirectedGraph{Int64} (n=8, m=6)
 ⋮
 UndirectedGraph{Int64} (n=8, m=6)

julia> noniso7 = grow_graphs(noniso6)
115-element Vector{UndirectedGraph{Int64}}:
 UndirectedGraph{Int64} (n=8, m=7)
 ⋮
 UndirectedGraph{Int64} (n=8, m=7)
```

In the process we learn that there are 11 non-isomorphic graphs having 4 edges,
and respectively 24 for 5 edges, 56 for 6 edges and 115 for 7 edges.

Now for each of these graphs let us find a maximum number of nodes that are
not connected. This can be done using the `max_indep_set` function.
Again we use the `devnull` trick to avoid printing of the output:

```
julia> mis6 = redirect_stdout(devnull) do
           return max_indep_set.(noniso6)
       end
56-element Vector{Set{Int64}}:
 Set([5, 4, 6, 7, 2, 8, 3])
 ⋮
 Set([4, 7, 8, 3])

julia> minimum(length.(mis6))
4
```

So we first see that for graphs on 6 edges we indeed have at least 4 nodes in the
independent set.

Let us now check the 7 node case:

```
julia> mis7 = redirect_stdout(devnull) do
           return max_indep_set.(noniso7)
       end
115-element Vector{Set{Int64}}:
 Set([5, 4, 6, 7, 2, 8, 3])
 ⋮
 Set([7, 2, 8, 3])

julia> minimum(length.(mis7))
3

julia> elist.(noniso7[length.(mis7) .== 3])
1-element Vector{Vector{Tuple{Int64, Int64}}}:
 [(1, 2), (1, 3), (2, 3), (4, 5), (4, 6), (5, 6), (7, 8)]
```

Here we see that there exists only one graph (up to isomorphism)
that has a property that at most three nodes are independent.
And looking at its edges it is the same graph that we have drawn
in our analytical solution.

# Conclusions

So is the analytical or computational solution more interesting?
For me both have their value and were fun.

If you like such puzzles, and do plan ahead, please consider joining
us next year. From June 3 to 7, 2024 we are going to host
the [WAW2024: 19th Workshop on Modelling and Mining Networks][waw2024]
in SGH Warsaw School of Economics. We invite all enthusiasts of graphs:
both theoreticians and practitioners.

[workshop]: https://mizad.sgh.waw.pl/en/social-networks-complex-systems-workshop-warsaw-june-20-21-2023
[pp]: https://math.ryerson.ca/~pralat/
[pm]: https://en.wikipedia.org/wiki/Probabilistic_method
[ig]: https://en.wikipedia.org/wiki/Graph_isomorphism
[waw2024]: https://math.ryerson.ca/waw2024/
[pig]: https://en.wikipedia.org/wiki/Pigeonhole_principle
