---
layout: post
title:  "Solving Sicherman dice puzzle using DataFrames.jl"
date:   2020-06-14 15:13:43 +0200
categories: julialang
---

# Sicherman dice puzzle

In many games players roll two normal dice to get a result that is then later
used do decide the course of play.

By normal die we understand a 6-sided die with faces numbered from 1 do 6.

Now the puzzle is to check if there exists other pairs of two 6-sided dice
with faces numbered with positive integers and have the same probability
distribution for the sum as normal dice.

A standard approach to answer this question is to use generating functions to
show that it is actually possible, see e.g. [Wikipedia][wikipedia] article about
Sicherman dice.

However, in this post we will want to use DataFrames.jl to enumerate the
possible solutions and find the feasible ones. The exercise mainly showcases
the new API for the `filter` function.

The code was tested under Julia 1.4.2 and DataFrames.jl 0.21, so please make
sure you have a their proper versions (the examples should work under any
post 1.0 release of Julia, but require DataFrames.jl to be at least 0.21).

# Getting the reference probability distribution

First let us get the distribution of outcomes on a pair of normal dice.
We start with defining the `getdist` function that takes the numbers on
faces of the dies and returns their distribution:

{% highlight julia %}
function getdist(d1, d2)
    min1, max1 = extrema(d1)
    min2, max2 = extrema(d2)
    @assert min1 > 0 && min2 > 0
    d = zeros(Rational{Int}, max1 + max2)
    for p1 in d1, p2 in d2
        s = p1 + p2
        d[s] += 1
    end
    d .//= length(d1) * length(d2)
    return d
end
{% endhighlight %}

In the function we assume that sides of both dies are numbered with positive
integers. As we are working with a finite probability space all probabilities
are rationals so we store them as `Rational{Int}` type to avoid rounding. We
could have used `Float64` (or even just `Int` without doing normalization), but
as we will soon learn our code will be still fast enough so no such
approximation is required.

To test the code let us get a distribution for two normal dice and store it in
the `NORMAL_DIST` constant (we will use it later):

```
julia> const NORMAL_DIST = getdist(1:6, 1:6)
12-element Array{Rational{Int64},1}:
 0//1
 1//36
 1//18
 1//12
 1//9
 5//36
 1//6
 5//36
 1//9
 1//12
 1//18
 1//36

julia> sum(NORMAL_DIST)
1//1
```

In the second line we have checked that we actually have a probability
distribution, as all its entries add up to 1.

# Generating all possible dice

In the next step we generate all possible 6-sided die that possibly could be
used to generate the `NORMAL_DIST` distribution. In order to use some new features
of DataFrames.jl let us make one observation. Note that the sum equal to 2
is obtained with 1/36 probability. This means that each dice must have 1 on
its face exactly once.

Now what is the maximal possible value on the face? We see that the sum of two
maximal values is 12 and is obtained with 1/36 probability. This means that
the maximal value must be unique on both dice. But this implies that it must be
at most 8:
* if it were 11, then the other die would have to have only 1 on all sides
  which is not possible;
* if it were 10, then the other die would have to have one 1 and five 2s
  on its sides, which again is not allowed;
* if it were 9, then the other die would have to be `(1,2,2,2,2,3)`, but this
  would mean that the probability of rolling 11 would be at least 1/9 and it
  must be equal to 1/18.

So let us create a data frame, call it `df1`, with six columns, where each
column represents a single side of the die, and rows represent possible numbers
on its sides:

