---
layout: post
title:  "An exercise in DataFrames.jl transformation minilanguage"
date:   2022-05-06 06:01:02 +0200
categories: julialang
---

# Introduction

Recently I answered an interesting question about transformation of a data
frame. I thought that the problem and solution are instructive enough to
warrant writing a blog post about them.

This post was written under Julia 1.7.0 and DataFrames.jl 1.3.4.

# The problem

Assume you are given the following data frame:

```
julia> using DataFrames

julia> df1 = DataFrame(reshape(1:30, 5, 6), vec(string.(["x", "y"], [1 2 3])))
5×6 DataFrame
 Row │ x1     y1     x2     y2     x3     y3
     │ Int64  Int64  Int64  Int64  Int64  Int64
─────┼──────────────────────────────────────────
   1 │     1      6     11     16     21     26
   2 │     2      7     12     17     22     27
   3 │     3      8     13     18     23     28
   4 │     4      9     14     19     24     29
   5 │     5     10     15     20     25     30
```

Before we move forward let me comment a bit about this code.
The `reshape(1:30, 5, 6)` part creates a 5x6 matrix filled with integers
ranging from 1 to 30:

```
julia> reshape(1:30, 5, 6)
5×6 reshape(::UnitRange{Int64}, 5, 6) with eltype Int64:
 1   6  11  16  21  26
 2   7  12  17  22  27
 3   8  13  18  23  28
 4   9  14  19  24  29
 5  10  15  20  25  30
```

Next the `string.(["x", "y"], [1 2 3])` part creates a matrix of column names:

```
julia> string.(["x", "y"], [1 2 3])
2×3 Matrix{String}:
 "x1"  "x2"  "x3"
 "y1"  "y2"  "y3"
```

I use `vec` on it since the `DataFrame` constructor requires column names to
be passed as a vector.

We want to create a new data frame having the following four columns:
* `x_minimum`: storing for each row minimum value stored in
               the columns containing `"x"` in their name;
* `x_maximum`: storing for each row maximum value stored in
               the columns containing `"x"` in their name;
* `y_minimum`: storing for each row minimum value stored in
               the columns containing `"y"` in their name;
* `y_maximum`: storing for each row maximum value stored in
               the columns containing `"y"` in their name.

The question is how to do it in DataFrames.jl. Below I will discuss several
options you can consider.

# Using a loop

Here is a simple approach for performing this operation which relies
on knowledge of Base Julia:

```
julia> df2 = DataFrame()
0×0 DataFrame

julia> for n in ["x", "y"]
           mat = Matrix(df1[:, Regex(n)])
           for fun in [minimum, maximum]
               df2[:, string(n, "_", fun)] = fun.(eachrow(mat))
           end
       end

julia> df2
5×4 DataFrame
 Row │ x_minimum  x_maximum  y_minimum  y_maximum
     │ Int64      Int64      Int64      Int64
─────┼────────────────────────────────────────────
   1 │         1         21          6         26
   2 │         2         22          7         27
   3 │         3         23          8         28
   4 │         4         24          9         29
   5 │         5         25         10         30
```

What we do in this code can be explained as follows. First we create an empty
target data frame `df2`. Next we iteratively add columns to it. To be able to
use Base Julia functionality we select from the data frame the columns
respectively having `"x"` or `"y"` in their name using a regular expression and
convert the result to a `Matrix`. Finally we apply the `minimum` or `maximum`
function to rows of this matrix with the `fun.(eachrow(mat))` expression and
assign the result to a new column in the `df2` data frame.

# Using broadcasting in transformation minilanguage

Now let us turn to using the DataFrames.jl transformation minilanguage
(if you do not have much experience with it I recommend you to first read
[this post][mini] before proceeding):

```
julia> df2 = select(df1, AsTable.([r"x" r"y"]) .=>
                         ByRow.([minimum, maximum]) .=>
                         string.(["x_" "y_"], [minimum, maximum]))
5×4 DataFrame
 Row │ x_minimum  x_maximum  y_minimum  y_maximum
     │ Int64      Int64      Int64      Int64
─────┼────────────────────────────────────────────
   1 │         1         21          6         26
   2 │         2         22          7         27
   3 │         3         23          8         28
   4 │         4         24          9         29
   5 │         5         25         10         30
```

