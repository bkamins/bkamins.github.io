---
layout: post
title:  "Hungarian meeting with Euler for the third anniversary"
date:   2023-05-05 06:43:13 +0200
categories: julialang
---

# Introduction

I have been writing this blog for three years now, so I was
thinking what to post about to celebrate this.

Recently I have learned about the [ProjectEuler.jl][pe] package.
I like it very much. It gives access to problems presented in
the [Project Euler][pewww] website in Julia REPL.
Additionally, when reading the documentation of the package it
mentioned a problem that I have not seen before. Therefore
I thought to solve it in this post.

This post was written under Julia 1.9.0-rc2, HiGHS v1.5.1,
Hungarian v0.7.0, JuMP v1.10.0, ProjectEuler v0.1.1.

# The puzzle

Let us use ProjectEuler.jl to get the description of the
problem we want to solve:

```
julia> import ProjectEuler

julia> ProjectEuler.question(345)

│             Source: The following problem is taken from Project Euler
│      Problem Title: Problem 345: Matrix Sum
│       Published On: Saturday, 3rd September 2011, 04:00 pm
│          Solved By: 5813
│  Difficulty Rating: 15%

Problem
≡≡≡≡≡≡≡≡≡≡
We define the Matrix Sum of a matrix as the maximum possible sum of matrix
elements such that none of the selected elements share the same row or column.

For example, the Matrix Sum of the matrix below equals
3315 ( = 863 + 383 + 343 + 959 + 767):

                                                   7  53 183 439 863
                                                 497 383 563  79 973
                                                 287  63 343 169 583
                                                 627 343 773 959 943
                                                 767 473 103 699 303

Find the Matrix Sum of:

                         7  53 183 439 863 497 383 563  79 973 287  63 343 169 583
                       627 343 773 959 943 767 473 103 699 303 957 703 583 639 913
                       447 283 463  29  23 487 463 993 119 883 327 493 423 159 743
                       217 623   3 399 853 407 103 983  89 463 290 516 212 462 350
                       960 376 682 962 300 780 486 502 912 800 250 346 172 812 350
                       870 456 192 162 593 473 915  45 989 873 823 965 425 329 803
                       973 965 905 919 133 673 665 235 509 613 673 815 165 992 326
                       322 148 972 962 286 255 941 541 265 323 925 281 601  95 973
                       445 721  11 525 473  65 511 164 138 672  18 428 154 448 848
                       414 456 310 312 798 104 566 520 302 248 694 976 430 392 198
                       184 829 373 181 631 101 969 613 840 740 778 458 284 760 390
                       821 461 843 513  17 901 711 993 293 157 274  94 192 156 574
                        34 124   4 878 450 476 712 914 838 669 875 299 823 329 699
                       815 559 813 459 522 788 168 586 966 232 308 833 251 631 107
                       813 883 451 509 615  77 281 613 459 205 380 274 302  35 805
```

# Manual solution

To start let us define the matrix that gives specification of the problem
and bind it to the `M` variable:

```
M = [  7  53 183 439 863 497 383 563  79 973 287  63 343 169 583
     627 343 773 959 943 767 473 103 699 303 957 703 583 639 913
     447 283 463  29  23 487 463 993 119 883 327 493 423 159 743
     217 623   3 399 853 407 103 983  89 463 290 516 212 462 350
     960 376 682 962 300 780 486 502 912 800 250 346 172 812 350
     870 456 192 162 593 473 915  45 989 873 823 965 425 329 803
     973 965 905 919 133 673 665 235 509 613 673 815 165 992 326
     322 148 972 962 286 255 941 541 265 323 925 281 601  95 973
     445 721  11 525 473  65 511 164 138 672  18 428 154 448 848
     414 456 310 312 798 104 566 520 302 248 694 976 430 392 198
     184 829 373 181 631 101 969 613 840 740 778 458 284 760 390
     821 461 843 513  17 901 711 993 293 157 274  94 192 156 574
      34 124   4 878 450 476 712 914 838 669 875 299 823 329 699
     815 559 813 459 522 788 168 586 966 232 308 833 251 631 107
     813 883 451 509 615  77 281 613 459 205 380 274 302  35 805]
```

Note that it is easy to do in Julia REPL, by copy-pasting the text
from the problem specification and just wrapping it with `M = [` and `]`.

To solve the problem manually let us make the following observations:

* Since every column has to be picked exactly once subtracting
  the same value from each element of some column does not affect the
  solution (the same holds for rows).
* If in every row maximal element is in a different column then we can
  just pick these maximal elements in each row and these entries
  are the solution to our problem.

Using these two facts we will try to solve our problem. First let us
check if in our initial matrix `M` each row has a unique column where
it has a maximum value:

```
julia> findall(==(0), M .- maximum(M, dims=2))
15-element Vector{CartesianIndex{2}}:
 CartesianIndex(15, 2)
 CartesianIndex(2, 4)
 CartesianIndex(5, 4)
 CartesianIndex(11, 7)
 CartesianIndex(3, 8)
 CartesianIndex(4, 8)
 CartesianIndex(12, 8)
 CartesianIndex(13, 8)
 CartesianIndex(6, 9)
 CartesianIndex(14, 9)
 CartesianIndex(1, 10)
 CartesianIndex(10, 12)
 CartesianIndex(7, 14)
 CartesianIndex(8, 15)
 CartesianIndex(9, 15)
```

Unfortunately, this is not the case. We see that e.g. rows 2 and 5 have
maximum in column 4. Therefore we cannot trivially solve our problem.

However, let us try subtracting some values from columns of our
matrix `M` hoping that we will get the desired uniqueness.

