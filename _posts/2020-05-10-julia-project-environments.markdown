---
layout: post
title:  "Activating project environment in Julia REPL automatically"
date:   2020-05-10 10:14:07 +0200
categories: julialang
---

# Project specific environments in Julia

One of the great things about the Julia programming language is that it is very
easy to manage the dependencies of your project. In short, two files
`Project.toml` and `Manifest.toml` uniquely specify what packages and in what
versions are required by your scripts. You can find out how it works in detail
[in the Julia manual][juliadoc_pkg].

What I describe in this post was tested under Julia 1.4.1.

If not specified otherwise, Julia is started in a default (global) environment.
Here is a quick example. Start a new Julia session and press `]` to switch to
package manager mode. You will see the following scrren

```
~$ julia
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.4.1 (2020-04-14)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

(@v1.4) pkg>
```

The `(@v1.4)` prefix tells you that you are in a default environment.

You can switch to the project specific environment using two options:

* the `activate` command from Julia package manager, or
* by passing `--project` command line argument when starting Julia.

For instance to activate the project environment in the current working
directory use `.` as a path specifier and write either:

{% highlight julia %}
(@v1.4) pkg> activate .
 Activating new environment at `~/Project.toml`
{% endhighlight %}

or start Julia with the following command `julia --project=.`.

Also as a special case you can write `julia --project=@.` in which case Julia
will be looking for files specifying the project files in the current directory
or its parents, as described [here][juliadoc_atdot].

All this functionality is very powerful and greatly simplifies reproducibility
of my work. In particular on a single machine one can have many projects,
and each of them can have different dependencies.

# My daily workflow with project environments

I like the `Project.toml`/`Manifest.toml` combo very much and use it for every
project I work on. There is one problem though --- you have to remember to
activate the project environment when you start Julia.

If you use Jupyter Notebooks, the default Julia kernel that is installed by
IJulia.jl automatically uses `--project=@.` flag when starting Julia,
see [here][ijulia_work]. So in this case everything works nice.

However, my typical working setup is to use a classical text editor and
a terminal where Julia (a.k.a Julia REPL) is run. A natural question is how can
one easily get a similar functionality in this case.

What I do is the following. In my `~/.julia/config/startup.jl` file
I have added the following lines:

{% highlight julia %}
using Pkg
if isfile("Project.toml") && isfile("Manifest.toml")
    Pkg.activate(".")
end
{% endhighlight %}

Now, every time I start my Julia REPL and the *current* working directory contains
`Project.toml` and `Manifest.toml` files they will get activated.

Note that I use `.` --- I typically do not want Julia to look for the files
in the parent directories.

In rare cases when I want to disable this functionality I run `julia` with
`--startup-file=no` flag to disable loading `~/.julia/config/startup.jl` file.

If you have never used `startup.jl` file you can read a description how it works
in [the Julia manual][juliadoc_intro].

[juliadoc_pkg]: https://docs.julialang.org/en/v1/stdlib/Pkg/
[juliadoc_atdot]:https://docs.julialang.org/en/v1/manual/environment-variables/index.html#JULIA_PROJECT-1
[ijulia_work]: https://github.com/JuliaLang/IJulia.jl#julia-projects
[juliadoc_intro]: https://docs.julialang.org/en/v1/manual/getting-started/
