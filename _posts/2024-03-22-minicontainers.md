---
layout: post
title:  "Storing vectors of vectors in DataFrames.jl"
date:   2024-03-22 06:32:12 +0200
categories: julialang
---

# Introduction

The beauty of DataFrames.jl design is that you can store any data
as columns of a data frame.
However, this leads to one tricky issue - what if we want to store
a vector as a single cell of a data frame? Today I will explain you
what is exactly the problem and how to solve it.

The post was written under Julia 1.10.1 and DataFrames.jl 1.6.1.

# Basic transformations of columns in DataFrames.jl

Let us start with a simple example:

```
julia> using DataFrames

julia> df = DataFrame(id=repeat(1:2, 5), x=1:10)
10×2 DataFrame
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     2      2
   3 │     1      3
   4 │     2      4
   5 │     1      5
   6 │     2      6
   7 │     1      7
   8 │     2      8
   9 │     1      9
  10 │     2     10
```

We want to group the `df` data frame by `"id"` and then store the `"x"` column unchanged in the result.

This can be done by writing:

```
julia> combine(groupby(df, "id", sort=true), "x")
10×2 DataFrame
 Row │ id     x
     │ Int64  Int64
─────┼──────────────
   1 │     1      1
   2 │     1      3
   3 │     1      5
   4 │     1      7
   5 │     1      9
   6 │     2      2
   7 │     2      4
   8 │     2      6
   9 │     2      8
  10 │     2     10
```

Note that the column `"x"` is expanded into multiple rows by `combine`. The rule that is applied here states that if some transformation of data returns a vector it gets expanded into multiple rows. The reason for such a behavior is that this is what we want most of the time.

However, what if we would want the vectors to be kept as they are without expanding them?
This can be achieved by writing:

```
julia> combine(groupby(df, "id", sort=true), "x" => Ref => "x")
2×2 DataFrame
 Row │ id     x
     │ Int64  SubArray…
─────┼─────────────────────────
   1 │     1  [1, 3, 5, 7, 9]
   2 │     2  [2, 4, 6, 8, 10]
```

We see that we got what we wanted, but the question is why does it work?
Let me explain.

# Containers holding one element in Julia

What we just did with `Ref` is that we wrapped some value in a container that held exactly one element.
There are three basic ways to create such a container in Julia.
The first is to wrap a vector within another vector:

```
julia> [[1,2,3]]
1-element Vector{Vector{Int64}}:
 [1, 2, 3]
```

Above you have a vector that has one element, which is a `[1, 2, 3]` vector.

The second method is to create a 0-dimensional array with `fill`:

```
julia> fill([1,2,3])
0-dimensional Array{Vector{Int64}, 0}:
[1, 2, 3]
```

The key point here is that 0-dimensional arrays are guaranteed to hold exactly one element (as opposed to a vector presented above).

The third approach is to use `Ref`:

```
julia> Ref([1,2,3])
Base.RefValue{Vector{Int64}}([1, 2, 3])
```

Wrapping an object with `Ref` also creates a 0-dimensional container. The difference between `Ref` and `fill` is that `fill` creates an array, while `Ref` is just a container (but not an array).

# How to use 1-element containers in DataFrames.jl as wrappers

All three methods described above can be used to ensure that we protect a vector from being expanded into multiple rows. Therefore the following operations give the same output:

```
julia> combine(groupby(df, "id", sort=true), "x" => (x -> [x]) => "x")
2×2 DataFrame
 Row │ id     x
     │ Int64  SubArray…
─────┼─────────────────────────
   1 │     1  [1, 3, 5, 7, 9]
   2 │     2  [2, 4, 6, 8, 10]

julia> combine(groupby(df, "id", sort=true), "x" => fill => "x")
2×2 DataFrame
 Row │ id     x
     │ Int64  SubArray…
─────┼─────────────────────────
   1 │     1  [1, 3, 5, 7, 9]
   2 │     2  [2, 4, 6, 8, 10]

julia> combine(groupby(df, "id", sort=true), "x" => Ref => "x")
2×2 DataFrame
 Row │ id     x
     │ Int64  SubArray…
─────┼─────────────────────────
   1 │     1  [1, 3, 5, 7, 9]
   2 │     2  [2, 4, 6, 8, 10]
```

The point is that `combine` unwraps the outer container (vector, 0-dimensional array, and `Ref` respectively) and stores its contents as a cell of a data frame.

Now, you might ask why initially I recommended `Ref`? The reason is that it is the method that has the smallest memory footprint:

```
julia> x = [1, 2, 3]
3-element Vector{Int64}:
 1
 2
 3

julia> @allocated [x]
64

julia> @allocated fill(x)
64

julia> @allocated Ref(x)
16
```

This difference is important if you have a huge data frame that has millions of groups.

Also writing `Ref` is simpler than writing `(x -> [x])` 😄.

# Aliasing trap

You might have noticed that in the above examples the resulting `"x"` column held `SubArrays`? Why it is the case?
To improve performance `combine` did not copy the inner vectors from the source `df` data frame, but instead made their views. This is faster and more memory efficient, but results in creating an alias between the source data frame and the result. In many cases this is not a problem.

However, in some cases you might want to avoid it. A most common case is when you later want to mutate `df` in place, but do not want the result of `combine` to reflect this change. If you want to de-alias data you need to `copy` the data in the produced columns. Therefore you should do:

```
julia> combine(groupby(df, "id", sort=true), "x" => Ref∘copy => "x")
2×2 DataFrame
 Row │ id     x
     │ Int64  Array…
─────┼─────────────────────────
   1 │     1  [1, 3, 5, 7, 9]
   2 │     2  [2, 4, 6, 8, 10]
```

Notice that now the `"x"` column stores `Array` (which indicates that the copy was made). The `Ref∘copy` expression signals function composition. We first applly the `copy` function to the source data and then pass the result to `Ref`.

# An alternative

Sometimes we want to keep the groups as columns not as rows of a data frame. In this case you can use `unstack` to achieve the desired result. Here is an example how to do it:

```
julia> unstack(df, :id, :x, combine=identity)
1×2 DataFrame
 Row │ 1                2
     │ SubArray…?       SubArray…?
─────┼───────────────────────────────────
   1 │ [1, 3, 5, 7, 9]  [2, 4, 6, 8, 10]
```

and a version copying the underlying data:

```
julia> unstack(df, :id, :x, combine=copy)
1×2 DataFrame
 Row │ 1                2
     │ Array…?          Array…?
─────┼───────────────────────────────────
   1 │ [1, 3, 5, 7, 9]  [2, 4, 6, 8, 10]
```

# Conclusions

Having read this post you should be comfortable with protecting vectors from being expanded into multiple rows when processing data frames in DataFrames.jl. Enjoy!
