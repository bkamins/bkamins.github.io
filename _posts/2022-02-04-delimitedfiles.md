---
layout: post
title:  "Using DelimitedFiles module with DataFrames.jl"
date:   2022-02-04 07:11:23 +0200
categories: julialang
---

# Introduction

From time to time users ask how to use the `DelimitedFiles` module with
[DataFrames.jl][df] so today I have decided to write a short tutorial on this
topic.

The examples were tested under Julia 1.7.0 and DataFrames.jl 1.3.2.

# Reading CSV files in Julia

The CSV.jl package is a go-to package for reading CSV files. It provides dozens
of options allowing you to customize parsing of such files and is fast
when you read large files.

However, there are cases when you have small data stored in CSV format and might
want to avoid having the CSV.jl package as a dependency. In such situations
you can consider using the built-in `DelimitedFiles` module. In what follows
I will explain you how to use it with DataFrames.jl.

# A basic scenario:

Assume you have the following data you want to parse as CSV:
```
csv = """
a,b,c
1,2,3
4,5,6
"""
```

Normally you would read the data from disk, but here I will use the `IOBuffer`
wrapper on a string to avoid creating such file.

Let us read this data using the `DelimimitedFiles` module:

```
julia> using DelimitedFiles

julia> mat, head = readdlm(IOBuffer(csv), ',', header=true)
([1.0 2.0 3.0; 4.0 5.0 6.0], AbstractString["a" "b" "c"])

julia> mat
2×3 Matrix{Float64}:
 1.0  2.0  3.0
 4.0  5.0  6.0

julia> head
1×3 Matrix{AbstractString}:
 "a"  "b"  "c"
```

The `readdlm` function, when passed `header=true` keyword argument returned two
values. The first is `mat` matrix containing data, the second is a 1-row matrix
`head` containing column names.

If you want to ingest this data into a data frame write:

```
julia> using DataFrames

julia> DataFrame(mat, vec(head))
2×3 DataFrame
 Row │ a        b        c
     │ Float64  Float64  Float64
─────┼───────────────────────────
   1 │     1.0      2.0      3.0
   2 │     4.0      5.0      6.0
```

A crucial element of the process is `vec(head)`. The `DataFrame` constructor
expects that column names are passed as a vector, not as a matrix. Therefore
we use the `vec` function.

# Mixed element types in CSV source

Let us now consider the following CSV data:

```
csv2 = """
a,b,c
1,2,x
4,5,y
"""
```

In this case we see that the first two columns contain numbers, but the last
column consists of strings. Reading it using `readdlm` produces:

```
julia> mat2, head2 = readdlm(IOBuffer(csv2), ',', header=true)
(Any[1 2 "x"; 4 5 "y"], AbstractString["a" "b" "c"])

julia> mat2
2×3 Matrix{Any}:
 1  2  "x"
 4  5  "y"

julia> head2
1×3 Matrix{AbstractString}:
 "a"  "b"  "c"
```

Observe that now element type of `mat2` matrix is `Any`. Therefore when we
create a data frame we will get columns having `Any` element type:

```
julia> df2 = DataFrame(mat2, vec(head2))
2×3 DataFrame
 Row │ a    b    c
     │ Any  Any  Any
─────┼───────────────
   1 │ 1    2    x
   2 │ 4    5    y
```

Fortunately it is easy to narrow down the element types of columns by
broadcasting the `identity` function:

```
julia> identity.(df2)
2×3 DataFrame
 Row │ a      b      c
     │ Int64  Int64  SubStrin…
─────┼─────────────────────────
   1 │     1      2  x
   2 │     4      5  y
```

# Conclusions

What is the benefit of using `DelimitedFiles` module over CSV.jl? The first that
it is shipped with Base Julia so it does not require installation. The second
is that for small files it will be faster on the first run as compilation
of functions from the CSV.jl package takes several seconds.

What are the drawbacks? For large data the `readdlm` function will be slower.
Additionally it lacks many options. `DelimitedFiles` is best suited for reading
data having homogeneous type. If columns have mixed types it becomes less
convenient. Similarly, e.g. when you have missing data in the CSV file you
would have to manually identify them after reading it in with
the `readdlm` function.

[df]: https://github.com/JuliaData/DataFrames.jl
