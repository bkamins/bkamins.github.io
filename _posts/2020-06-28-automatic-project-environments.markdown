---
layout: post
title:  "How to embed project environment setup in your Julia script"
date:   2020-06-28 19:45:07 +0200
categories: julialang
---

# Project dependency management in Julia

I have already written several posts about project dependency management
practices in Julia. You can find these materials [here][ref1], [here][ref2],
and [here][ref3].

In all these posts I emphasize that one should use project-specific Project.toml
and Manifest.toml files. But the question is what should one do if one wants
to distribute some Julia code as a single .jl file without Project.toml
and Manifest.toml files attached?

Now, you might ask why someone would have a problem with adding Project.toml
and Manifest.toml to ones files when distributing them?
There are several potential reasons:
* It is easy to forget to attach them to your source code.
* By accident you might update your Project.toml and Manifest.toml files
  (and unless you use version control of your repository you have a problem;
  sometimes `undo` command in the Package Manager mode will help you, but not
  in all cases).
* You store many projects in the same directory (so it gets hard to manage
  Project.toml and Manifest.toml files then) --- this scenario happens when
  you have e.g. a collection of small scripts for data preprocessing that you
  reuse across different tasks.
* If an inexperienced user of Julia gets your source code along with
  Project.toml and Manifest.toml, one might have a problem understanding how to
  properly activate the project environment and (especially when working on
  a fresh installation) run into problems by not running `instantiate` command
  in the Package Manager.

If any of these scenarios is your use case in this post I explain how to
automatically set up a proper project environment within a Julia script.

The codes were run under Julia 1.4.2.

# Embedding project environment setup in your Julia script

I will use here a simple example of a script that depends on DataFrames.jl
and CSV.jl, creates a random `DataFrame` and writes it to a disk.
So the operation we want to do is the following:

{% highlight julia %}
using CSV, DataFrames

CSV.write("random_file.csv", DataFrame(rand(100, 5)))
{% endhighlight %}

Assume that I want to use CSV.jl version 0.6.1 and DataFrames.jl version 0.20.1
(note that I picked some old versions of the packages to make sure that indeed
we test that we get what we want).

How to make sure they are properly loaded by the script above? Just add the
following header to it:

{% highlight julia %}
using Pkg

cd(mktempdir()) do
    Pkg.activate(".")
    Pkg.add(PackageSpec(name="CSV", version="0.6.1"))
    Pkg.add(PackageSpec(name="DataFrames", version="0.20.1"),
            preserve=PRESERVE_DIRECT)
    Pkg.status()
end
{% endhighlight %}

What are important elements of this process:

* Using `mktempdir` we create a temporary directory that will be deleted after
  our script terminates (so each time we run the code a new random directory
  will be created and instantiated).
* As we add packages one-by one and want their specific versions in all but the
  first `Pkg.add` commands I use `preserve=PRESERVE_DIRECT` to ensure that the
  Package Manager does not change the version of an already added package.
* I run `Pkg.status` to visually make sure that all is installed correctly.
* Note that Julia normally uses a federated repository of packages; this means
  that when such a script is run several times Julia will just reuse the
  already downloaded packages (so it does not have to fetch packages from the
  Internet every time and the process is relatively fast)
* Finally, in this way you ensure that it is clear within the script what are
  versions of its dependencies.

So here is a full code of our example (if you would want to copy-paste it
for testing):

{% highlight julia %}
using Pkg

cd(mktempdir()) do
    Pkg.activate(".")
    Pkg.add(PackageSpec(name="CSV", version="0.6.1"))
    Pkg.add(PackageSpec(name="DataFrames", version="0.20.1"),
            preserve=PRESERVE_DIRECT)
    Pkg.status()
end

using CSV, DataFrames

CSV.write("random_file.csv", DataFrame(rand(100, 5)))
{% endhighlight %}