```
julia> using DataFrames

julia> df1 = DataFrame(Iterators.product((2:8 for i in 1:5)...));

julia> insertcols!(df1, 1, "0" => 1);

julia> show(df1, eltypes=false)
16807×6 DataFrame
│ Row   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │
├───────┼───┼───┼───┼───┼───┼───┤
│ 1     │ 1 │ 2 │ 2 │ 2 │ 2 │ 2 │
│ 2     │ 1 │ 3 │ 2 │ 2 │ 2 │ 2 │
│ 3     │ 1 │ 4 │ 2 │ 2 │ 2 │ 2 │
│ 4     │ 1 │ 5 │ 2 │ 2 │ 2 │ 2 │
│ 5     │ 1 │ 6 │ 2 │ 2 │ 2 │ 2 │
│ 6     │ 1 │ 7 │ 2 │ 2 │ 2 │ 2 │
│ 7     │ 1 │ 8 │ 2 │ 2 │ 2 │ 2 │
│ 8     │ 1 │ 2 │ 3 │ 2 │ 2 │ 2 │
⋮
│ 16799 │ 1 │ 7 │ 7 │ 8 │ 8 │ 8 │
│ 16800 │ 1 │ 8 │ 7 │ 8 │ 8 │ 8 │
│ 16801 │ 1 │ 2 │ 8 │ 8 │ 8 │ 8 │
│ 16802 │ 1 │ 3 │ 8 │ 8 │ 8 │ 8 │
│ 16803 │ 1 │ 4 │ 8 │ 8 │ 8 │ 8 │
│ 16804 │ 1 │ 5 │ 8 │ 8 │ 8 │ 8 │
│ 16805 │ 1 │ 6 │ 8 │ 8 │ 8 │ 8 │
│ 16806 │ 1 │ 7 │ 8 │ 8 │ 8 │ 8 │
│ 16807 │ 1 │ 8 │ 8 │ 8 │ 8 │ 8 │
```

After loading the DataFrames.jl package, we first create a `df1` data frame
with sides not equal to `1` (remember that there is exactly one such side).
For this we use the `Iterators.product` function, which gives us an iterator
that is a product of passed iterators. As the side with value `1` on it is
excluded we used five such iterators. Also note that by default `DataFrame`
constructor treats the values produced by the iterator as *rows* of the produced
data frame.

Next we use `insertcols!` function to insert a column containing only `1` in the
first position and name it `"0"` (the reason will be soon seen -- the
`DataFrame` constructor by default has named the other columns as `"1"`, `"2"`,
etc.). Note that the `column_name => value` syntax of `insertcols!` performs
automatic broadcasting of single values if needed (similar behavior is
implemented in `DataFrame` constructor, `select`, `transform`, and `combine`).

Note that we could have just written:
```
julia> DataFrame(Iterators.product(1, (2:8 for i in 1:5)...))
16807×6 DataFrame
│ Row   │ 1     │ 2     │ 3     │ 4     │ 5     │ 6     │
│       │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │
├───────┼───────┼───────┼───────┼───────┼───────┼───────┤
│ 1     │ 1     │ 2     │ 2     │ 2     │ 2     │ 2     │
│ 2     │ 1     │ 3     │ 2     │ 2     │ 2     │ 2     │
│ 3     │ 1     │ 4     │ 2     │ 2     │ 2     │ 2     │
│ 4     │ 1     │ 5     │ 2     │ 2     │ 2     │ 2     │
│ 5     │ 1     │ 6     │ 2     │ 2     │ 2     │ 2     │
│ 6     │ 1     │ 7     │ 2     │ 2     │ 2     │ 2     │
│ 7     │ 1     │ 8     │ 2     │ 2     │ 2     │ 2     │
│ 8     │ 1     │ 2     │ 3     │ 2     │ 2     │ 2     │
⋮
│ 16799 │ 1     │ 7     │ 7     │ 8     │ 8     │ 8     │
│ 16800 │ 1     │ 8     │ 7     │ 8     │ 8     │ 8     │
│ 16801 │ 1     │ 2     │ 8     │ 8     │ 8     │ 8     │
│ 16802 │ 1     │ 3     │ 8     │ 8     │ 8     │ 8     │
│ 16803 │ 1     │ 4     │ 8     │ 8     │ 8     │ 8     │
│ 16804 │ 1     │ 5     │ 8     │ 8     │ 8     │ 8     │
│ 16805 │ 1     │ 6     │ 8     │ 8     │ 8     │ 8     │
│ 16806 │ 1     │ 7     │ 8     │ 8     │ 8     │ 8     │
│ 16807 │ 1     │ 8     │ 8     │ 8     │ 8     │ 8     │
```
to get a similar result (but with different column names), but I wanted to show
the use of the `insertcols!` function.

