---
layout: post
title:  "Transforming multiple columns in DataFrames.jl"
date:   2024-03-15 13:32:12 +0200
categories: julialang
---

# Introduction

Today I want to comment on a recurring topic that DataFrames.jl users raise.
The question is how one should transform multiple columns of a data frame using
operation specification syntax.

The post was written under Julia 1.10.1 and DataFrames.jl 1.6.1.

# What is operation specification syntax?

In DataFrames.jl the `combine`, `select`, and `transform` functions allow
users for passing the requests for data transformation using operation
specification syntax. This syntax is feature-rich, and you can find its
description for example [here][oss]. Today I want to focus on its principal concept.

In a general form each request for making an operation on data has the (E)xtract-(T)ransform-(L)oad form.
That means that we need to specify:

* source columns to get data from (the *extract* part);;
* the operation to apply to these columns (the *transform* part);
* the target columns where we want to store the result of the operation (the *load* part).

These tree parts are syntactically expressed using the following form:

```
[source columns specification] => [transformation function] => [target columns specification]
```

Let me give an example. Assume you have the following data:

```
julia> using DataFrames

julia> df = DataFrame(reshape(1:15, 5, 3), :auto)
5×3 DataFrame
 Row │ x1     x2     x3
     │ Int64  Int64  Int64
─────┼─────────────────────
   1 │     1      6     11
   2 │     2      7     12
   3 │     3      8     13
   4 │     4      9     14
   5 │     5     10     15
```

We want to compute the sum of column `"x1"` and store it in column names `"x1_sum"`
Since the `sum` function performs the addition operation the syntax specification should be:

```
"x1" => sum => "x1_sum"
```

Let us check it with the `combine` function:

```
julia> combine(df, "x1" => sum => "x1_sum")
1×1 DataFrame
 Row │ x1_sum
     │ Int64
─────┼────────
   1 │     15
```

In this syntax it is important to note two things:

* the `"x1"` column as a whole was passed to the `sum` function (as we want to compute its sum);
* the `"x1"` column is a single *positional argument* passed to the `sum` function.

Two natural questions that arise are the following:

* What if I do not want to perform an operation on a whole column, but on its elements (a.k.a. vectorization of operation)?
* What if I want to pass multiple columns as a source for computations?

We will now investigate these two dimensions.

# Vectorization of operations

Vectorization in DataFrames.jl is easy. Just wrap the function you use in the `ByRow` object. Here is an example:

```
julia> combine(df, "x1" => string => "x1_str")
1×1 DataFrame
 Row │ x1_str
     │ String
─────┼─────────────────
   1 │ [1, 2, 3, 4, 5]

julia> combine(df, "x1" => ByRow(string) => "x1_strs")
5×1 DataFrame
 Row │ x1_strs
     │ String
─────┼─────────
   1 │ 1
   2 │ 2
   3 │ 3
   4 │ 4
   5 │ 5
```

Note that `"x1" => string => "x1_str"` passed the whole `"x1"` column to the `string` function so we got a single `"[1, 2, 3, 4, 5]"`
string in the output.

While writing `"x1" => ByRow(string) => "x1_strs"` passed each element of `"x1"` column to the `string` function individually,
so in the result we got a vector of five string representations of numbers of the numbers from the source.

# Passing multiple columns

Now let us have a look at passing multiple columns. There are two ways you can do it.

The first is when your function accepts *multiple positional arguments*. An example of such function is `string` see:

```
julia> string(df.x1, df.x2)
"[1, 2, 3, 4, 5][6, 7, 8, 9, 10]"
```

If we pass a collection of columns as a source in operation specification syntax we get this behavior:

```
julia> combine(df, ["x1", "x2"] => string => "x1_x2_str")
1×1 DataFrame
 Row │ x1_x2_str
     │ String
─────┼─────────────────────────────────
   1 │ [1, 2, 3, 4, 5][6, 7, 8, 9, 10]
```

Naturally, the above combines with vectorization. Therefore since:

```
julia> string.(df.x1, df.x2)
5-element Vector{String}:
 "16"
 "27"
 "38"
 "49"
 "510"
```

we also have:

```
julia> combine(df, ["x1", "x2"] => ByRow(string) => "x1_x2_strs")
5×1 DataFrame
 Row │ x1_x2_strs
     │ String
─────┼────────────
   1 │ 16
   2 │ 27
   3 │ 38
   4 │ 49
   5 │ 510
```

However, there are cases when we have a function that expects multiple columns to be passed as a single positional argument.
This is handled in DataFrames.jl with the `AsTable` wrapper, which you can apply to the source columns.
If you use it then instead of getting multiple positional arguments the function will get a single positional argument
that will be a `NamedTuple` holding the source columns.

To convince ourselves that this is indeed what happens let us create a helper function:

```
julia> function helper(x)
           @show x
           return string(x.x1, x.x2)
       end
helper (generic function with 1 method)
```

This helper function first prints us its only argument `x` and next assumes that it has `x1` and `x2` fields and applies the `string` function to them.
Let us first check it in practice:

```
julia> helper((x1=[1, 2, 3, 4, 5], x2=[6, 7, 8, 9, 10]))
x = (x1 = [1, 2, 3, 4, 5], x2 = [6, 7, 8, 9, 10])
"[1, 2, 3, 4, 5][6, 7, 8, 9, 10]"
```

Now let us use the `helper` function with `combine`:

```
julia> combine(df, AsTable(["x1", "x2"]) => helper => "x1_x2_str")
x = (x1 = [1, 2, 3, 4, 5], x2 = [6, 7, 8, 9, 10])
1×1 DataFrame
 Row │ x1_x2_str
     │ String
─────┼─────────────────────────────────
   1 │ [1, 2, 3, 4, 5][6, 7, 8, 9, 10]
```

Indeed, we see that `helper` got a named tuple holding two columns of the source data frame.

Again, this syntax plays well with `ByRow`:

```
julia> combine(df, AsTable(["x1", "x2"]) => ByRow(helper) => "x1_x2_strs")
x = (x1 = 1, x2 = 6)
x = (x1 = 2, x2 = 7)
x = (x1 = 3, x2 = 8)
x = (x1 = 4, x2 = 9)
x = (x1 = 5, x2 = 10)
5×1 DataFrame
 Row │ x1_x2_strs
     │ String
─────┼────────────
   1 │ 16
   2 │ 27
   3 │ 38
   4 │ 49
   5 │ 510
```

We see that this time `helper` got a separate named tuple for each row of source data frame.

# Conclusions

In summary today we discussed two special operations in DataFrames.jl operation specification syntax:

* the `ByRow` which vectorizes the function passed to it;
* the `AsTable` which allows us to pass source columns as a single named tuple to the transformation function
  (instead of passing them as consecutive positional arguments, which is the default).

I hope these examples were useful in helping you understand the design of operation specification syntax.

[oss]: https://dataframes.juliadata.org/stable/man/split_apply_combine/
