---
layout: post
title:  "How to check the version of a package?"
date:   2021-02-27 15:14:22 +0200
categories: julialang
---

# Introduction

I think the most common type of question related to DataFrames.jl package is
that users are reporting that some functionality does not work as documented.

Sometimes it is indeed a bug but in the majority of cases the reason is that
the user does not have a correct version of the package installed. In this post
I discuss several ways of checking the version of the package one has in
a current project environment.

This post was written under Julia 1.6.0-rc1, DataFrames.jl 0.22.5 and Chain.jl
0.4.4 (so as usual you can expect some exercises in using DataFrames.jl).

# Elementary methods

If you are in an interactive mode then you have two basic options. The first one
uses package manager mode. Press `]` in Julia REPL and then write the following:

```
(bkamins) pkg> status
      Status `~/Project.toml`
  [4c9194b5] ABCDGraphGenerator v0.1.0 `https://github.com/bkamins/ABCDGraphGenerator.jl#master`
  [8be319e6] Chain v0.4.4
  [9a962f9c] DataAPI v1.6.0 `~/.julia/dev/DataAPI`
  [a93c6f00] DataFrames v0.22.5
```

or if you are interested in a particular package do:

```
(bkamins) pkg> status DataFrames
      Status `~/Project.toml`
  [a93c6f00] DataFrames v0.22.5
```

In the above output it is worth to note two common scenarios:

* ABCDGraphGenerator.jl is tracking `master` branch of a GitHub repository as a
  source (so it means it was not installed from Julia registry);
* DataAPI.jl is checked out for development (using `dev` command) and the package
  is tracking a local folder.

Alternatively we could have generated the same outputs using API like this:

```
julia> using Pkg

julia> Pkg.status()
      Status `~/Project.toml`
  [4c9194b5] ABCDGraphGenerator v0.1.0 `https://github.com/bkamins/ABCDGraphGenerator.jl#master`
  [8be319e6] Chain v0.4.4
  [9a962f9c] DataAPI v1.6.0 `~/.julia/dev/DataAPI`
  [a93c6f00] DataFrames v0.22.5

julia> Pkg.status("DataFrames")
      Status `~/Project.toml`
  [a93c6f00] DataFrames v0.22.5
```

The downside of both approaches is that they produce information to the screen.
However, often one is interested in processing programmatically the installed
packages status.

# Working with `Pkg.dependencies`

The `Pkg.dependencies` function returns a dictionary mapping package UUIDs
to information about them. As you can check in the documentation string
of the function the available information is stored in the following fields:

* `name`: the name of the package
* `version`: the version of the package (this is Nothing for stdlibs)
* `is_direct_dep`: the package is a direct dependency
* `is_tracking_path`: whether a package is directly tracking a directory
* `is_pinned`: whether a package is pinned
* `source`: the directory containing the source code for that package
* `dependencies`: the dependencies of that package as a vector of UUIDs

Using `Pkg.dependencies` we can easily write a function that returns a version
of the package. Here is an example:

```
julia> using Chain

julia> get_pkg_version(name::AbstractString) =
           @chain Pkg.dependencies() begin
               values
               [x for x in _ if x.name == name]
               only
               _.version
           end
get_pkg_version (generic function with 1 method)

julia> get_pkg_version("DataFrames")
v"0.22.5"
```

Here is another example getting summary statistics about installed packages in a
data frame:

```
julia> using DataFrames

julia> get_pkg_status(;direct::Bool=true) =
           @chain Pkg.dependencies() begin
               values
               DataFrame
               direct ? _[_.is_direct_dep, :] : _
               select(:name, :version,
                      [:is_tracking_path, :is_tracking_repo, :is_tracking_registry] =>
                      ByRow((a, b, c) -> ["path", "repo", "registry"][a+2b+3c]) =>
                      :tracking)
           end
get_pkg_status (generic function with 1 method)

julia> get_pkg_status()
4×3 DataFrame
 Row │ name                version  tracking
     │ String              Union…   String
─────┼───────────────────────────────────────
   1 │ DataAPI             1.6.0    path
   2 │ DataFrames          0.22.5   registry
   3 │ Chain               0.4.4    registry
   4 │ ABCDGraphGenerator  0.1.0    repo
```

As you see I have selected to provide only the most essential information
about packages in the output: name, version and whether package is tracking
registry, local path, or external repository.

If you would pass `direct=false` you get information about all available
packages (direct and indirect dependencies of the project). It is usually not
very useful, however, as the list tends to be long, as you can see here:

```
julia> get_pkg_status(direct=false)
62×3 DataFrame
 Row │ name                         version  tracking
     │ String                       Union…   String
─────┼────────────────────────────────────────────────
   1 │ OrderedCollections           1.4.0    registry
   2 │ LibSSH2_jll                           registry
   3 │ Statistics                            registry
   4 │ ArgTools                              registry
   5 │ Compat                       3.25.0   registry
   6 │ Reexport                     1.0.0    registry
   7 │ SharedArrays                          registry
  ⋮  │              ⋮                  ⋮        ⋮
  56 │ Dates                                 registry
  57 │ MbedTLS_jll                           registry
  58 │ Serialization                         registry
  59 │ IteratorInterfaceExtensions  1.0.0    registry
  60 │ Libdl                                 registry
  61 │ Artifacts                             registry
  62 │ InteractiveUtils                      registry
                                       48 rows omitted
```

# Concluding remarks

I hope you might find these patterns useful in your work with the Julia language.

Before finishing, let me mention one other case that you might occasionally
need. The above examples show you the version of the package in your current
project environment. However, in one Julia session you can change active project
environment many times. If you would be interested in getting information about
a version of the currently loaded package here is the way to do it (this will not
work for packages from stdlib as they are bundled with Julia and have a fixed
version):

```
julia> Pkg.TOML.parsefile(joinpath(pkgdir(DataFrames), "Project.toml"))["version"]
"0.22.5"
```

Let us check that indeed the loaded version does not change if we change project
environment:

```
(bkamins) pkg> status DataFrames
      Status `~/Project.toml`
  [a93c6f00] DataFrames v0.22.5

(bkamins) pkg> add DataFrames@0.21
   Resolving package versions...
    Updating `~/Project.toml`
  [a93c6f00] ↓ DataFrames v0.22.5 ⇒ v0.21.8
    Updating `~/Manifest.toml`
  [324d7699] ↓ CategoricalArrays v0.9.3 ⇒ v0.8.3
  [a8cc5b0e] - Crayons v4.0.4
  [a93c6f00] ↓ DataFrames v0.22.5 ⇒ v0.21.8
  [59287772] - Formatting v0.4.2
  [2dfb63ee] ↓ PooledArrays v1.1.0 ⇒ v0.5.3
  [08abe8d2] - PrettyTables v0.11.1
  [189a3867] ↓ Reexport v1.0.0 ⇒ v0.2.0
  Progress [========================================>]  3/3
  ? DataFrames
2 dependencies successfully precompiled in 2 seconds (21 already precompiled)
1 dependency failed but may be precompilable after restarting julia

(bkamins) pkg> status DataFrames
      Status `~/Project.toml`
  [a93c6f00] DataFrames v0.21.8

julia> Pkg.TOML.parsefile(joinpath(pkgdir(DataFrames), "Project.toml"))["version"]
"0.22.5"
```

and we see that although project version of the package is changed the loaded
version remains the same.

[binding]: https://docs.julialang.org/en/v1/manual/variables-and-scoping/#Loops-and-Comprehensions
[issue]: https://github.com/JuliaLang/julia/issues/37690
[jn]: https://github.com/vtjnash