Finally we show our data frame, but pass `eltypes=false` keyword argument to
avoid printing the column type information, as we do not need it.

We immediately notice that some rows in our `df1` data frame are duplicates. For
example row 2 and row 8 represent the same die (permutation of numbers on sides
does not affect the distribution of outcomes). We get rid of the duplicates
in-place by requiring that the numbers on sides are sorted in the `filter!`
function:
```
julia> filter!(AsTable(:) => issorted, df1)
462×6 DataFrame
│ Row │ 0     │ 1     │ 2     │ 3     │ 4     │ 5     │
│     │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │
├─────┼───────┼───────┼───────┼───────┼───────┼───────┤
│ 1   │ 1     │ 2     │ 2     │ 2     │ 2     │ 2     │
│ 2   │ 1     │ 2     │ 2     │ 2     │ 2     │ 3     │
│ 3   │ 1     │ 2     │ 2     │ 2     │ 3     │ 3     │
│ 4   │ 1     │ 2     │ 2     │ 3     │ 3     │ 3     │
│ 5   │ 1     │ 2     │ 3     │ 3     │ 3     │ 3     │
│ 6   │ 1     │ 3     │ 3     │ 3     │ 3     │ 3     │
│ 7   │ 1     │ 2     │ 2     │ 2     │ 2     │ 4     │
│ 8   │ 1     │ 2     │ 2     │ 2     │ 3     │ 4     │
⋮
│ 454 │ 1     │ 6     │ 7     │ 8     │ 8     │ 8     │
│ 455 │ 1     │ 7     │ 7     │ 8     │ 8     │ 8     │
│ 456 │ 1     │ 2     │ 8     │ 8     │ 8     │ 8     │
│ 457 │ 1     │ 3     │ 8     │ 8     │ 8     │ 8     │
│ 458 │ 1     │ 4     │ 8     │ 8     │ 8     │ 8     │
│ 459 │ 1     │ 5     │ 8     │ 8     │ 8     │ 8     │
│ 460 │ 1     │ 6     │ 8     │ 8     │ 8     │ 8     │
│ 461 │ 1     │ 7     │ 8     │ 8     │ 8     │ 8     │
│ 462 │ 1     │ 8     │ 8     │ 8     │ 8     │ 8     │
```
Notice that we have significantly reduced the number of possibilities this way
-- from 16807 to 462.

In the `filter!` call note that we used `AsTble(:) => issorted` predicate
specifier. It means that each row of a `DataFrame` is converted to a
`NamedTuple` before being passed to the `issorted` function.

# Going from one die to two dice

In `df1` we have all possible configuration of one die. Now let us generate
all possibilities for two dice:
```
julia> df2 = crossjoin(df1, df1, makeunique=true);

julia> rename!(df2, [Symbol(die, side) for die in ["l", "r"] for side in 1:6]);

julia> show(df2, eltypes=false)
213444×12 DataFrame
│ Row    │ l1 │ l2 │ l3 │ l4 │ l5 │ l6 │ r1 │ r2 │ r3 │ r4 │ r5 │ r6 │
├────────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│ 1      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │
│ 2      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 2  │ 2  │ 2  │ 2  │ 3  │
│ 3      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 2  │ 2  │ 2  │ 3  │ 3  │
│ 4      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 2  │ 2  │ 3  │ 3  │ 3  │
│ 5      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 2  │ 3  │ 3  │ 3  │ 3  │
│ 6      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 3  │ 3  │ 3  │ 3  │ 3  │
│ 7      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 2  │ 2  │ 2  │ 2  │ 4  │
│ 8      │ 1  │ 2  │ 2  │ 2  │ 2  │ 2  │ 1  │ 2  │ 2  │ 2  │ 3  │ 4  │
⋮
│ 213436 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 6  │ 7  │ 8  │ 8  │ 8  │
│ 213437 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 7  │ 7  │ 8  │ 8  │ 8  │
│ 213438 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 2  │ 8  │ 8  │ 8  │ 8  │
│ 213439 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 3  │ 8  │ 8  │ 8  │ 8  │
│ 213440 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 4  │ 8  │ 8  │ 8  │ 8  │
│ 213441 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 5  │ 8  │ 8  │ 8  │ 8  │
│ 213442 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 6  │ 8  │ 8  │ 8  │ 8  │
│ 213443 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 7  │ 8  │ 8  │ 8  │ 8  │
│ 213444 │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │ 1  │ 8  │ 8  │ 8  │ 8  │ 8  │
```
Using `crossjoin` we generate all possible combinations of both dice. We use
`makeunique=true` as we pass `df1` as the left and right data frame in cross
join. Therefore we next `rename!` the data frame that we got to properly name
its columns so that they clearly show if we are considering left or right data
frame and which side of it (note that now we number sides from 1 to 6).

