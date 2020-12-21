---
layout: post
title:  "DataFrames.jl minilanguage explained"
date:   2020-12-24 06:51:35 +0200
categories: julialang
---

# Introduction

As it is end-of-year I thought of writing a longer post that would be a useful
reference to [DataFrames.jl][df] users.

In DataFrames.jl we have five functions that can be used to perform
transformations of columns of a data frame:
* `combine` create a new `DataFrame` populated with columns that are results of
  transformations applied to the source data frame columns;
* `select`: create a new `DataFrame` that has the same number of rows as the
  source data frame populated with columns that are results of transformations
  applied to the source data frame columns; (the exception to the above number
  of rows invariant is `select(df)` which produces an empty data frame);
* `select!`: the same as `select` but updates the passed data frame in place;
* `transform`: the same as `select` but keeps the columns that were already
  present in the data frame (note though that these columns can be potentially
  modified by the transformations passed to `transform`);
* `transform!`: the same as `transform` but updates the passed data frame in
  place.

The same functions also work with `GroupedDataFrame` with the difference
that the transformations are applied to groups and then combined. Here, an
important distinction is that `combine` again allows transformations to produce
any number of rows and they are combined in the order of groups in the
`GroupedDataFrame`. On the other hand `select`, `select!`, `transform` and
`transform!` require transformations to produce the same number of rows for each
group as the source group and produce a result that has the same row order
as the `parent` data frame of `GroupedDataFrame` passed. This rule has two
important implications:
* it is not allowed to perform `select`, `select!`, `transform` and
  `transform!` operations on a `GroupedDataFrame` whose groups do not cover all
  rows of the `parent` data frame;
* `select`, `select!`, `transform` and `transform!`, contrary to `combine`,
  ignore the order of groups in the `GroupedDataFrame`.

In this post I want to explain what I mean when I write **transformations**.
These transformations follow a so-called DataFrames.jl minilanguage that is
largely based on using `=>` operator. You can find a specification of this
minilanguage [here][rules]. However, because the rules have to be precise they
are relatively hard to read. Therefore in this post I will introduce them by
example. I will give all the examples with data frames as a source, but as noted
above they naturally extend to `GroupedDataFrame` case so I will add the
examples for `GroupedDataFrame` only in cases when it is especially relevant (in
order to read these parts of the post please earlier check out how `groupby`
function works in DataFrames.jl).

This post is written under Julia 1.5.3 and DataFrames.jl 0.22.2.

The data frame that we are going to use in our examples is defined follows
(I sampled some random data to populate it, with the exception of one name
to celebrate [this][game] recent masterpiece):

```
julia> df = DataFrame(id = 1:6,
                      name = ["Aaron Aardvark", "Belen Barboza",
                              "春 陈", "Даниил Дубов",
                              "Elżbieta Elbląg", "Felipe Fittipaldi"],
                      age = [50, 45, 40, 35, 30, 25],
                      eye = ["blue", "brown", "hazel", "blue", "green", "brown"],
                      grade_1 = [95, 90, 85, 90, 95, 90],
                      grade_2 = [75, 80, 65, 90, 75, 95],
                      grade_3 = [85, 85, 90, 85, 80, 85])
6×7 DataFrame
 Row │ id     name               age    eye     grade_1  grade_2  grade_3
     │ Int64  String             Int64  String  Int64    Int64    Int64
─────┼────────────────────────────────────────────────────────────────────
   1 │     1  Aaron Aardvark        50  blue         95       75       85
   2 │     2  Belen Barboza         45  brown        90       80       85
   3 │     3  春 陈                 40  hazel        85       65       90
   4 │     4  Даниил Дубов          35  blue         90       90       85
   5 │     5  Elżbieta Elbląg       30  green        95       75       80
   6 │     6  Felipe Fittipaldi     25  brown        90       95       85

```

# Column selection

These are the most simple operations that the discussed functions can perform.
In this case the result of `combine` and `select` are the same (using
`transform` is not very useful here, as it retains all the columns anyway).

The allowed column selectors are: column number, column name, a regular
expression, `Not`, `Between`, `Cols`, `All()`, and `:`. In order to select a
column or a set of columns you just pass them as arguments. Here is an example
of a simple column selection:

```
julia> select(df, r"grade", :)
6×7 DataFrame
 Row │ grade_1  grade_2  grade_3  id     name               age    eye
     │ Int64    Int64    Int64    Int64  String             Int64  String
─────┼────────────────────────────────────────────────────────────────────
   1 │      95       75       85      1  Aaron Aardvark        50  blue
   2 │      90       80       85      2  Belen Barboza         45  brown
   3 │      85       65       90      3  春 陈                 40  hazel
   4 │      90       90       85      4  Даниил Дубов          35  blue
   5 │      95       75       80      5  Elżbieta Elbląg       30  green
   6 │      90       95       85      6  Felipe Fittipaldi     25  brown
```

Above, we use a common pattern showing how one can move columns to the front of
the data frame. The `r"grade"` regular expression picks all columns that contain
`"grade"` string in their name, then `:` picks all the remaining columns.

An important rule here is that if we pass several column selectors that pick
multiple columns, as you can see above, it is allowed that they select the same
columns and they are included in the target data frame using first-in-first-out
policy. However, it is not allowed to select the same column twice using single
column selectors, so this works:

```
julia> select(df, "name", [2])
6×1 DataFrame
 Row │ name
     │ String
─────┼───────────────────
   1 │ Aaron Aardvark
   2 │ Belen Barboza
   3 │ 春 陈
   4 │ Даниил Дубов
   5 │ Elżbieta Elbląg
   6 │ Felipe Fittipaldi
```

but this fails:

```
julia> select(df, "name", 2)
ERROR: ArgumentError: duplicate output column name: :name
```

The same rule applies also to more advanced transformations that I cover below
(i.e. duplicates are allowed only in plain column selectors picking multiple
columns).

# Basic transformations

The simplest way to specify a transformation is

```
source_column => transformation => target_column_name
```

In this scenario the `source_column` is passed as an argument to
`transformation` and stored in `target_column_name` column.

Here is an example:

```
julia> using Statistics

julia> select(df, :age => mean => :meanage)
6×1 DataFrame
 Row │ meanage
     │ Float64
─────┼─────────
   1 │    37.5
   2 │    37.5
   3 │    37.5
   4 │    37.5
   5 │    37.5
   6 │    37.5

julia> combine(df, :age => mean => :meanage)
1×1 DataFrame
 Row │ meanage
     │ Float64
─────┼─────────
   1 │    37.5
```

Observe that because `select` produces as many rows in the produced data frame
as there are rows in the source data frame, a single value is repeated
accordingly. This is not the case for `combine`. However, if other columns
in `combine` would produce multiple rows the repetition also happens:

```
julia> combine(df, :age => mean => :meanage, :eye => unique => :eye)
4×2 DataFrame
 Row │ meanage  eye
     │ Float64  String
─────┼─────────────────
   1 │    37.5  blue
   2 │    37.5  brown
   3 │    37.5  hazel
   4 │    37.5  green
```

Note, however, that it is not allowed to return vectors of different lengths in
different transformations:
```
julia> combine(df, :age, :eye => unique => :eye)
ERROR: ArgumentError: New columns must have the same length as old columns
```

Let us discuss one more example, this case using `GroupedDataFrame`. As you
can see below vectors get expanded into multiple columns by default, e.g.:

```
julia> combine(groupby(df, :eye),
               :name => (x -> identity(x)) => :name,
               :age => mean => :age)
6×3 DataFrame
 Row │ eye     name               age
     │ String  String             Float64
─────┼────────────────────────────────────
   1 │ blue    Aaron Aardvark        42.5
   2 │ blue    Даниил Дубов          42.5
   3 │ brown   Belen Barboza         35.0
   4 │ brown   Felipe Fittipaldi     35.0
   5 │ hazel   春 陈                 40.0
   6 │ green   Elżbieta Elbląg       30.0
```

(as a side note observe that I used `(x -> identity(x))` transformation although
in this case it would be enough just to write `:name`; the reason is that I
wanted to highlight that if you use an anonymous function in the transformation
definition you have to wrap it in `(` and `)` as shown above; otherwise the
expression is parsed by Julia in an unexpected way due to the `=>` operator
precedence level)

However, in some cases it would be more natural not to expand the `:name`
column into multiple rows. You can easily achieve it by protecting the value
returned by the transformation using `Ref` (just as in broadcasting):

