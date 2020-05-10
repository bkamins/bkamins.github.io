---
layout: post
title:  "Understanding package version restrictions in Julia"
date:   2020-05-11 00:42:32 +0200
categories: julialang
---

# Dependencies of the packages in Julia

Each package in Julia has a list of other packages it depends on.
They are specified in `Project.toml` file. [Here][daataframes_project] is an
example from DataFrames.jl pacakge release 0.21.0:

```
name = "DataFrames"
uuid = "a93c6f00-e57d-5684-b7b6-d8193f3e46c0"
version = "0.21.0"

[deps]
CategoricalArrays = "324d7699-5711-5eae-9e2f-1d82baa6b597"
Compat = "34da2185-b29b-5c13-b0c7-acf172513d20"
DataAPI = "9a962f9c-6df0-11e9-0e5d-c546b8b5ee8a"
Future = "9fa8497b-333b-5362-9e8d-4d0656e87820"
InvertedIndices = "41ab1584-1d38-5bbf-9106-f11c6c58b48f"
IteratorInterfaceExtensions = "82899510-4779-5014-852e-03e436cf321d"
Missings = "e1d29d7a-bbdc-5cf2-9ac0-f12de2c33e28"
PooledArrays = "2dfb63ee-cc39-5dd5-95bd-886bf059d720"
Printf = "de0858da-6303-5e67-8744-51eddeeeb8d7"
Reexport = "189a3867-3050-52da-a836-e630ba90ab69"
REPL = "3fa0cd96-eef1-5676-8a61-b3b8758bbffb"
SortingAlgorithms = "a2af1166-a08f-5f64-846c-94a0d3cef48c"
Statistics = "10745b16-79ce-11e8-11f9-7d13ad32a3b2"
Tables = "bd369af6-aec1-5ad0-b16a-f7cc5008161c"
TableTraits = "3783bdb8-4a98-5b6b-af9a-565f29a5fe9c"
Unicode = "4ec0a83e-493e-50e2-b9ac-8f72acf5a8f5"

[extras]
DataStructures = "864edb3b-99cc-5e75-8d2d-829cb0a9cfe8"
DataValues = "e7dc6d0d-1eca-5fa6-8ad6-5aecde8b7ea5"
Dates = "ade2ca70-3891-5945-98fb-dc099432e06a"
Logging = "56ddb016-857b-54e1-b83d-db4d58db5568"
Random = "9a3f8284-a2c9-5f02-9a11-845980a1fd5c"
Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"

[targets]
test = ["DataStructures", "DataValues", "Dates", "Logging", "Random", "Test"]

[compat]
julia = "1"
CategoricalArrays = "0.8"
Compat = "2.2, 3"
DataAPI = "1.2"
InvertedIndices = "1"
IteratorInterfaceExtensions = "0.1.1, 1"
Missings = "0.4.2"
PooledArrays = "0.5"
Reexport = "0.1, 0.2"
SortingAlgorithms = "0.1, 0.2, 0.3"
Tables = "1"
TableTraits = "0.4, 1"
```

The details how to read this file are given [here][pkg_project]. For this post
the important part is `[compat]` section (again [here][pkg_compat] you can find
the details). This section informs Julia package manager which versions of other
packages are allowed to be installed with version `0.21.0` of DataFrames.jl.

# Conflicting dependency requirements

The crucial thing is that when you install packages Julia package manager
analyzes the `[compat]` section of each package required to be installed in the
project and tries to find such a combination of package versions that is
possible to be active at the same time.

Sometimes it is impossible to find such a combination and you get an error
that starts with the words `Unsatisfiable requirements detected for package`.
This is a problematic situation, and [here][pkg_conflict] you have a description
how to read such a message, and [here][pkg_resolve] some tips how to fix them.

While getting an error is a critically problematic situation it has one positive
element: you know that something went wrong.

