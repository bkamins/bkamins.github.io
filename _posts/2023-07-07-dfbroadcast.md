---
layout: post
title:  "Tips and tricks of broadcasting in DataFrames.jl"
date:   2023-07-07 05:14:43 +0200
categories: julialang
---

# Introduction

Broadcasting, a.k.a. the `.` operator, is a powerful feature of Julia that allows you to
easily vectorize any function. Today I want to write about some common broadcasting
patterns that can be used in DataFrames.jl.

The post was written under Julia 1.9.2 and DataFrames.jl 1.5.0.

# Conditional replacement of values in a data frame

Let us create a data frame with some random contents:

```
julia> using DataFrames

julia> using Random

julia> df = DataFrame(rand(5, 6), :auto)
5×6 DataFrame
 Row │ x1         x2         x3         x4        x5        x6
     │ Float64    Float64    Float64    Float64   Float64   Float64
─────┼─────────────────────────────────────────────────────────────────
   1 │ 0.364225   0.690894   0.240867   0.720774  0.331573  0.549766
   2 │ 0.225226   0.241412   0.0793279  0.418206  0.775367  0.35275
   3 │ 0.19913    0.0633375  0.767805   0.280096  0.721995  0.917259
   4 │ 0.708132   0.230088   0.702677   0.947402  0.928979  0.66101
   5 │ 0.0267573  0.0122425  0.549734   0.331788  0.32658   0.00476749
```

Assume we want to get a new data frame that has `true` when the value stored
in the cell is greater than `0.5` and `false` otherwise. This is easy.
We just broadcast the `>` operator:

```
julia> df .> 0.5
5×6 DataFrame
 Row │ x1     x2     x3     x4     x5     x6
     │ Bool   Bool   Bool   Bool   Bool   Bool
─────┼──────────────────────────────────────────
   1 │ false   true  false   true  false   true
   2 │ false  false  false  false   true  false
   3 │ false  false   true  false   true   true
   4 │  true  false   true   true   true   true
   5 │ false  false   true  false  false  false
```

Now assume we want to replace all values greater than `0.5` with `0.5` and
keep the lower values untouched. This can be done with `ifelse`:

```
julia> ifelse.(df .> 0.5, 0.5, df)
5×6 DataFrame
 Row │ x1         x2         x3         x4        x5        x6
     │ Float64    Float64    Float64    Float64   Float64   Float64
─────┼─────────────────────────────────────────────────────────────────
   1 │ 0.364225   0.5        0.240867   0.5       0.331573  0.5
   2 │ 0.225226   0.241412   0.0793279  0.418206  0.5       0.35275
   3 │ 0.19913    0.0633375  0.5        0.280096  0.5       0.5
   4 │ 0.5        0.230088   0.5        0.5       0.5       0.5
   5 │ 0.0267573  0.0122425  0.5        0.331788  0.32658   0.00476749
```

Or with `clamp`:

```
julia> clamp.(df, -Inf, 0.5)
5×6 DataFrame
 Row │ x1         x2         x3         x4        x5        x6
     │ Float64    Float64    Float64    Float64   Float64   Float64
─────┼─────────────────────────────────────────────────────────────────
   1 │ 0.364225   0.5        0.240867   0.5       0.331573  0.5
   2 │ 0.225226   0.241412   0.0793279  0.418206  0.5       0.35275
   3 │ 0.19913    0.0633375  0.5        0.280096  0.5       0.5
   4 │ 0.5        0.230088   0.5        0.5       0.5       0.5
   5 │ 0.0267573  0.0122425  0.5        0.331788  0.32658   0.00476749
```

Similarly we could clamp values to the `[0.1, 0.9]` interval:

```
julia> clamp.(df, 0.1, 0.9)
5×6 DataFrame
 Row │ x1        x2        x3        x4        x5        x6
     │ Float64   Float64   Float64   Float64   Float64   Float64
─────┼────────────────────────────────────────────────────────────
   1 │ 0.364225  0.690894  0.240867  0.720774  0.331573  0.549766
   2 │ 0.225226  0.241412  0.1       0.418206  0.775367  0.35275
   3 │ 0.19913   0.1       0.767805  0.280096  0.721995  0.9
   4 │ 0.708132  0.230088  0.702677  0.9       0.9       0.66101
   5 │ 0.1       0.1       0.549734  0.331788  0.32658   0.1
```

Importantly, we do not need to keep the element type of the source column fixed.
Assume that we want to set values greater than `0.5` to `missing`:

