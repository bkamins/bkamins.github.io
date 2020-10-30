---
layout: post
title:  "Train your brain with the Julia language"
date:   2020-10-30 19:01:02 +0200
categories: julialang
---

# Introduction

Together with [Paweł Prałat][pralat] we have writen [Train Your Brain][tyb]
book, that contains mathematical problems that are challenging (at least for us)
yet only require elementary mathematics.

In general we solve the problems using pen-and-paper only, but often we use
codes in the Julia programming language to support the reasoning. We mostly use
it to build the intuition that helps to find the proof.
In [A Julia language companion][jlc] to the book we provide a brief introduction
to the Julia language and discuss in detail the source codes we use in the main
book.

One of the problems from the book I especially like is the following:

> Every point on a circle is painted with one of three colors: red, green, and
> blue. Prove that there are three points on the circle that have the same color
> and form an isosceles triangle.

In this post I will discuss how computer support can be used in the process
of solving of this problem.

# Warm up

Let us start with a simpler problem:

> Paint every point on a circle using one of two colors red and green in such a
> way that there do not exist three points on the circle that have the same
> color and form an equilateral triangle.


To solve it pick any diameter of the circle and denote its endpoints by $$A$$
and $$B$$. Now start in point $$A$$ and moving clockwise paint the arc from
$$A$$ (inclusive) to $$B$$ (exclusive) red and arc from $$B$$ (inclusive) to
$$A$$ (exclusive) green. We will argue that this coloring meets the requirements
of the problem.

Consider points $$P$$, $$Q$$, and $$R$$ located on a circle that are all painted
with the same color. This means that the triangle $$PQR$$ formed by them does
not contain the center of the circle, and thus it is obtuse, so it is not
equilateral. This finishes the proof.

# The actual problem

Solving the actual problem is more challenging. Here, I discuss how computer
support can be used to attack it.

The first observation is that in order to use the computer we probably should
reformulate the problem. Instead of considering the circle we will consider
a regular $$n$$-gon that is inscribed in the circle. We choose such a
simplification as, intuitively, a regular $$n$$-gon contains many isosceles
triangles. Now observe that in such polygon at least $$k=\lceil n / 3 \rceil$$
points are painted using one color, by [the pigeonhole principle][php].
This means that it is enough to find such $$n$$ for which any $$k$$-element
subset of its vertices forms at least one isosceles triangle.

From this we see that the most promising values of $$n$$ should be of the form
$$3k-2$$.

Clearly picking $$k=3$$ is not enough as then $$n=7$$ and we can pick
three vertices from a $$7$$-gon that do not form an isosceles triangle.

The next guess is $$k=4$$. Then we have $$n=10$$. However, also in this case
we notice that if we pick four vertices that form a rectangle (for example
verices number 1, 2, 6, and 7 if we number vertices clockwise) they do not
form an isosceles triangle.

We thus turn to $$k=5$$ and $$n=13$$. It is not that clear what happens in this
case. Let us use the Julia language to verify this. Again number vertices
clockwise, this time from 1 to 13.

Let us start with a simpler task given three vertex numbers `a`, `b`, and `c`
how to check if they form an isosceles triangle. Here is a simple code that
verifies this (I did not try to make it super efficient, so probably I would not
get an approval for it if I wanted to merge it to [DataFrames.jl][df], but this
is my blog post so I can approve the PRs myself):

```
function isisosceles(a::Int, b::Int, c::Int)
    # sorting ensures we can calculate the distance easily
    a, b, c = sort!([a, b, c])

    # it is always safe to make sure we got what we expected
    @assert 0 < a < b < c < 14

    d1 = b - a # distance from point a to b
    d2 = c - b # distance from point b to c
    d3 = 13 - d1 - d2 # distance from point c to a
    # if any of the distances are equal as then we have an isosceles triangle
    return d1 == d2 || d2 == d3 || d3 == d1
end
```
(note in this code that the tricky case is when `d1`, `d2`, or `d3` is greater
than 6 but in this case clearly it is only possible that the triangle is
isosceles if two remaining values are equal)

Let us quickly check if our function produces the correct results on two sample
triangles:
```
julia> isisosceles(1, 2, 3)
true

julia> isisosceles(1, 2, 4)
false
```
Indeed it seems to work correctly (of course this is not the kind of tests one
should write when developing a package).

Let us move to checking all five element subsets of the set
$$\{1,2,\ldots,12,13\}$$ to make sure each of them contains at least one
isosceles triangle. Here is the code that does the job (the code was tested
under Julia 1.5.2 and Combinatorics.jl 1.0.2):
```
julia> using Combinatorics

julia> for p5 in combinations(1:13, 5)
           if !any(isisosceles(p3...) for p3 in combinations(p5, 3))
               @error "vertex set $p5 does not form any isosceles triangles"
           end
       end

julia>
```
As you can see no error message was printed, so indeed we have a support for the
claim that if one takes a regular $$13$$-gon and paints its vertices using three
colors then there will be always such three vertices that form an isosceles
triangle and have the same color.

# Concluding remarks

Before I finish let me make two remarks.

Firstly, in [the book][tyb] we show how to prove this claim without computer
support (as it is always possible that there is some bug in: Julia Base,
Combinatorics.jl, or my code).

Secondly, you might want to check out [the Julia companion][jlc] in which
I additionally discuss how to find the coloring of the regular $$13$$-gon that
generates the lowest number of isosceles triangles.

[pralat]: https://math.ryerson.ca/~pralat/
[tyb]: https://www.ryerson.ca/train-your-brain/
[jlc]: https://math.ryerson.ca/~pralat/train-your-brain.pdf
[php]: https://en.wikipedia.org/wiki/Pigeonhole_principle
[df]: https://github.com/JuliaData/DataFrames.jl