```
julia> combine(groupby(df, :eye),
                      :name => (x -> Ref(identity(x))) => :name,
                      :age => mean => :age)
4×3 DataFrame
 Row │ eye     name                               age
     │ String  SubArray…                          Float64
─────┼────────────────────────────────────────────────────
   1 │ blue    ["Aaron Aardvark", "Даниил Дубов…     42.5
   2 │ brown   ["Belen Barboza", "Felipe Fittip…     35.0
   3 │ hazel   ["春 陈"]                             40.0
   4 │ green   ["Elżbieta Elbląg"]                   30.0
```

In this case the vectors produced by our transformation are kept in a single row
of the produced data frame. Note that in this case we also could have just
written:

```
julia> combine(groupby(df, :eye), :name => Ref => :name, :age => mean => :age)
4×3 DataFrame
 Row │ eye     name                               age
     │ String  SubArray…                          Float64
─────┼────────────────────────────────────────────────────
   1 │ blue    ["Aaron Aardvark", "Даниил Дубов…     42.5
   2 │ brown   ["Belen Barboza", "Felipe Fittip…     35.0
   3 │ hazel   ["春 陈"]                             40.0
   4 │ green   ["Elżbieta Elbląg"]                   30.0
```

as we were applying `identity` transformation to our data, which can be skipped.

Often you want to apply some function not to the whole column of a data frame,
but rather to its individual elements. Normally one would achieve this using
broadcasting like this:

```
julia> select(df, :name => (x -> uppercase.(x)) => :NAME)
6×1 DataFrame
 Row │ NAME
     │ String
─────┼───────────────────
   1 │ AARON AARDVARK
   2 │ BELEN BARBOZA
   3 │ 春 陈
   4 │ ДАНИИЛ ДУБОВ
   5 │ ELŻBIETA ELBLĄG
   6 │ FELIPE FITTIPALDI
```

This pattern is encountered very often in practice, therefore there is a `ByRow`
convenience wrapper for a function that creates its broadcasted variant:

```
julia> select(df, :name => ByRow(uppercase) => :NAME)
6×1 DataFrame
 Row │ NAME
     │ String
─────┼───────────────────
   1 │ AARON AARDVARK
   2 │ BELEN BARBOZA
   3 │ 春 陈
   4 │ ДАНИИЛ ДУБОВ
   5 │ ELŻBIETA ELBLĄG
   6 │ FELIPE FITTIPALDI
```

One can skip specifying a target column name, in which case it is generated
automatically by suffixing source column name by function name that is applied
to it, e.g.:

```
julia> select(df, :name => ByRow(uppercase))
6×1 DataFrame
 Row │ name_uppercase
     │ String
─────┼───────────────────
   1 │ AARON AARDVARK
   2 │ BELEN BARBOZA
   3 │ 春 陈
   4 │ ДАНИИЛ ДУБОВ
   5 │ ELŻBIETA ELBLĄG
   6 │ FELIPE FITTIPALDI
```

If you want to avoid this then pass `renemecols=false` keyword argument (this
is mostly useful if you perform transformations):

```
julia> select(df,
              :grade_1 => ByRow(x -> x / 100),
              :grade_2 => ByRow(x -> x / 100),
              :grade_3 => ByRow(x -> x / 100),
              renamecols=false)
6×3 DataFrame
 Row │ grade_1  grade_2  grade_3
     │ Float64  Float64  Float64
─────┼───────────────────────────
   1 │    0.95     0.75     0.85
   2 │    0.9      0.8      0.85
   3 │    0.85     0.65     0.9
   4 │    0.9      0.9      0.85
   5 │    0.95     0.75     0.8
   6 │    0.9      0.95     0.85
```

Similarly to skipping target column name you can skip the transformation part of
the pattern we discussed, in which case it is assumed to be the `identity`
function. In effect we get a way to rename columns in transformations:

```
julia> select(df, r"grade", :grade_1 => :g1, :grade_2 => :g2, :grade_3 => :g3)
6×6 DataFrame
 Row │ grade_1  grade_2  grade_3  g1     g2     g3
     │ Int64    Int64    Int64    Int64  Int64  Int64
─────┼────────────────────────────────────────────────
   1 │      95       75       85     95     75     85
   2 │      90       80       85     90     80     85
   3 │      85       65       90     85     65     90
   4 │      90       90       85     90     90     85
   5 │      95       75       80     95     75     80
   6 │      90       95       85     90     95     85
```

If you want to perform multiple column transformations that have a similar
structure (as in the example above) you can pass a vector of transformations:

```
julia> select(df,
              [x => ByRow(x -> x / 100) for x in [:grade_1, :grade_2, :grade_3]],
              renamecols=false)
6×3 DataFrame
 Row │ grade_1  grade_2  grade_3
     │ Float64  Float64  Float64
─────┼───────────────────────────
   1 │    0.95     0.75     0.85
   2 │    0.9      0.8      0.85
   3 │    0.85     0.65     0.9
   4 │    0.9      0.9      0.85
   5 │    0.95     0.75     0.8
   6 │    0.9      0.95     0.85
```

or even shorter using broadcasting and taking advantage of `names` function:

```
julia> select(df, names(df, r"grade") .=> ByRow(x -> x / 100), renamecols=false)
6×3 DataFrame
 Row │ grade_1  grade_2  grade_3
     │ Float64  Float64  Float64
─────┼───────────────────────────
   1 │    0.95     0.75     0.85
   2 │    0.9      0.8      0.85
   3 │    0.85     0.65     0.9
   4 │    0.9      0.9      0.85
   5 │    0.95     0.75     0.8
   6 │    0.9      0.95     0.85
```

You now know almost all about single column selection. The only thing left to
learn is that `nrow` function has a special treatment. It does not require
passing source column name (but allows specifying of target column name; the
default name is `:nrow`) and produces number of rows in the passed data frame.
Here is an example of both:

```
julia> combine(groupby(df, :eye), nrow, nrow => :count)
4×3 DataFrame
 Row │ eye     nrow   count
     │ String  Int64  Int64
─────┼──────────────────────
   1 │ blue        2      2
   2 │ brown       2      2
   3 │ hazel       1      1
   4 │ green       1      1
```

That was a lot of information. Probably this is a good moment to take a short
break.

Next we move on to the cases when there are multiple source columns or multiple
columns are produced as a result of the transformation.

# Multiple source columns

The simplest way to pass multiple columns to a transformation is to pass their
list in a vector. In this case these columns are passed as consecutive
positional arguments to the function. Here is an example (also observe how
automatic column naming is done in this case):

```
julia> combine(df,
               [:age, :grade_1] => cor,
               [:age, :grade_2] => cor,
               [:age, :grade_3] => cor)
1×3 DataFrame
 Row │ age_grade_1_cor  age_grade_2_cor  age_grade_3_cor
     │ Float64          Float64          Float64
─────┼───────────────────────────────────────────────────
   1 │       0.0710072        -0.536745         0.338062
```

Alternatively you can pass the columns as a single argument being a
`NamedTuple`, in order to achieve this wrap the list of passed columns in
`AsTable`. Here is a simple example:

```
julia> combine(df, :grade_1, :grade_2,
               AsTable([:grade_1, :grade_2]) => ByRow(x -> x.grade_1 > x.grade_2))
6×3 DataFrame
 Row │ grade_1  grade_2  grade_1_grade_2_function
     │ Int64    Int64    Bool
─────┼────────────────────────────────────────────
   1 │      95       75                      true
   2 │      90       80                      true
   3 │      85       65                      true
   4 │      90       90                     false
   5 │      95       75                      true
   6 │      90       95                     false
```

Let us see these two rules at play in a common task of finding a sum of several
columns:
```
julia> select(df, names(df, r"grade") => +, AsTable(names(df, r"grade")) => sum)
6×2 DataFrame
 Row │ grade_1_grade_2_grade_3_+  grade_1_grade_2_grade_3_sum
     │ Int64                      Int64
─────┼────────────────────────────────────────────────────────
   1 │                       255                          255
   2 │                       255                          255
   3 │                       240                          240
   4 │                       265                          265
   5 │                       250                          250
   6 │                       270                          270
```

# Multiple target columns

Normally all the transformation functions assume that a single column is
returned from a transformation function. Here is a more advanced example:

```
julia> select(df,
              :name => ByRow(x -> (; ([:firsname, :lastname] .=> split(x))...)))
6×1 DataFrame
 Row │ name_function
     │ NamedTuple…
─────┼───────────────────────────────────
   1 │ (firsname = "Aaron", lastname = …
   2 │ (firsname = "Belen", lastname = …
   3 │ (firsname = "春", lastname = "陈…
   4 │ (firsname = "Даниил", lastname =…
   5 │ (firsname = "Elżbieta", lastname…
   6 │ (firsname = "Felipe", lastname =…
```

