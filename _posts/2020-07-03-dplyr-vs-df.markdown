---
layout: post
title:  "Comparing dplyr vs DataFrames.jl"
date:   2020-07-03 12:01:41 +0200
categories: julialang
---

# Introduction

This time the post is inspired by the proposal of [Andrey Oskin][Arkoniak]
(thank you for submitting it, below I have adapted business problem description
and `dplyr` source codes that Andrey provided).

Andrey shared with me typical tasks that he is faced with when doing logs
analysis. To make things concrete, assume that you have a site and you collect
users' clicks. In the output of this process you get a table with two fields:
time of click (`ts` column below, measured in seconds) and the user identifier
(`user_id` column below).

Given such data there are natural business questions, that we can ask, like:

1. How many sessions an average user has?
2. How many users have exactly two sessions?
3. Find top 10 users, ordered by the descending number of sessions?
4. What is the average time between sessions start?

Session in these questions is a more or less arbitrary thing, usually,
it has a meaning of sequence of events that come together as there is
a short time difference between consecutive events. In the examples we
assume that if a user has not clicked on our site for 900 seconds after the last
click the session is over.

What I do in this post is take a toy data set that has this structure and
`dplyr` codes that Andrey shared with me that answer the business questions
presented above and rewrite them to DataFrames.jl.

The objective of this post is to compare the syntaxes of `dplyr` and
DataFrames.jl. Therefore neither `dplyr` nor DataFrames.jl codes were tuned
to be optimal. Rather I have just taken what Andrey proposed in `dplyr` and
translated it to DataFrames.jl in a way that first came to my mind (but trying
to use piping). However, in the last part of the post I out of curiosity I
decided compare the performance of the codes.

All codes were  tested under R version 4.0.2 and dplyr 1.0.0.
For Julia I used version 1.5.0-rc1.0 and packages: DataFrames.jl 0.21.4,
Pipe.jl 1.3.0, and ShiftedArrays 1.0.0. If you do not have much experience
with setting-up Julia project environments, in [this post][envsetup] I give
a simple recipe how you can do it easily while ensuring you use exactly the same
versions of the packages as I do.

# Setting up the stage

In the first step we load the required packages, create a data frame that
will be used later and sort it by the `ts` column.

In all examples in this post I first present R code, and then Julia code.
The expected output is shown in a comment. After each step I briefly comment
on the Julia code.

*dplyr*
{% highlight r %}
library(dplyr)

df <- data.frame(ts = c(1,10,20,1000,1010,1200,2200,2220,30,500,1500,1600),
                 user_id = c(1,1,1,1,1,1,1,1,2,2,2,2)) %>%
    arrange(ts)
df
#      ts user_id
# 1     1       1
# 2    10       1
# 3    20       1
# 4    30       2
# 5   500       2
# 6  1000       1
# 7  1010       1
# 8  1200       1
# 9  1500       2
# 10 1600       2
# 11 2200       1
# 12 2220       1
{% endhighlight %}

*DataFrames.jl*
{% highlight julia %}
using DataFrames
using Pipe
using ShiftedArrays
using Statistics

df = @pipe DataFrame!(ts = [1,10,20,1000,1010,1200,2200,2220,30,500,1500,1600],
                      user_id = [1,1,1,1,1,1,1,1,2,2,2,2]) |>
    sort(_, :ts)
# 12×2 DataFrame
# │ Row │ ts    │ user_id │
# │     │ Int64 │ Int64   │
# ├─────┼───────┼─────────┤
# │ 1   │ 1     │ 1       │
# │ 2   │ 10    │ 1       │
# │ 3   │ 20    │ 1       │
# │ 4   │ 30    │ 2       │
# │ 5   │ 500   │ 2       │
# │ 6   │ 1000  │ 1       │
# │ 7   │ 1010  │ 1       │
# │ 8   │ 1200  │ 1       │
# │ 9   │ 1500  │ 2       │
# │ 10  │ 1600  │ 2       │
# │ 11  │ 2200  │ 1       │
# │ 12  │ 2220  │ 1       │
{% endhighlight %}

In this step I used two things that are worth learning:

* A `@pipe` macro from the Pipes.jl package allows to pass result of the left
  hand side of `|>` to the right hand side in the position where `_` is placed.
  In this case `_` is a first argument to `sort`.
* I used `DataFrame!` constructor; the `!` in this case means that columns
  passed to a freshly constructed data frame *are not copied* (by default
  `DataFrame` constructor copies passed columns for safety).

# Compute session identifier for each row of data

