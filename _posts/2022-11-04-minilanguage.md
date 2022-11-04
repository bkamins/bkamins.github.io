---
layout: post
title:  "DataFrames.jl 1.4: operation specification syntax news"
date:   2022-11-04 07:03:44 +0200
categories: julialang
---

# Introduction

Operation specification syntax in DataFrames.jl is used to pass information
how functions like `select`, `transform`, or `combine` should process
data frames or grouped data frames.

If you have never used it I recommend you to first read an
[introductory post][mini] about it.

Today I want to discuss what additions to operation specification language we
made in DataFrames.jl 1.4.

The presented code was tested under Julia 1.8.2 and DataFrames.jl 1.4.2.

# Preliminaries

Operation specification syntax is built around ETL (extract-transform-load)
process, that you might know from data integration. Its general form is:

```
[source columns] => [operation] => [target columns names]
```

Here is a simple example:

```
julia> using DataFrames

julia> df = DataFrame(customer=[1, 1, 2, 2, 2, 3],
                      transaction_id=1:6,
                      volume=[2, 3, 1, 4, 5, 9])
6×3 DataFrame
 Row │ customer  transaction_id  volume
     │ Int64     Int64           Int64
─────┼──────────────────────────────────
   1 │        1               1       2
   2 │        1               2       3
   3 │        2               3       1
   4 │        2               4       4
   5 │        2               5       5
   6 │        3               6       9

julia> combine(df, :volume => sum => :total_volume)
1×1 DataFrame
 Row │ total_volume
     │ Int64
─────┼──────────────
   1 │           24

julia> combine(groupby(df, :customer), :volume => sum => :total_volume)
3×2 DataFrame
 Row │ customer  total_volume
     │ Int64     Int64
─────┼────────────────────────
   1 │        1             5
   2 │        2            10
   3 │        3             9
```

In these examples we first aggregated `volume` column to get total volume
for the whole data frame, and next we computed total volume per customer.

In both cases we used the same operation specification syntax:

```
:volume => sum => :total_volume
```

Which says:

* *extract* column `:volume`;
* *transform* it using `sum`;
* *load* it to `:total_volume` column.

However, there are cases when there is no natural source column on which we
might want to perform computations. One of such common cases is getting the
number of rows per group. For this special case we have a short syntax `nrow`
or `nrow => [target column]` to compute number of rows in a data frame or in
each group of a data frame. Notice that there is no *extract* part in this
syntax as number of rows is not a property of a specific column, but of a data
frame as a whole.

Here is an example how it works:

```
julia> combine(df, nrow)
1×1 DataFrame
 Row │ nrow
     │ Int64
─────┼───────
   1 │     6

julia> combine(groupby(df, :customer), nrow => :transactions_per_customer)
3×2 DataFrame
 Row │ customer  transactions_per_customer
     │ Int64     Int64
─────┼─────────────────────────────────────
   1 │        1                          2
   2 │        2                          3
   3 │        3                          1
```

There are three other common operations that have the same nature:

* adding a column with row number;
* adding a column with group number (makes sense only for working with grouped
  data frame);
* computing fraction of rows (also for grouped data frames only).

In DataFrames.jl 1.4 these three operations are now supported through
`eachindex`, `groupindices`, and `proprow` operations. Let me show you how they
work.

# Adding a column with row number

This is the simplest functionality. The `eachindex` operation adds row number
in a data frame or per group in a grouped data frame. Here is an example:

```
julia> combine(df, eachindex, :transaction_id)
6×2 DataFrame
 Row │ eachindex  transaction_id
     │ Int64      Int64
─────┼───────────────────────────
   1 │         1               1
   2 │         2               2
   3 │         3               3
   4 │         4               4
   5 │         5               5
   6 │         6               6

julia> combine(groupby(df, :customer),
               eachindex => :transaction_number,
               :transaction_id)
6×3 DataFrame
 Row │ customer  transaction_number  transaction_id
     │ Int64     Int64               Int64
─────┼──────────────────────────────────────────────
   1 │        1                   1               1
   2 │        1                   2               2
   3 │        2                   1               3
   4 │        2                   2               4
   5 │        2                   3               5
   6 │        3                   1               6
```