In the above code we have used the following pattern that is a convenient way
of programmatic creation of `NamedTuple`s:

```
julia> (; :a => 1, :b => 2)
(a = 1, b = 2)

julia> (; [:a => 1, :b => 2]...)
(a = 1, b = 2)
```

However, it is natural to ask if it is possible to produce the separate
`:firstname` and `:lastname` columns in the data frame. You can do it by
adding `AsTable` as target column names specifier:

```
julia> select(df, :name =>
                  ByRow(x -> (; ([:firsname, :lastname] .=> split(x))...)) =>
                  AsTable)
6×2 DataFrame
 Row │ firsname   lastname
     │ SubStrin…  SubStrin…
─────┼───────────────────────
   1 │ Aaron      Aardvark
   2 │ Belen      Barboza
   3 │ 春         陈
   4 │ Даниил     Дубов
   5 │ Elżbieta   Elbląg
   6 │ Felipe     Fittipaldi
```

In general it is not required that one has to produce a vector of `NamedTuple`s
to get this result. It is enough to have any standard iterable (e.g. a vector
or a `Tuple`):

```
julia> select(df, :name => ByRow(split) => AsTable)
6×2 DataFrame
 Row │ x1         x2
     │ SubStrin…  SubStrin…
─────┼───────────────────────
   1 │ Aaron      Aardvark
   2 │ Belen      Barboza
   3 │ 春         陈
   4 │ Даниил     Дубов
   5 │ Elżbieta   Elbląg
   6 │ Felipe     Fittipaldi
```

Note that as `split` produces a vector (without column names) the names are
generated automatically as `:x1` and `:x2`. You can override this behavior by
specifying the target column names explicitly:

```
julia> select(df, :name => ByRow(split) => [:firsname, :lastname])
6×2 DataFrame
 Row │ firsname   lastname
     │ SubStrin…  SubStrin…
─────┼───────────────────────
   1 │ Aaron      Aardvark
   2 │ Belen      Barboza
   3 │ 春         陈
   4 │ Даниил     Дубов
   5 │ Elżbieta   Elbląg
   6 │ Felipe     Fittipaldi
```

Several values that can be returned by a transformation are treated to produce
multiple columns by default. Therefore they are not allowed to be returned
from a function unless `AsTable` or multiple target column names are specified.

Here is an example:
```
julia> combine(df, :age => x -> (age=x, age2 = x.^2))
ERROR: ArgumentError: Table returned but a single output column was expected

julia> combine(df, :age => (x -> (age=x, age2 = x.^2)) => AsTable)
6×2 DataFrame
 Row │ age    age2
     │ Int64  Int64
─────┼──────────────
   1 │    50   2500
   2 │    45   2025
   3 │    40   1600
   4 │    35   1225
   5 │    30    900
   6 │    25    625
```

The list of return value types treated in this way consists of four elements:
* `AbstractDataFrame`,
* `DataFrameRow`,
* `AbstractMatrix`,
* `NamedTuple`.

In the example above we returned a `NamedTuple`. Note, however, that returning
a `NamedTuple` is not the same as returning a vector of `NamedTuples` (as we
did in the topmost example in this section) as this is allowed.

For the purposes of detection of column names conflicts multiple columns
returned are treated as a series of single returned columns (so duplicates are
not allowed):

```
julia> select(df, :name => ByRow(split) => [:name, :lastname], :name)
ERROR: ArgumentError: duplicate output column name: :name
```

Just for completeness of discussion note that this works:

```
julia> select(df, :name => ByRow(split) => [:name, :lastname], [:name])
6×2 DataFrame
 Row │ name       lastname
     │ SubStrin…  SubStrin…
─────┼───────────────────────
   1 │ Aaron      Aardvark
   2 │ Belen      Barboza
   3 │ 春         陈
   4 │ Даниил     Дубов
   5 │ Elżbieta   Elbląg
   6 │ Felipe     Fittipaldi

```

as `[:names]` is considered to be a multiple column selector (thus it is not
producing a duplicate and is just ignored in this case).

# Traditional transformation style

The traditional transformation style in the `combine` function was to pass a
transformation as a function (without using the `=>` minilanguage). This style
is also allowed currently in all transformation functions. Additionally if you
want to use this approach you can pass the function as a first argument of the
function (which is often convenient in combination with `do` block style).