So the first task is to identify sessions in our data. For each user
a `session_id` column gives a number of session for this user, starting from
zero. Remember, that we assume that a fresh session starts for some user, if
two consecutive events for this user are separated by at least 900 seconds.

*dplyr*
{% highlight r %}
session_df <- df %>%
    group_by(user_id) %>%
    mutate(prev_ts = lag(ts)) %>%
    mutate(diff_ts = ts - prev_ts) %>%
    mutate(diff_ts = ifelse(is.na(diff_ts), 0, diff_ts)) %>%
    mutate(session_start = ifelse(diff_ts >= 900, 1, 0)) %>%
    mutate(session_id = cumsum(session_start)) %>%
    select(-prev_ts, -diff_ts, -session_start)
session_df
# # A tibble: 12 x 3
# # Groups:   user_id [2]
#       ts user_id session_id
#    <dbl>   <dbl>      <dbl>
#  1     1       1          0
#  2    10       1          0
#  3    20       1          0
#  4    30       2          0
#  5   500       2          0
#  6  1000       1          1
#  7  1010       1          1
#  8  1200       1          1
#  9  1500       2          1
# 10  1600       2          1
# 11  2200       1          2
# 12  2220       1          2
{% endhighlight %}

*DataFrames.jl*
{% highlight julia %}
session_df = @pipe df |>
    groupby(_, :user_id) |>
    combine(:ts => ts -> begin
        prev_ts = lag(ts)
        diff_ts = ts .- prev_ts
        diff_ts = coalesce.(diff_ts, 0)
        session_start = diff_ts .> 900
        session_id = cumsum(session_start)
        return (ts=ts, session_id=session_id)
    end, _, ungroup=false)
# GroupedDataFrame with 2 groups based on key: user_id
# First Group (8 rows): user_id = 1
# │ Row │ user_id │ ts    │ session_id │
# │     │ Int64   │ Int64 │ Int64      │
# ├─────┼─────────┼───────┼────────────┤
# │ 1   │ 1       │ 1     │ 0          │
# │ 2   │ 1       │ 10    │ 0          │
# │ 3   │ 1       │ 20    │ 0          │
# │ 4   │ 1       │ 1000  │ 1          │
# │ 5   │ 1       │ 1010  │ 1          │
# │ 6   │ 1       │ 1200  │ 1          │
# │ 7   │ 1       │ 2200  │ 2          │
# │ 8   │ 1       │ 2220  │ 2          │
# ⋮
# Last Group (4 rows): user_id = 2
# │ Row │ user_id │ ts    │ session_id │
# │     │ Int64   │ Int64 │ Int64      │
# ├─────┼─────────┼───────┼────────────┤
# │ 1   │ 2       │ 30    │ 0          │
# │ 2   │ 2       │ 500   │ 0          │
# │ 3   │ 2       │ 1500  │ 1          │
# │ 4   │ 2       │ 1600  │ 1          │
{% endhighlight %}

Now I could have rewritten the `dpyr` code to DataFrames.jl in many ways, but
a most natural thing to do it was for me to use the following syntax:
```
combine(source_column => transformation_function, grouped_data_frame)
```
With this approach I can conveniently define an anonymous function
within a `begin`-`end` block and return a `(ts=ts, session_id=session_id)` value
that is a `NamedTuple` and will get expanded into two columns of a data frame.

I use `ungroup=false` syntax to keep the result a `GroupedDataFrame` to match
what we get in `dplyr`.

Also, in the code of the function I use the `lag` function from ShiftedArrays.jl.

Now we have all information to answer our business questions.

# How many sessions an average user has?

*dplyr*
{% highlight r %}
session_df %>%
    summarize(session_num = max(session_id) + 1) %>%
    ungroup() %>%
    summarize(avg_sessions_per_user = sum(session_num) / nrow(.))
# # A tibble: 1 x 1
#   avg_sessions_per_user
#                   <dbl>
# 1                   2.5
{% endhighlight %}

*DataFrames.jl*
{% highlight julia %}
@pipe session_df |>
    combine(_, :session_id => (x -> maximum(x) + 1) => :session_num) |>
    combine(_, :session_num => mean => :avg_sessions_per_user)
# 1×1 DataFrame
# │ Row │ avg_sessions_per_user │
# │     │ Float64               │
# ├─────┼───────────────────────┤
# │ 1   │ 2.5                   │
{% endhighlight %}

Observe, that in the DataFrames.jl code the first `combine` is applied
to `GroupedDataFrame` while the second `combine` is applied to a `DataFrame`.

# How many users have exactly two sessions?

*dplyr*
{% highlight r %}
session_df %>%
    summarize(session_num = max(session_id) + 1) %>%
    filter(session_num == 2) %>%
    summarize(number_of_two_session_users = nrow(.))