In this post we want to concentrate on a less critical case: you install several
packages, you get no errors, but for some strange reasons your code does not
work as expected. The likely situation is caused by the fact that the package
manager has decided that it is only possible to install some old version of a
package, that e.g. can have a different API than the latest version of the
package that you have implemented your code against.

From my experience such situations are quite common and hard to detect
especially if you have dozens of installed packages.

Let me give a simple example of such a situation. Recently I wanted to install
Plots.jl and GraphPlot.jl packages. Here is a dump of the steps I have made in
a clean environment:

```
(@v1.4) pkg> activate .
 Activating new environment at `~/Project.toml`

(bkamins) pkg> status
Status `~/Project.toml`
  (empty environment)

(bkamins) pkg> add Plots
   Updating registry at `~/.julia/registries/General`
   Updating git-repo `https://github.com/JuliaRegistries/General.git`
  Resolving package versions...
   Updating `~/Project.toml`
  [91a5bcdd] + Plots v1.2.3
   Updating `~/Manifest.toml`
  [6e34b625] + Bzip2_jll v1.0.6+2
  [35d6a980] + ColorSchemes v3.9.0
  [3da002f7] + ColorTypes v0.10.3
  [5ae59095] + Colors v0.12.0
  [d38c429a] + Contour v0.5.3
  [9a962f9c] + DataAPI v1.3.0
  [864edb3b] + DataStructures v0.17.15
  [c87230d0] + FFMPEG v0.3.0
  [b22a6f82] + FFMPEG_jll v4.1.0+3
  [53c48c17] + FixedPointNumbers v0.8.0
  [d7e528f0] + FreeType2_jll v2.10.1+2
  [559328eb] + FriBidi_jll v1.0.5+3
  [28b8d3ca] + GR v0.49.1
  [4d00f742] + GeometryTypes v0.8.3
  [682c06a0] + JSON v0.21.0
  [c1c5ebd0] + LAME_jll v3.100.0+1
  [dd192d2f] + LibVPX_jll v1.8.1+1
  [442fdcdd] + Measures v0.3.1
  [e1d29d7a] + Missings v0.4.3
  [77ba4419] + NaNMath v0.3.3
  [e7412a2a] + Ogg_jll v1.3.4+0
  [458c3c95] + OpenSSL_jll v1.1.1+2
  [91d4177d] + Opus_jll v1.3.1+1
  [bac558e1] + OrderedCollections v1.2.0
  [69de0a69] + Parsers v1.0.3
  [ccf2f8ad] + PlotThemes v2.0.0
  [995b91a9] + PlotUtils v1.0.2
  [91a5bcdd] + Plots v1.2.3
  [3cdcf5f2] + RecipesBase v1.0.1
  [01d81517] + RecipesPipeline v0.1.9
  [189a3867] + Reexport v0.2.0
  [ae029012] + Requires v1.0.1
  [992d4aef] + Showoff v0.3.1
  [a2af1166] + SortingAlgorithms v0.3.1
  [90137ffa] + StaticArrays v0.12.3
  [2913bbd2] + StatsBase v0.33.0
  [83775a58] + Zlib_jll v1.2.11+9
  [0ac62f75] + libass_jll v0.14.0+2
  [f638f0a6] + libfdk_aac_jll v0.1.6+2
  [f27f6e37] + libvorbis_jll v1.3.6+4
  [1270edf5] + x264_jll v2019.5.25+2
  [dfaa095f] + x265_jll v3.0.0+1
  [2a0f44e3] + Base64
  [ade2ca70] + Dates
  [8bb1440f] + DelimitedFiles
  [8ba89e20] + Distributed
  [b77e0a4c] + InteractiveUtils
  [76f85450] + LibGit2
  [8f399da3] + Libdl
  [37e2e46d] + LinearAlgebra
  [56ddb016] + Logging
  [d6f4376e] + Markdown
  [a63ad114] + Mmap
  [44cfe95a] + Pkg
  [de0858da] + Printf
  [3fa0cd96] + REPL
  [9a3f8284] + Random
  [ea8e919c] + SHA
  [9e88b42a] + Serialization
  [6462fe0b] + Sockets
  [2f01184e] + SparseArrays
  [10745b16] + Statistics
  [8dfed614] + Test
  [cf7118a7] + UUIDs
  [4ec0a83e] + Unicode