In this traditional style we are not allowed to specify source columns.
Therefore such a function always takes a data frame (it is most useful with
`GroupedDataFrame` where the data frame is representing a given group of the
source data frame)

Similarly traditional style does not allow specifying target column names.
Therefore the following defaults are taken:
* if `AbstractDataFrame`, `DataFrameRow`, `AbstractMatrix`, or `NamedTuple` is
  returned then it is assumed to produce multiple columns (in the case of
  `NamedTuple` it must contain only vectors or only single values, as if they
  are mixed an error is thrown).
* all other return values are treated as a single column (where returning a
  vector produces multiple rows and returning any other value produces one row).

Here are examples of this style:

```
julia> combine(groupby(df, :eye)) do sdf
           return mean(sdf.age)
       end
4×2 DataFrame
 Row │ eye     x1
     │ String  Float64
─────┼─────────────────
   1 │ blue       42.5
   2 │ brown      35.0
   3 │ hazel      40.0
   4 │ green      30.0

julia> combine(groupby(df, :eye), sdf -> mean(sdf.age))
4×2 DataFrame
 Row │ eye     x1
     │ String  Float64
─────┼─────────────────
   1 │ blue       42.5
   2 │ brown      35.0
   3 │ hazel      40.0
   4 │ green      30.0

julia> combine(groupby(df, :eye),
               sdf -> (count = nrow(sdf),
                       mean_age = mean(sdf.age),
                       std_age = std(sdf.age)))
4×4 DataFrame
 Row │ eye     count  mean_age  std_age
     │ String  Int64  Float64   Float64
─────┼───────────────────────────────────
   1 │ blue        2      42.5   10.6066
   2 │ brown       2      35.0   14.1421
   3 │ hazel       1      40.0  NaN
   4 │ green       1      30.0  NaN
```

# Special treatment of grouping columns

As a special rule for processing of `GroupedDataFrame` object is that
it is not allow to change the values stored in the grouping columns as they
are kept by default.

Here is a simple example:

```
julia> combine(groupby(df, :eye), :eye => ByRow(uppercase) => :eye)
ERROR: ArgumentError: column :eye in returned data frame is not equal to grouping key :eye
```

However, you are allowed to do this if you drop the grouping columns by passing
`keepkeys=false` keyword argument:

```
julia> transform(groupby(df, :eye),
                 :eye => ByRow(uppercase) => :eye,
                 keepkeys=false)
6×7 DataFrame
 Row │ id     name               age    eye     grade_1  grade_2  grade_3
     │ Int64  String             Int64  String  Int64    Int64    Int64
─────┼────────────────────────────────────────────────────────────────────
   1 │     1  Aaron Aardvark        50  BLUE         95       75       85
   2 │     2  Belen Barboza         45  BROWN        90       80       85
   3 │     3  春 陈                 40  HAZEL        85       65       90
   4 │     4  Даниил Дубов          35  BLUE         90       90       85
   5 │     5  Elżbieta Elbląg       30  GREEN        95       75       80
   6 │     6  Felipe Fittipaldi     25  BROWN        90       95       85
```

(note that for `transform` the key columns would be retained in the produced
data frame even with `keepkeys=false`; in this case this keyword argument only
influences the fact if we check that key columns have not changed in this case)

# Conclusions

This was long. As a conclusion let me comment that all the above-mentioned
styles can be freely combined in a single transformation call:

```
julia> combine(groupby(df, :eye),
               r"grade", names(df, r"grade") => (+) => :total_grade,
               sdf -> nrow(sdf), nrow, :age => :AGE)
6×8 DataFrame
 Row │ eye     grade_1  grade_2  grade_3  total_grade  x1     nrow   AGE
     │ String  Int64    Int64    Int64    Int64        Int64  Int64  Int64
─────┼─────────────────────────────────────────────────────────────────────
   1 │ blue         95       75       85          255      2      2     50
   2 │ blue         90       90       85          265      2      2     35
   3 │ brown        90       80       85          255      2      2     45
   4 │ brown        90       95       85          270      2      2     25
   5 │ hazel        85       65       90          240      1      1     40
   6 │ green        95       75       80          250      1      1     30
```

If you got here then congratulations -- you have mastered the DataFrames.jl
transformation minilanguage.

[df]: https://github.com/JuliaData/DataFrames.jl
[rules]: https://dataframes.juliadata.org/latest/man/split_apply_combine/
[game]: https://www.chessgames.com/perl/chessgame?gid=2015531