# # A tibble: 1 x 1
#   number_of_two_session_users
                        <int>
1                           1
{% endhighlight %}

*DataFrames.jl*
{% highlight julia %}
@pipe session_df |>
    combine(_, :session_id => (x -> maximum(x) + 1) => :session_num) |>
    filter(:session_num => ==(2), _) |>
    DataFrame(number_of_two_session_users = nrow(_))
# 1×1 DataFrame
# │ Row │ number_of_two_session_users │
# │     │ Int64                       │
# ├─────┼─────────────────────────────┤
# │ 1   │ 1                           │
{% endhighlight %}

In this code observe that `:session_num => ==(2)` syntax means that in the
`filter` function we pass each element of `:session_num` column to `==(2)`
function, which is a [curried][curry] version of a standard `x == 2` comparison.

# Find top 10 users, ordered by the descending number of sessions?

*dplyr*
{% highlight r %}
session_df %>%
    summarize(session_num = max(session_id) + 1) %>%
    ungroup() %>%
    arrange(desc(session_num)) %>%
    mutate(rn = row_number()) %>%
    filter(rn <= 10) %>%
    select(-rn)
# # A tibble: 2 x 2
#   user_id session_num
#     <dbl>       <dbl>
# 1       1           3
# 2       2           2
{% endhighlight %}

*DataFrames.jl*
{% highlight julia %}
@pipe session_df |>
    combine(_, :session_id => (x -> maximum(x) + 1) => :session_num) |>
    sort(_, :session_num, rev=true) |>
    first(_, 10)
# 2×2 DataFrame
# │ Row │ user_id │ session_num │
# │     │ Int64   │ Int64       │
# ├─────┼─────────┼─────────────┤
# │ 1   │ 1       │ 3           │
# │ 2   │ 2       │ 2           │
{% endhighlight %}

Here note that in the `:session_id => (x -> maximum(x) + 1) => :session_num`
expression we have to wrap `x -> maximum(x) + 1` in parentheses to get the
correct result (if you would omit it `=> :session_num` would be treated as a
part of an anonymous function definition).

# What is the average time between sessions start?

*dplyr*
{% highlight r %}
session_df %>%
    arrange(ts) %>%
    group_by(user_id, session_id) %>%
    mutate(rn = row_number()) %>%
    filter(rn == 1) %>%
    group_by(user_id) %>%
    mutate(prev_start = lag(ts)) %>%
    filter(!is.na(prev_start)) %>%
    mutate(sess_diff = ts - prev_start) %>%
    ungroup() %>%
    summarize(avg_session_starts = mean(sess_diff))
# # A tibble: 1 x 1
#   avg_session_starts
#                <dbl>
# 1               1223
{% endhighlight %}

*DataFrames.jl*
{% highlight julia %}
@pipe session_df |>
    parent |>
    sort(_, :ts) |>
    groupby(_, [:user_id, :session_id]) |>
    combine(first, _) |>
    groupby(_, :user_id) |>
    combine(_, :ts => diff => :sess_diff) |>
    combine(_, :sess_diff => mean => :avg_session_starts)
# 1×1 DataFrame
# │ Row │ avg_session_starts │
# │     │ Float64            │
# ├─────┼────────────────────┤
# │ 1   │ 1223.0             │
{% endhighlight %}

Here we show that you can use the `parent` function to get access to the data
frame of which `GroupedDataFrame` is a view. Also a common pattern is
`combine(first, _)` to extract the first row of each group in a
`GroupedDataFrame`.

# Scaling the computations to a larger input data set

In this part I want to check the performance of `dplyr` and DataFrames.jl codes
that we have just discussed. For this I want to replicate `df` 5,000,000 times.
In order to avoid having only users number 1 and 2 (and thus to have more groups
to analyze), we will want to create new user ids in an arbitrary fashion.

We will want to have a look what is timing of the considered operations.