Note that when we work on a whole data frame we got the same column as
`:transaction_id`. However, when working on a grouped data frame we got
transaction numbers per customer.

# Adding a column with group number

The `eachindex` operation added row within group. So it is natural to ask for a
function that does produce a group number. The `groupindices` operation is
designed to achieve this goal. Here is an example:

```
julia> combine(groupby(df, :customer), groupindices)
3×2 DataFrame
 Row │ customer  groupindices
     │ Int64     Int64
─────┼────────────────────────
   1 │        1             1
   2 │        2             2
   3 │        3             3

julia> combine(groupby(df, :customer), groupindices => :customer_id, :customer)
6×2 DataFrame
 Row │ customer  customer_id
     │ Int64     Int64
─────┼───────────────────────
   1 │        1            1
   2 │        1            1
   3 │        2            2
   4 │        2            2
   5 │        2            2
   6 │        3            3
```

Note that in our example the produced numbers are the same as values in the
`customer` column. However, in general it does not have to be the case.
Let us subset the grouped data frame before the operation:

```
julia> gdf = groupby(df, :customer)[[3, 2]]
GroupedDataFrame with 2 groups based on key: customer
First Group (1 row): customer = 3
 Row │ customer  transaction_id  volume
     │ Int64     Int64           Int64
─────┼──────────────────────────────────
   1 │        3               6       9
⋮
Last Group (3 rows): customer = 2
 Row │ customer  transaction_id  volume
     │ Int64     Int64           Int64
─────┼──────────────────────────────────
   1 │        2               3       1
   2 │        2               4       4
   3 │        2               5       5

julia> combine(gdf, groupindices)
2×2 DataFrame
 Row │ customer  groupindices
     │ Int64     Int64
─────┼────────────────────────
   1 │        3             1
   2 │        2             2
```

As you can see `groupindices` returns the number of a group within the grouped
data frame.

As I have mentioned earlier this operation is not supported for data frames:

```
julia> combine(df, groupindices)
ERROR: ArgumentError: groupindices only supports `GroupedDataFrame` as an
argument. Additionally it can be used in transformation functions (combine,
select, etc.) when processing a `GroupedDataFrame`, using the syntax
`groupindices => target_col_name` or just `groupindices`
```

# Computing the fraction of rows per group

DataFrames.jl supports `nrow` convenience function for a long time already as
it was a common use case that users needed. An almost as frequent use-case is
to get a faction of rows per group. This can be achieved using the `proprow`
operation:

```
julia> combine(groupby(df, :customer), nrow, proprow)
3×3 DataFrame
 Row │ customer  nrow   proprow
     │ Int64     Int64  Float64
─────┼───────────────────────────
   1 │        1      2  0.333333
   2 │        2      3  0.5
   3 │        3      1  0.166667

julia> combine(groupby(df, :customer), nrow => :count, proprow => :proportion)
3×3 DataFrame
 Row │ customer  count  proportion
     │ Int64     Int64  Float64
─────┼─────────────────────────────
   1 │        1      2    0.333333
   2 │        2      3    0.5
   3 │        3      1    0.166667
```

Similarly to `groupindices` the `proprow` operation is only supported for
grouped data frames:

```
julia> combine(df, proprow)
ERROR: ArgumentError: proprow can only be used in transformation functions
(combine, select, etc.) when processing a `GroupedDataFrame`, using the syntax
`proprow => target_col_name` or just `proprow`
```

# Conclusions

I hope you will find the `eachindex`, `groupindices` and `proprow` operations
useful in your daily data wrangling with DataFrames.jl.

[mini]: https://bkamins.github.io/julialang/2020/12/24/minilanguage.html