Here is the output I got when running it (I show everything that is printed
as in this case it is relevant):
```
julia> using Pkg

julia> cd(mktempdir()) do
           Pkg.activate(".")
           Pkg.add(PackageSpec(name="CSV", version="0.6.1"))
           Pkg.add(PackageSpec(name="DataFrames", version="0.20.1"),
                   preserve=PRESERVE_DIRECT)
           Pkg.status()
       end
 Activating new environment at `/tmp/jl_Mit7P2/Project.toml`
   Updating registry at `~/.julia/registries/General`
   Updating git-repo `https://github.com/JuliaRegistries/General.git`
  Resolving package versions...
   Updating `/tmp/jl_Mit7P2/Project.toml`
  [336ed68f] + CSV v0.6.1
   Updating `/tmp/jl_Mit7P2/Manifest.toml`
  [336ed68f] + CSV v0.6.1
  [324d7699] + CategoricalArrays v0.7.7
  [34da2185] + Compat v3.12.0
  [9a962f9c] + DataAPI v1.3.0
  [a93c6f00] + DataFrames v0.20.2
  [864edb3b] + DataStructures v0.17.19
  [e2d170a0] + DataValueInterfaces v1.0.0
  [48062228] + FilePathsBase v0.7.0
  [41ab1584] + InvertedIndices v1.0.0
  [82899510] + IteratorInterfaceExtensions v1.0.0
  [682c06a0] + JSON v0.21.0
  [e1d29d7a] + Missings v0.4.3
  [bac558e1] + OrderedCollections v1.2.0
  [69de0a69] + Parsers v1.0.6
  [2dfb63ee] + PooledArrays v0.5.3
  [189a3867] + Reexport v0.2.0
  [a2af1166] + SortingAlgorithms v0.3.1
  [3783bdb8] + TableTraits v1.0.0
  [bd369af6] + Tables v1.0.4
  [ea10d353] + WeakRefStrings v0.6.2
  [2a0f44e3] + Base64
  [ade2ca70] + Dates
  [8bb1440f] + DelimitedFiles
  [8ba89e20] + Distributed
  [9fa8497b] + Future
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
  [1a1011a3] + SharedArrays
  [6462fe0b] + Sockets
  [2f01184e] + SparseArrays
  [10745b16] + Statistics
  [8dfed614] + Test
  [cf7118a7] + UUIDs
  [4ec0a83e] + Unicode
  Resolving package versions...
   Updating `/tmp/jl_Mit7P2/Project.toml`
  [a93c6f00] + DataFrames v0.20.1
   Updating `/tmp/jl_Mit7P2/Manifest.toml`
  [a93c6f00] ↓ DataFrames v0.20.2 ⇒ v0.20.1
Status `/tmp/jl_Mit7P2/Project.toml`
  [336ed68f] CSV v0.6.1
  [a93c6f00] DataFrames v0.20.1

julia> using CSV, DataFrames

julia> CSV.write("random_file.csv", DataFrame(rand(100, 5)))
"random_file.csv"
```

What are important things to note here:
* Before you terminate the Julia session the temporary directory will exist
  (in my case its name is /tmp/jl_Mit7P2 as you can see in the output; you can
  check yourself that it is the case on your system).
* When you exit Julia the directory gets deleted (again --- you can make sure
  that this is the case).
* Initially DataFrames.jl was added in version v0.20.2, and later we downgraded
  its version (so as you can see sequential adding of packages influences the
  versions of packages already added, that is why `preserve=PRESERVE_DIRECT`
  is an important switch to make sure that the package you added are in correct
  versions).
* The Package Manager updates registry by looking up
  https://github.com/JuliaRegistries/General.git, so it has some cost, but later
  in my case the packages themselves were not downloaded as they were already
  present in the federated package repository (so the process is quite fast,
  yet still has a non-negligible cost; the additional benefit of doing this
  is that you do not have to run `instantiate` as adding packages explicitly
  ensures that they get instantiated).

# What are the limitations of the proposed approach

There are three details that one should keep in mind when using the method
that I have presented in this post:
* Sharing Project.toml and Manifest.toml makes Julia recreate the project
  environment exactly. In the process I have described above we only ensure
  that the packages we install have a fixed version. Versions of their recursive
  dependencies will be dynamically resolved by the Package Manager (and thus
  might differ from versions used when the script was created; however, in
  practice this is not a problem unless some of these packages do not correctly
  follow [Semantic Versioning][semver]).
* The process that dynamically creates the project environment costs a bit ---
  it is of course better to just ship Project.toml and Manifest.toml along your
  files so that you do not have to pay extra start-up time (but if your script
  does some heavy computations this is probably negligible).

[ref1]: https://bkamins.github.io/julialang/2020/05/18/project-workflow.html
[ref2]: https://bkamins.github.io/julialang/2020/05/11/package-version-restrictions.html
[ref3]: https://bkamins.github.io/julialang/2020/05/10/julia-project-environments.html
[semver]: https://semver.org/