*dplyr*
{% highlight r %}
time_test <- function(df) {
    n <- 5*10^6
    print(system.time(df <- df %>%
        slice(rep(row_number(), n)) %>%
        mutate(user_id = user_id + rep(1:n, each=nrow(df))) %>%
        arrange(ts)
    ))
    print(system.time(
    session_df <- df %>% group_by(user_id) %>% mutate(prev_ts = lag(ts)) %>%
        mutate(diff_ts = ts - prev_ts) %>%
        mutate(diff_ts = ifelse(is.na(diff_ts), 0, diff_ts)) %>%
        mutate(session_start = ifelse(diff_ts >= 900, 1, 0)) %>%
        mutate(session_id = cumsum(session_start)) %>%
        select(-prev_ts, -diff_ts, -session_start)
    ))
    print(system.time(
    session_df %>%
        summarize(session_num = max(session_id) + 1) %>%
        ungroup() %>%
        summarize(avg_sessions_per_user = sum(session_num)/nrow(.))
    ))
    print(system.time(
    session_df %>%
        summarize(session_num = max(session_id) + 1) %>%
        filter(session_num == 1) %>%
        summarize(number_of_one_session_users = nrow(.))
    ))
    print(system.time(
    session_df %>%
        summarize(session_num = max(session_id) + 1) %>%
        ungroup() %>%
        arrange(desc(session_num)) %>%
        mutate(rn = row_number()) %>%
        filter(rn <= 10) %>%
        select(-rn)
    ))
    print(system.time(
    session_df %>%
        arrange(ts) %>%
        group_by(user_id, session_id) %>%
        mutate(rn = row_number()) %>%
        filter(rn == 1) %>%
        group_by(user_id) %>%
        mutate(prev_start = lag(ts)) %>%
        filter(!is.na(prev_start)) %>%
        mutate(sess_diff = ts - prev_start) %>%
        ungroup() %>%
        summarize(avg_session_starts = mean(sess_diff))
    ))
}
time_test(df)
#    user  system elapsed
#   6.909   2.099   9.009
#    user  system elapsed
# 222.398   2.159 224.567
# `summarise()` ungrouping output (override with `.groups` argument)
#    user  system elapsed
#   6.115   0.000   6.116
# `summarise()` ungrouping output (override with `.groups` argument)
#    user  system elapsed
#   6.842   0.000   6.843
# `summarise()` ungrouping output (override with `.groups` argument)
#    user  system elapsed
#   7.409   0.000   7.409
#    user  system elapsed
# 273.404   2.156 275.569
{% endhighlight %}

*DataFrames.jl*
{% highlight julia %}
function time_test(df)
    n = 5*10^6
    @time df = @pipe repeat(df, n) |>
        setindex!(_, _.user_id + repeat(1:n, inner=nrow(df)), !, :user_id) |>
        sort(_, :ts)
    @time session_df = @pipe df |>
        groupby(_, :user_id) |>
        combine(:ts => ts -> begin
            prev_ts = lag(ts)
            diff_ts = ts .- prev_ts
            diff_ts = coalesce.(diff_ts, 0)
            session_start = diff_ts .> 900
            session_id = cumsum(session_start)
            return (ts=ts, session_id=session_id)
        end, _, ungroup=false)
    @time @pipe session_df |>
        combine(_, :session_id => (x -> maximum(x) + 1) => :session_num) |>
        combine(_, :session_num => mean => :avg_sessions_per_user)
    @time @pipe session_df |>
        combine(_, :session_id => (x -> maximum(x) + 1) => :session_num) |>
        filter(:session_num => ==(1), _) |>
        DataFrame(avg_sessions_per_user = nrow(_))
    @time @pipe session_df |>
        combine(_, :session_id => (x -> maximum(x) + 1) => :session_num) |>
        sort(_, :session_num, rev=true) |>
        first(_, 10)
    @time @pipe session_df |>
        parent |>
        sort(_, :ts) |>
        groupby(_, [:user_id, :session_id]) |>
        combine(first, _) |>
        groupby(_, :user_id) |>
        combine(_, :ts => diff => :sess_diff) |>
        combine(_, :sess_diff => mean => :avg_session_starts)
    return nothing
end

time_test(df)
#   9.301248 seconds (37.16 M allocations: 17.919 GiB, 10.01% gc time)
#  35.576141 seconds (485.70 M allocations: 24.391 GiB, 8.58% gc time)
#   2.170619 seconds (40.30 M allocations: 1.990 GiB, 21.22% gc time)
#   1.507405 seconds (40.30 M allocations: 1.580 GiB, 8.43% gc time)
#   1.852281 seconds (40.67 M allocations: 1.634 GiB, 7.21% gc time)
#  20.191882 seconds (166.55 M allocations: 22.924 GiB, 8.62% gc time)
{% endhighlight %}

As you can see, in the example queries we have investigated DataFrames.jl turns
out to be competitive with `dplyr` in terms of timing of the operations (let me
stress again that my objective here was not to write the fastest possible codes
in R and Julia that yield the desired results, therefore these values should be
treated lightly).

[Arkoniak]: https://github.com/Arkoniak
[envsetup]: https://bkamins.github.io/julialang/2020/06/28/automatic-project-environments.html
[curry]: https://en.wikipedia.org/wiki/Currying
