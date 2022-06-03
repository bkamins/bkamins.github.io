---
layout: post
title:  "DataFrames.jl for work and pleasure"
date:   2022-06-03 10:24:14 +0200
categories: julialang
---

# Introduction

This week I have read a post on [Why Vim is better than VSCode][vim].
In it the author discusses a lot the *operator - text object - motion*
pattern in Vim. The post argues that it is not only efficient but fun to
learn and use.

It reminded me of the structure of the operation specification language
we have in DataFrames.jl that follows the pattern:
```
input columns => transformation => output column names
```

I have already written two posts about this topic that you can find
[here][p1] and [here][p2]. Therefore, today I decided to take the
*fun part* of using the minilanguage.

The post is written under Julia 1.7.2 and DataFrames.jl 1.3.4.

# The challenge

The user has some data frame and wants to drop a `:col` column from it,
but the user is not sure if this column is present in the data frame.

Let us first create two test data frames on which we will test our solutions:

```
julia> using DataFrames

julia> df1 = DataFrame(a=1:2, b=3:4)
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4

julia> df2 = DataFrame(a=1:2, col=["drop", "me"], b=3:4)
2×3 DataFrame
 Row │ a      col     b
     │ Int64  String  Int64
─────┼──────────────────────
   1 │     1  drop        3
   2 │     2  me          4
```

# A basic approach

A natural thing to try is using the `Not` selector for this task. Let us
check it:

```
julia> select(df1, Not(:col))
ERROR: ArgumentError: column name :col not found in the data frame

julia> select(df2, Not(:col))
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4
```

The operation worked on `df2`, but failed on `df1`.

You might ask why `Not` selector is so restrictive? The reason is to avoid bugs.
You could accidentally mistype column name and then, if such operation worked,
instead of erroring, your incorrect result would propagate.

# An intermediate solution

A first solution that comes to mind is to drop the column only if it is present
in a data frame so you might write something like this:

```
julia> select(df1, names(df1) .!= "col")
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4

julia> select(df2, names(df2) .!= "col")
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4
```

This works, but you need to write the name of the source data frame twice,
so the solution feels a bit heavy.

# The fun part

What is the way I find nice to do this operation then? Here is the approach:

```
julia> select(df1, Cols(!=("col")))
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4

julia> select(df2, Cols(!=("col")))
2×2 DataFrame
 Row │ a      b
     │ Int64  Int64
─────┼──────────────
   1 │     1      3
   2 │     2      4
```

We are using a combo of a bit advanced features here.

First `!=("col")` creates a function that compares its argument to `"col"` using
`!=`. This is a very nice feature of Base Julia that it allows partial function
application for the `!=` operator.

Next the `Cols` function accepts a predicate, in our case `!=("col")`. Then it
selects all columns of a data frame for which this predicate returns `true`.

# Conclusions

The beauty of Julia is that it not only does the job you want done, but also is
quite fun to code with. At the same time, its design often helps you with
catching common possible bugs in code (like the `Not` behavior I have described
in this post).

Enjoy!

[vim]: https://sean-warman.medium.com/why-vim-is-better-than-vscode-d09e2355eb37
[p1]: https://bkamins.github.io/julialang/2022/05/06/minilanguage.html
[p2]: https://bkamins.github.io/julialang/2020/12/24/minilanguage.html