(bkamins) pkg> add GraphPlot
  Resolving package versions...
   Updating `~/Project.toml`
  [a2cc645c] + GraphPlot v0.3.1
   Updating `~/Manifest.toml`
  [ec485272] + ArnoldiMethod v0.0.4
  [a81c6b42] + Compose v0.8.2
  [a2cc645c] + GraphPlot v0.3.1
  [d25df0c9] + Inflate v0.1.2
  [c8e1da08] + IterTools v1.3.0
  [093fc24a] + LightGraphs v1.3.3
  [1914dd2f] + MacroTools v0.5.5
  [699a6c99] + SimpleTraits v0.9.2
  [1a1011a3] + SharedArrays

(bkamins) pkg> status
Status `~/Project.toml`
  [a2cc645c] GraphPlot v0.3.1
  [91a5bcdd] Plots v1.2.3
```

All seems to went through without a problem. Except that the current release of
GraphPlot.jl (as of time of writing this post) is 0.4.2.

So we try adding this version of the package:

```
(bkamins) pkg> add GraphPlot@0.4.2
  Resolving package versions...
   Updating `~/Project.toml`
  [a2cc645c] ↑ GraphPlot v0.3.1 ⇒ v0.4.2
  [91a5bcdd] ↓ Plots v1.2.3 ⇒ v1.0.14
   Updating `~/Manifest.toml`
  [35d6a980] - ColorSchemes v3.9.0
  [3da002f7] ↓ ColorTypes v0.10.3 ⇒ v0.9.1
  [5ae59095] ↓ Colors v0.12.0 ⇒ v0.11.2
  [53c48c17] ↓ FixedPointNumbers v0.8.0 ⇒ v0.7.1
  [28b8d3ca] ↓ GR v0.49.1 ⇒ v0.48.0
  [a2cc645c] ↑ GraphPlot v0.3.1 ⇒ v0.4.2
  [ccf2f8ad] ↓ PlotThemes v2.0.0 ⇒ v1.0.3
  [995b91a9] ↓ PlotUtils v1.0.2 ⇒ v0.6.5
  [91a5bcdd] ↓ Plots v1.2.3 ⇒ v1.0.14