# Final step -- finding Sicherman dice

So now we want to find if our `df2` data frame contains any pair of dice that
produces the same probability distribution as `NORMAL_DIST` we have computed
above. For this we define a helper function:

{% highlight julia %}
function test_dice(x...)
    d1 = ntuple(i -> x[i], 6)
    d2 = ntuple(i -> x[i + 6], 6)
    return d1 <= d2 && getdist(d1, d2) == NORMAL_DIST
end
{% endhighlight %}

The function assumes it is passed twelve positional arguments (soon these will
be values sored in a single row of our data frame). It constructs `d1` and `d2`
tuples from them, to represent the dice. First we check if `d1 <= d2` to avoid
permuted duplicates in the results, and if this test passes we check if the
probability distribution produced by our dice is the same as `NORMAL_DIST`.

Let us run the `filter` function to find the solution of the Sicherman dice
puzzle:
```
julia> show(filter(All() => test_dice, df2), eltypes=false)
2×12 DataFrame
│ Row │ l1 │ l2 │ l3 │ l4 │ l5 │ l6 │ r1 │ r2 │ r3 │ r4 │ r5 │ r6 │
├─────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│ 1   │ 1  │ 2  │ 2  │ 3  │ 3  │ 4  │ 1  │ 3  │ 4  │ 5  │ 6  │ 8  │
│ 2   │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │

```

Indeed we get that there exists only one pair of dice different from two normal
dice that meets our requirements, namely `(1,2,2,3,3,4)` and `(1,3,4,5,6,8)`.
A surprising finding indeed!

Note that this time in `filter` we have used `All() => test_dice` predicate.
It means that `test_dice` for each row of a data frame is passed all its column
as positional arguments.

# Concluding remarks

I hope you found the examples interesting and giving you some insight how
filtering in DataFrames.jl can be used efficiently.

Note that the proposed codes are not only relatively terse but quite fast:
```
julia> @time begin
           df1 = DataFrame(Iterators.product((2:8 for i in 1:5)...))
           insertcols!(df1, 1, "0" => 1)
           filter!(AsTable(:) => issorted, df1)
           df2 = crossjoin(df1, df1, makeunique=true)
           rename!(df2, [Symbol(die, side) for die in ["l", "r"] for side in 1:6])
           @time filter(All() => test_dice, df2)
       end
  0.306816 seconds (307.21 k allocations: 62.547 MiB, 2.62% gc time)
2×12 DataFrame
│ Row │ l1    │ l2    │ l3    │ l4    │ l5    │ l6    │ r1    │ r2    │ r3    │ r4    │ r5    │ r6    │
│     │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │ Int64 │
├─────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┤
│ 1   │ 1     │ 2     │ 2     │ 3     │ 3     │ 4     │ 1     │ 3     │ 4     │ 5     │ 6     │ 8     │
│ 2   │ 1     │ 2     │ 3     │ 4     │ 5     │ 6     │ 1     │ 2     │ 3     │ 4     │ 5     │ 6     │
```
As you can see we are able to solve the puzzle in sub-second time.

I have not attempted to reproduce these examples using data frames in R or
Python, as I do not feel competent enough to write exemplary codes for these
environments. However, I would be interested to see how to reproduce the steps
I have shown here and how fast they would run.

If someone would be interested to make such an implementation and its benchmark
please contact me with your proposal and I will update this post below,
giving a solution and a credit to the submitter.

# Update: example Python and R codes