```
julia> ifelse.(df .> 0.5, missing, df)
5×6 DataFrame
 Row │ x1               x2               x3               x4              x5              x6
     │ Float64?         Float64?         Float64?         Float64?        Float64?        Float64?
─────┼─────────────────────────────────────────────────────────────────────────────────────────────────────
   1 │       0.364225   missing                0.240867   missing               0.331573  missing
   2 │       0.225226         0.241412         0.0793279        0.418206  missing               0.35275
   3 │       0.19913          0.0633375  missing                0.280096  missing         missing
   4 │ missing                0.230088   missing          missing         missing         missing
   5 │       0.0267573        0.0122425  missing                0.331788        0.32658         0.00476749
```

Note that the operation performed an automatic promotion of column element types.

As a final operation consider taking `sign(log(x) + 1)` on our data frame:

```
julia> sign.(log.(df) .+ 1)
5×6 DataFrame
 Row │ x1       x2       x3       x4       x5       x6
     │ Float64  Float64  Float64  Float64  Float64  Float64
─────┼──────────────────────────────────────────────────────
   1 │    -1.0      1.0     -1.0      1.0     -1.0      1.0
   2 │    -1.0     -1.0     -1.0      1.0      1.0     -1.0
   3 │    -1.0     -1.0      1.0     -1.0      1.0      1.0
   4 │     1.0     -1.0      1.0      1.0      1.0      1.0
   5 │    -1.0     -1.0      1.0     -1.0     -1.0     -1.0
```

Again - things are easy and intuitive. Data frame behaves just like a matrix in all operations.

I hope now you are comfortable with creation of a new data frame using broadcasting.

We can turn to in-place operations on a data frame.

# In-place update of values in a data frame

In general it is enough to just put data frame on a right hand side of a broadcasted assignment
operator to update it in-place:

```
julia> df2 = copy(df)
5×6 DataFrame
 Row │ x1         x2         x3         x4        x5        x6
     │ Float64    Float64    Float64    Float64   Float64   Float64
─────┼─────────────────────────────────────────────────────────────────
   1 │ 0.364225   0.690894   0.240867   0.720774  0.331573  0.549766
   2 │ 0.225226   0.241412   0.0793279  0.418206  0.775367  0.35275
   3 │ 0.19913    0.0633375  0.767805   0.280096  0.721995  0.917259
   4 │ 0.708132   0.230088   0.702677   0.947402  0.928979  0.66101
   5 │ 0.0267573  0.0122425  0.549734   0.331788  0.32658   0.00476749

julia> df3 = df2
5×6 DataFrame
 Row │ x1         x2         x3         x4        x5        x6
     │ Float64    Float64    Float64    Float64   Float64   Float64
─────┼─────────────────────────────────────────────────────────────────
   1 │ 0.364225   0.690894   0.240867   0.720774  0.331573  0.549766
   2 │ 0.225226   0.241412   0.0793279  0.418206  0.775367  0.35275
   3 │ 0.19913    0.0633375  0.767805   0.280096  0.721995  0.917259
   4 │ 0.708132   0.230088   0.702677   0.947402  0.928979  0.66101
   5 │ 0.0267573  0.0122425  0.549734   0.331788  0.32658   0.00476749

julia> df2 .= log.(df)
5×6 DataFrame
 Row │ x1         x2         x3         x4          x5          x6
     │ Float64    Float64    Float64    Float64     Float64     Float64
─────┼─────────────────────────────────────────────────────────────────────
   1 │ -1.00998   -0.369769  -1.42351   -0.32743    -1.10391    -0.598262
   2 │ -1.49065   -1.42125   -2.53417   -0.87178    -0.254419   -1.04199
   3 │ -1.6138    -2.75928   -0.26422   -1.27262    -0.325737   -0.0863653
   4 │ -0.345124  -1.46929   -0.352858  -0.0540319  -0.0736689  -0.413987
   5 │ -3.62095   -4.40285   -0.598321  -1.10326    -1.11908    -5.34594

julia> df2 === df3
true
```

Note that with the last check I made sure that indeed the `df2 .= log.(df)` was in-place.
We updated the contents of `df2`, and not created a new object.

However, sometimes things are more tricky. Consider the `df .> 0.5` operation we did above:

```
julia> df2 .= df .> 0.5
5×6 DataFrame
 Row │ x1       x2       x3       x4       x5       x6
     │ Float64  Float64  Float64  Float64  Float64  Float64
─────┼──────────────────────────────────────────────────────
   1 │     0.0      1.0      0.0      1.0      0.0      1.0
   2 │     0.0      0.0      0.0      0.0      1.0      0.0
   3 │     0.0      0.0      1.0      0.0      1.0      1.0
   4 │     1.0      0.0      1.0      1.0      1.0      1.0
   5 │     0.0      0.0      1.0      0.0      0.0      0.0
```

Note that there is a difference from creating a new data frame with `df .> 0.5`.
The issue is that columns of `df2` keep their original types. This is expected, as we
wanted a fully in-place operation. However, sometimes you might want to change the
element type of a column when doing broadcasting. This is possible, however,
then you need to use data frame indexing with a special `!` row selector which signals
that column replacement is requested:

```
julia> df2[!, :] .= df .> 0.5
5×6 DataFrame
 Row │ x1     x2     x3     x4     x5     x6
     │ Bool   Bool   Bool   Bool   Bool   Bool
─────┼──────────────────────────────────────────
   1 │ false   true  false   true  false   true
   2 │ false  false  false  false   true  false
   3 │ false  false   true  false   true   true
   4 │  true  false   true   true   true   true
   5 │ false  false   true  false  false  false

julia> df3
5×6 DataFrame
 Row │ x1     x2     x3     x4     x5     x6
     │ Bool   Bool   Bool   Bool   Bool   Bool
─────┼──────────────────────────────────────────
   1 │ false   true  false   true  false   true
   2 │ false  false  false  false   true  false
   3 │ false  false   true  false   true   true
   4 │  true  false   true   true   true   true
   5 │ false  false   true  false  false  false
```

Indeed we got what we wanted. I showed `df3` variable to convince you that
still all operations were done on the same data frame object and `df2` and `df3`
are still pointing to it.

Let me give an example where the difference between in-place and column replace
operations particularly matters and is a common surprise for new users.
It is a case when we want to introduce `missing` values to a column that initially does not allow them.

```
julia> df2 = copy(df)
5×6 DataFrame
 Row │ x1         x2         x3         x4        x5        x6
     │ Float64    Float64    Float64    Float64   Float64   Float64
─────┼─────────────────────────────────────────────────────────────────
   1 │ 0.364225   0.690894   0.240867   0.720774  0.331573  0.549766
   2 │ 0.225226   0.241412   0.0793279  0.418206  0.775367  0.35275
   3 │ 0.19913    0.0633375  0.767805   0.280096  0.721995  0.917259
   4 │ 0.708132   0.230088   0.702677   0.947402  0.928979  0.66101
   5 │ 0.0267573  0.0122425  0.549734   0.331788  0.32658   0.00476749

julia> df2 .= ifelse.(df .> 0.5, missing, df)
ERROR: MethodError: Cannot `convert` an object of type Missing to an object of type Float64

julia> df2[!, :] .= ifelse.(df .> 0.5, missing, df)
5×6 DataFrame
 Row │ x1               x2               x3               x4              x5              x6
     │ Float64?         Float64?         Float64?         Float64?        Float64?        Float64?
─────┼─────────────────────────────────────────────────────────────────────────────────────────────────────
   1 │       0.364225   missing                0.240867   missing               0.331573  missing
   2 │       0.225226         0.241412         0.0793279        0.418206  missing               0.35275
   3 │       0.19913          0.0633375  missing                0.280096  missing         missing
   4 │ missing                0.230088   missing          missing         missing         missing
   5 │       0.0267573        0.0122425  missing                0.331788        0.32658         0.00476749
```

Note that `df2` originally does not allow `missing` values in any of the columns. Therefore
`df2 .= ifelse.(df .> 0.5, missing, df)` fails. However, replacing `df2 .=` by `df2[!, :] .=`
works, because the `!` selector explicitly requests for overwriting of the original columns
with new ones, possibly changing their types.

# Conclusions

I hope you found these examples useful and they will help you to work with DataFrames.jl more
easily and confidently.

As a final comment let me explain why `df2 .= ifelse.(df .> 0.5, missing, df)` is fully in-place
(and does not replace the columns with proper element types like `df2[!, :] .=`) as this is a common question.
There are three reasons for this:

* performance: a fully in-place operation is faster and allocates less;
* safety: in production code we might want to make sure that the type of a column is not changed by mistake;
* design consistency: the operation was designed to work the same way as broadcasting on matrices
  (broadcasted assignment used with a matrix does allow to change its element type).