To understand what is going on in this expression we first need to inspect
what is passed to `select` as a second argument:

```
julia> AsTable.([r"x" r"y"]) .=>
       ByRow.([minimum, maximum]) .=>
       string.(["x_" "y_"], [minimum, maximum])
2×2 Matrix{Pair{AsTable}}:
 AsTable(r"x")=>(ByRow{typeof(minimum)}(minimum)=>"x_minimum")  AsTable(r"y")=>(ByRow{typeof(minimum)}(minimum)=>"y_minimum")
 AsTable(r"x")=>(ByRow{typeof(maximum)}(maximum)=>"x_maximum")  AsTable(r"y")=>(ByRow{typeof(maximum)}(maximum)=>"y_maximum")
```

As you can see Julia broadcasting mechanism *magically* created four operation
specification expressions. The trick is what since we wanted function names to
change faster I passed them in a vector `[minimum, maximum]` twice, while I
wanted column names to change slower, so I passed them to broadcasting as one
row matrices with `[r"x" r"y"]` and `["x_" "y_"]` expressions respectively.

The second part is understanding what each of the operation specification
expression means. Let us concentrate on the first one:

```
AsTable(r"x")=>(ByRow{typeof(minimum)}(minimum)=>"x_minimum")
```

The decomposition is:
* `AsTable(r"x")` means: select all columns that contain `"x"` and pass them
  to the transformation function as a single positional argument (this is what
  `AsTable` serves for here);
* the `ByRow(minimum)` part means that we want to apply the `minimum` function
  to each row of the data passed to it;
* finally `"x_minimum"` part means that we want to store the result in the column
  having this name.

An alternative way to write this transformation would be to replace `minimum`
and `maximum` with `min` and `max`. The difference is that `min` and `max`
take multiple positional arguments. It means that we would need to drop the
`AsTable` part in the transformation specification like this:

```
julia> df2 = select(df1, [r"x" r"y"] .=>
                         ByRow.([min, max]) .=>
                         string.(["x_" "y_"], [minimum, maximum]))
5×4 DataFrame
 Row │ x_minimum  x_maximum  y_minimum  y_maximum
     │ Int64      Int64      Int64      Int64
─────┼────────────────────────────────────────────
   1 │         1         21          6         26
   2 │         2         22          7         27
   3 │         3         23          8         28
   4 │         4         24          9         29
   5 │         5         25         10         30

```

As you can see we get the same result. You might ask why I have not used this
style initially? The reason is that `r"x"` potentially could have selected
thousands of columns from the source data frame. In Julia, in general, it is
not a good idea to pass very many positional arguments to functions as in some
cases it might put too much strain on the Julia compiler. In such cases
`AsTable` wrapper is preferred as it guarantees to pass a single argument to
the function(a collection of passed columns).


# Using a comprehension in transformation minilanguage

Above I have shown you how to use broadcasting to achieve the desired result.
Let me show below that it is equally easy to use a comprehension to achieve
the same:

```
julia> df2 = select(df1, [AsTable(Regex(n)) => ByRow(fun) => string(n, "_", fun)
                    for n in ["x", "y"] for fun in [minimum, maximum]])
5×4 DataFrame
 Row │ x_minimum  x_maximum  y_minimum  y_maximum
     │ Int64      Int64      Int64      Int64
─────┼────────────────────────────────────────────
   1 │         1         21          6         26
   2 │         2         22          7         27
   3 │         3         23          8         28
   4 │         4         24          9         29
   5 │         5         25         10         30
```

The choice between using broadcasting and a comprehension is mostly a personal
preference.

# Conclusions

I hope you will find the presented examples useful to better understand how to
write complex transformations in DataFrames.jl.

The codes might look scary to you at a first glance. However, in my experience,
after having some practice with broadcasting or writing comprehensions in
Julia they become natural.

[mini]: https://bkamins.github.io/julialang/2020/12/24/minilanguage.html