The values we subtract from each column are:

```
julia> sub = [55 0 23 56 40 0 101 171 175 62 53 151 0 0 26]
1×15 Matrix{Int64}:
 55  0  23  56  40  0  101  171  175  62  53  151  0  0  26
```

Let us check them:

```
julia> M2 = M .- sub
15×15 Matrix{Int64}:
 -48   53  160  383  823  497  282   392  -96  911  234  -88  343  169  557
 572  343  750  903  903  767  372   -68  524  241  904  552  583  639  887
 392  283  440  -27  -17  487  362   822  -56  821  274  342  423  159  717
 162  623  -20  343  813  407    2   812  -86  401  237  365  212  462  324
 905  376  659  906  260  780  385   331  737  738  197  195  172  812  324
 815  456  169  106  553  473  814  -126  814  811  770  814  425  329  777
 918  965  882  863   93  673  564    64  334  551  620  664  165  992  300
 267  148  949  906  246  255  840   370   90  261  872  130  601   95  947
 390  721  -12  469  433   65  410    -7  -37  610  -35  277  154  448  822
 359  456  287  256  758  104  465   349  127  186  641  825  430  392  172
 129  829  350  125  591  101  868   442  665  678  725  307  284  760  364
 766  461  820  457  -23  901  610   822  118   95  221  -57  192  156  548
 -21  124  -19  822  410  476  611   743  663  607  822  148  823  329  673
 760  559  790  403  482  788   67   415  791  170  255  682  251  631   81
 758  883  428  453  575   77  180   442  284  143  327  123  302   35  779

julia> sol = findall(==(0), M2 .- maximum(M2, dims=2))
15-element Vector{CartesianIndex{2}}:
 CartesianIndex(6, 1)
 CartesianIndex(15, 2)
 CartesianIndex(8, 3)
 CartesianIndex(5, 4)
 CartesianIndex(4, 5)
 CartesianIndex(12, 6)
 CartesianIndex(11, 7)
 CartesianIndex(3, 8)
 CartesianIndex(14, 9)
 CartesianIndex(1, 10)
 CartesianIndex(2, 11)
 CartesianIndex(10, 12)
 CartesianIndex(13, 13)
 CartesianIndex(7, 14)
 CartesianIndex(9, 15)
```

Now we see that we have exactly one maximum value in each row and each of these values
is in a different column. Thus the solution to the problem can be calculated as
(I do not show the solution to encourage you to try solving the problem yourself):

```
sum(M[sol])
```

Now you might ask how one could get the `sub` vector?
You could find it by trial and error, or use a more systematic way.
Interestingly the problem we solve today is an important question
in operations research, and a specialized algorithm was developed to solve it.

# Hungarian solution

The algorithm that can be used to solve this class of problems is called
[Hungarian algorithm][ha]. It is implemented in Julia in the [Hungarian.jl][hap]
package. I encourage you to study it, however, let me just mention that it uses
a refined version of the two observations we have made when developing the manual
solution.

The package is easy to use. You just need to remember that by default it minimizes
the sum, so we need to use the `-M` matrix. Here is how you can get
the solution (I show you the indices, but drop displaying the value of the solution):

```
julia> using Hungarian

julia> hungarian(-M)[1]
15-element Vector{Int64}:
 10
 11
  8
  5
  4
  1
 14
  3
 15
 12
  7
  6
 13
  9
  2
```

You might ask how we could check if our manual solution and the solution obtained
using the package match. You can do it as follows:

```
julia> getindex.(sol, 1) == hungarian(-M')[1]
true
```

All matches as expected.

Note that for the check I used the `hungarian` function with the transposition
of the `M` matrix as our cartesian indices are sorted by column number
(the reason is that Julia uses column major storage order) and by default
`hungarian` returns column indices sorted by row number.

# Solver solution

What if we did not have the Hungarian.jl package?
In this case the problem can be solved using mixed integer programming.
I have decided to use the JuMP.jl and HiGHS.jl packages to get the answer
(as usual - the value of the solution is not shown):

```
using JuMP
import HiGHS
model = Model(HiGHS.Optimizer)
@variable(model, x[1:15, 1:15], Bin)
for i in 1:15
    @constraint(model, sum(x[i, j] for j in 1:15) == 1)
    @constraint(model, sum(x[j, i] for j in 1:15) == 1)
end
@objective(model, Max, sum(x[i, j] * M[i, j] for i in 1:15, j in 1:15))
optimize!(model)
value.(x)
```

I really enjoy using the JuMP.jl API for solving optimization problems.

As above let us check if the solution matches the solution we obtained manually:

```
julia> findall(>=(0.5), value.(x)) == sol
true
```

Note that I use `0.5` to separate `0` from `1` solutions as this is a
safe boundary value even if there were some numerical inaccuracies in the
returned solution.

# Conclusions

I really enjoy solving Project Euler puzzles using Julia.
The syntax and package ecosystem that I have at hand make it
quite convenient. The resulting codes are usually short and easy to read.

If you liked the problem let me give you a challenge. Notice that
the sum of values we subtracted in the manual solution was:

```
julia> sum(sub)
913
```

The challenge for you is to find such non-negative entries of `sub`
that uniquely solve our problem (in the manual approach) and
minimize the sum of entries of `sub`. I hope you will enjoy solving
this extra puzzle!

[pe]: https://github.com/udohjeremiah/ProjectEuler.jl
[pewww]: https://projecteuler.net/
[ha]: https://en.wikipedia.org/wiki/Hungarian_algorithm
[hap]: https://github.com/Gnimuc/Hungarian.jl