Using the feedback from the readers of the post (thank you for sending it) here
are example Python and R codes that reproduce the computations.

I think that there are two aspects to compare: code readability and performance.
As for the comparison of how easy the codes are to understand I leave it for the
readers of the blog to judge for themselves. Performance can be compared more
objectively. Both Python and R codes are over 100x slower than Julia.

#### Python code

The code was proposed by [Kevin Squire][ks]. In general it follows the Julia
implementation exactly. I have minimally edited the original code I have
received to make it fit better the blog post format.

{% highlight python %}
import itertools
from fractions import Fraction
import numpy as np
import pandas as pd

def getdist(d1, d2):
    min1, max1 = min(d1), max(d1)
    min2, max2 = min(d2), max(d2)
    assert min1 > 0 and min2 > 0
    d = np.full(max1 + max2, Fraction(0, 1))
    for p1 in d1:
        for p2 in d2:
            s = p1 + p2
            d[s - 1] += 1
    d /= len(d1) * len(d2)
    return d

NORMAL_DIST = getdist(range(1, 7), range(1, 7))

def issorted(itr):
    return all(itr[i] <= itr[i + 1] for i in range(len(itr) - 1))

def cross_join(df1, df2, key="_key"):
    df1[key] = 0
    df2[key] = 0
    return df1.merge(df2, on=key, how="outer").drop(columns=key)

def test_dice(x):
    d1 = tuple(x[:6])
    d2 = tuple(x[6:])
    if d1 > d2:
        return False
    dist = getdist(d1, d2)
    if len(dist) != len(NORMAL_DIST):
        return False
    return (dist == NORMAL_DIST).all()

df1 = pd.DataFrame(itertools.product(*(range(2, 9) for _ in range(5))),
                   columns=[1, 2, 3, 4, 5])
df1.insert(0, 0, 1)
df1 = df1[df1.apply(issorted, axis=1)].reset_index(drop=True)
df2 = cross_join(df1, df1)
df2.columns = [f"{lr}{num}" for lr in ["l", "r"] for num in range(1, 7)]
df2[df2.apply(test_dice, axis=1)]
{% endhighlight %}

#### R code

The initial was proposed by [Dai ZJ][dai]. Here, I have made some more changes
in the code, but left the initial ideas of the original proposal (chiefly, I
have dropped the dependency on `data.table`, as it did not improve the speed).

In particular note that the code differs in two places from Julia/Python
implementations:

* I filter out permuted dice using temporary `ID.x` and `ID.y` columns;
* I use a different strategy for `getdist` implementation, and in particular
  do not use rational numbers but floats.

In all cases the choice was driven by the fact that I could not find a
convenient and efficient way to implement the solution to match the original
Julia codes.

{% highlight r %}
getdist <- function(d1, d2) {
  stopifnot(min(d1) > 0 && min(d2) > 0)
  prop.table(table(rowSums(expand.grid(d1, d2))))
}

NORMAL_DIST <- getdist(1:6, 1:6)

df1 <- do.call(expand.grid, c(list(1), replicate(5, list(2:8))))
df1 <- df1[!apply(df1, 1, is.unsorted), ]
df1$ID <- seq.int(nrow(df1))

df1$dummy <- 1
df2 <- merge(df1, df1, by = "dummy", allow.cartesian = TRUE)
df2$dummy <- NULL
df2 <- df2[df2$ID.x <= df2$ID.y,]
df2$ID.x <- NULL
df2$ID.y <- NULL

test_dice <- function(x) {
  d1 <- as.numeric(x[1:6])
  d2 <- as.numeric(x[7:12])
  new_dist <- getdist(d1, d2)
  if(length(new_dist) == length(NORMAL_DIST)) {
    if(all(new_dist == NORMAL_DIST)) {
      if(all(names(new_dist) == names(NORMAL_DIST))) {
        return(TRUE)
      }
    }
  }
  return(FALSE)
}

df2[apply(df2, 1, test_dice), ]
{% endhighlight %}

[wikipedia]: https://en.wikipedia.org/wiki/Sicherman_dice
[ks]: https://github.com/kmsquire
[dai]: https://github.com/xiaodaigh