```

And we see that multiple packages got downgraded to their earlier versions.

In the next two sections we will discuss: (a) how to detect such situations
automatically, and (b) how to diagnose what is the root cause of the problem.

# Automatically detecting packages installed in a version that is not latest

Here is a short snippet that lists you the packages that are installed but
are not in their latest versions. It is probably not 100% proof for corner
cases of various configurations (this is what Julia package manager does, and
it is a piece of complex code) but shoul be good enough for typical use cases.
It produces us a dictionary of packages that are not installed in their latest
versions giving us a tuple indicating installed and latest version for each
such package.

{% highlight julia %}
julia> using Pkg

julia> cd(joinpath(DEPOT_PATH[1], "registries", "General")) do
           deps = Pkg.dependencies()
           registry = Pkg.TOML.parse(read("Registry.toml", String))
           general_pkgs = registry["packages"]

           constrained = Dict{String, Tuple{VersionNumber,VersionNumber}}()
           for (uuid, dep) in deps
               suuid = string(uuid)
               dep.is_direct_dep || continue
               dep.version === nothing && continue
               haskey(general_pkgs, suuid) || continue
               pkg_meta = general_pkgs[suuid]
               pkg_path = joinpath(pkg_meta["path"], "Versions.toml")
               versions = Pkg.TOML.parse(read(pkg_path, String))
               newest = maximum(VersionNumber.(keys(versions)))
               if newest > dep.version
                   constrained[dep.name] = (dep.version, newest)
               end
           end

           return constrained
       end
Dict{String,Tuple{VersionNumber,VersionNumber}} with 1 entry:
  "Plots" => (v"1.0.14", v"1.2.3")
{% endhighlight %}

As you can see we get an information that Plots.jl currently is not in its
latest version as expected. We already know that it was downgraded when forcing
to install GraphPlot.jl in version 0.4.2.

Let me give a few comments how the presented code works:
* we run our function in the `registries/General` subdirectory of the first
  entry of the `DEPOT_PATH` variable (this is where Julia package manager first
  looks for installed packages);
* the `deps` variable is a a dictionary of all dependencies of the current
  project;
* `Registry.toml` holds information about available packages in the registry,
  we parse it and store dictionary with package information in the
  `general_pkgs` variable;
* `constrained` is a dictionary that we will populate with information about
  packages that are not in their latest version
* we iteratively scan all packages in `deps` dictionary we are interested only
  in packages that are direct dependencies (that were explicitly installed),
  have a version (packages from standard library do not have a version but we
  do not install them anyway), and that are present in `general_pkgs` dictionary
  (this means that if you are using several depots or sone non-standard way
  of installing packages this code is not guaranteed to be 100% correct);
* if we have a package that is interesting for us we get a list of all its
  available versions from `Versions.toml` that is stored in its directory,
  we extract all the versions, convert them to `VersionNumber` type and select
  a maximum version (note that `VersionNumber` type has a properly defned order
  so this is safe to do);
* finally we compare the installed and latest versions of the packages and if
  they differ this information is stored and later returned.

# Diagnosing the problems with package version constraints

We have learned that upgrading GraphPlot.jl package to version 0.4.2 caused us
dependency problems. Let us then find out what are its copat sepecifications.
Here is the code that does the job:

{% highlight julia %}
julia> using GraphPlot
[ Info: Precompiling GraphPlot [a2cc645c-3eea-5389-862e-a155d0052231]

julia> toml_loc = joinpath(dirname(pathof(GraphPlot)), "..", "Project.toml");

julia> Pkg.TOML.parse(read(toml_loc, String))["compat"]
Dict{String,Any} with 7 entries:
  "Compose"               => "0.7, 0.8"
  "julia"                 => "1"
  "ColorTypes"            => "0.9"
  "ArnoldiMethod"         => "0.0.4"
  "LightGraphs"           => "1.1"
  "VisualRegressionTests" => ">= 0.2"
  "Colors"                => "0.11"
{% endhighlight %}

And we see that ColorTypes.jl and Colors.jl had to be downgraded when we
installed the 0.4.2 version of GraphPlot.jl (we can see in the listing above
that they were downgraded). And downgrading these two packages, in cascade,
made Julia package manager change the versions of other packages, Plots.jl in
particular.

In the code above note that in order to get a path of `GraphPlot.jl` file from
GraphPlot.jl package we first have to load it with `using GraphPlot` command.
Then in order to properly construct the `toml_loc` variable we have to go one
step up in the path with `..` as `GraphPlot.jl` file is located in `src`
subdirectory in the package sources.

In order to resolve such problems systematically is best to submit a PR to the
packages that have outdated dependencies. This is exactly what I did
[here][example_pr] (and the PR got merged as of this writing).

[daataframes_project]: https://github.com/JuliaData/DataFrames.jl/blob/v0.21.0/Project.toml
[pkg_project]: https://julialang.github.io/Pkg.jl/v1/creating-packages/
[pkg_compat]: https://julialang.github.io/Pkg.jl/v1/compatibility/
[pkg_conflict]: https://julialang.github.io/Pkg.jl/dev/managing-packages/#conflicts-1
[pkg_resolve]: https://julialang.github.io/Pkg.jl/dev/compatibility/#Fixing-conflicts-1
[example_pr]: https://github.com/JuliaGraphs/GraphPlot.jl/pull/109
