---
layout: post
title:  "My practices for managing project dependencies in Julia"
date:   2020-05-18 06:17:13 +0200
categories: julialang
---

# Project dependencies in Julia

When you work on a project using the Julia language most likely you will use
some packages that are available in the Julia ecosystem. In particular,
[JuliaHub][juliahub] is a great place to look for packages that might be useful
for you.

Most likely the first thing you start doing is adding the package to your
default environment. This is the easiest thing to do, but has one significant
downside --- Julia package ecosystem is evolving very fast. This, in particular,
means that during your project life cycle the versions of the packages provided
by their maintainers can go up and introduce breaking changes. In consequence
your code might suddenly stop working for no apparent reason.

In this post I have collected some practices I find useful to avoid such
problems. Even if you do not end up using the functionalities of the Julia
package manager I discuss on daily basis I think it is worth to be aware of
their existence.

All examples were tested under Julia 1.4.1.

# For every project keep a separate project environment

This is a basic rule. Unless I do quick-and-dirty interactive calculations
I always create a project environment for my work.

Fortunately this is really easy. Just use `generate` command in the Julia
package manager. The steps to achieve it are easy:

1. start `julia` in the folder in which you want to create a project
2. press `]` character to enter package manager mode
3. execute `generate [target folder name]` and press enter
4. press backspace to leave this mode
5. write `exit()`

Here is a screen shot of the session where I executed these steps:
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

(@v1.4) pkg> generate test_project
 Generating  project test_project:
    test_project/Project.toml
    test_project/src/test_project.jl

julia> exit()
~$
```

Alternatively you can achieve this by running the following command:
```
~$ julia -e 'using Pkg; Pkg.generate("test_project")'
 Generating  project test_project:
    test_project/Project.toml
    test_project/src/test_project.jl
~$
```
When I use `-e` command Julia executes the commands that I pass and quits.
In this case I loaded the `Pkg` module and executed `Pkg.generate` function that
is a part of `Pkg` module API.

Let us check what is the contents of the `test_project` folder:
```
~$ cd test_project/
~/test_project$ ls -R
.:
Project.toml  src

./src:
test_project.jl
~/test_project$
```
You can see that two files were created: a `Project.toml` file in the top-level
directory that specifies the dependencies of our project
and `src/test_project.jl` which is a placeholder for our code.
Let us quickly inspect their contents:
```
~/test_project$ cat Project.toml
name = "test_project"
uuid = "d78710ad-1861-4169-903b-684d2f77c7fa"
authors = ["Bogumił Kamiński <bkamins@sgh.waw.pl>"]
version = "0.1.0"
~/test_project$ cat src/test_project.jl
module test_project

greet() = print("Hello World!")

end # module
~/test_project$
```
You can change the contents or rename the `test_project.jl` file in whatever
way you like, but do not touch `Project.toml` file as it contains the list
of dependencies of your project (currently there are none). [Here][projecttoml]
you can find the details of the specification of `Project.toml` file contents.

Now --- a crucial step is that whenever you want to work with your project
always `activate` the project environment specified by `Project.toml` file
when starting Julia.

The easiest way to do it is to make sure that you are in the folder that
contains `Project.toml` file and write:
```
~/test_project$ julia --project=.
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.4.1 (2020-04-14)
 _/ |\__'_|_|_|\__'_|  |  Official https://julialang.org/ release
|__/                   |

julia>
```
The `--project=.` part tells Julia to start the interpreter using `Project.toml`
file from the current working directory as a specification of your dependencies.

I [this post][activate] I discuss how to make Julia automatically activate the
project environment in your current working directory on startup.

Now we can check that indeed we are in a correct project environment that is
empty:

{% highlight julia %}
julia> using Pkg

julia> Pkg.status()
Project test_project v0.1.0
Status `~/test_project/Project.toml`
  (empty environment)

julia>
{% endhighlight %}

In the next section I discuss how to add packages to your project.


# When adding packages use `preserve=PRESERVE_DIRECT` keyword argument

You can add dependencies to your project using the `Pkg.add` function in the
project manager. You can find the list of the available options by running the
help for the `Pkg.add` function (I omit the output here as it is long).

It is crucial that the `Pkg.add` function has a `preserve` keyword argument that
tells Julia what it is allowed to do with the already installed packages. In
[this post][versionconflicts] I have discussed potential problems when multiple
packages you install have conflicting dependencies. Therefore my practice is to
use `preserve=PRESERVE_DIRECT` keyword argument. I do not want the packages that
my code depends on to change versions (and if there is some conflict generated
due to this restriction I prefer to get an error rather than a package version
change). However, I typically allow changing versions of recursive dependencies
as normally it should not affect my code.

Let me give an example how one can run such a command (in this case it does not
really matter if we add the `preserve=PRESERVE_DIRECT` keyword argument as we do
not have any packages installed):
{% highlight julia %}
julia> Pkg.add("Pipe", preserve=PRESERVE_DIRECT)
   Updating registry at `~/.julia/registries/General`
   Updating git-repo `https://github.com/JuliaRegistries/General.git`
  Resolving package versions...
   Updating `~/test_project/Project.toml`
  [b98c9c47] + Pipe v1.2.0
   Updating `~/test_project/Manifest.toml`
  [b98c9c47] + Pipe v1.2.0

julia>
{% endhighlight %}

If you are curious what Pipe.jl package does you can check it out
[here][pipejl].

We are informed that `Project.toml` and `Manifest.toml` were updated.
Let us inspect their contents from within Julia as an exercise:

{% highlight julia %}
julia> print(read("Project.toml", String))
name = "test_project"
uuid = "d78710ad-1861-4169-903b-684d2f77c7fa"
authors = ["Bogumił Kamiński <bkamins@sgh.waw.pl>"]
version = "0.1.0"

[deps]
Pipe = "b98c9c47-44ae-5843-9183-064241ee97a0"

julia> print(read("Manifest.toml", String))
# This file is machine-generated - editing it directly is not advised

[[Pipe]]
git-tree-sha1 = "f11840ebaf295b39319c2750f158621a96173fc5"
uuid = "b98c9c47-44ae-5843-9183-064241ee97a0"
version = "1.2.0"
julia>
{% endhighlight %}

In general `Manifest.toml` contains the exact specification of all dependencies
of our project (direct and recursive) and `Project.toml` lists only essential
information about direct dependencies.

Before we move on let me stress in what cases using `preserve=PRESERVE_DIRECT`
is most important. Assume you have worked on some project for some time already.
It had several packages as its dependencies. Now you decide that you need to add
some new package to its direct dependencies. The potential problem is that it is
possible (as you have worked on your project already for some time) that the
packages that you have installed previously have new versions available. Most
likely you **do not want** these packages to change their versions when you add
a new package as your code might stop working. This is exactly what
`preserve=PRESERVE_DIRECT` keyword argument safeguards you against.

# When updating packages use `level=UPLEVEL_PATCH` option

However, if you work on a project for some time the packages that you depend on,
might have released patches (e.g. bug fixes or documentation enhancements) that
you want to allow in your project.

You can achieve such an update by running `Pkg.update(level=UPLEVEL_PATCH)`.
Here is an example of this command run:

{% highlight julia %}
julia> Pkg.update(level=UPLEVEL_PATCH)
   Updating registry at `~/.julia/registries/General`
   Updating git-repo `https://github.com/JuliaRegistries/General.git`
   Updating `~/test_project/Project.toml`
 [no changes]
   Updating `~/test_project/Manifest.toml`
 [no changes]

julia>
{% endhighlight %}

In this case nothing happened as we have just installed that package. To check
out all the options of the `Pkg.update` function run its help. In particular, it is
OK to allow changing major or minor versions of your dependencies when running
the `Pkg.update` function, but remember that in this case your code might stop
working correctly, so if you do this please make sure to test that your code
produces the expected results after the update (and if it fails you can use the
`Pkg.undo()` function to undo the latest change to the active project).


# Final notes on using package manager

In this post I used the `Pkg` module API by calling functions. All this
functionality is also available via a package manager, which you enter by
pressing `]` in the Julia command line. The names of the commands are the same,
and you can get help on them by writing e.g. `help add`.

Finally, let me add one more comment, as this issue has raised some discussion.
The methods I describe for adding packages (with `preserve=PRESERVE_DIRECT`) and
updating them (with `level=UPLEVEL_PATCH`) are meant to be a safe way to
perform these operations (so as I have written in the introduction this is an
option to be aware of). The core idea is that when working with production
code you want to:
* separate steps of adding new packages and updating packages you already have
  in your project;
* avoid package updates to introduce breaking changes in your dependencies
  unless you explicitly allow for this.

[juliahub]: https://juliahub.com/ui/Home
[projecttoml]: https://julialang.github.io/Pkg.jl/dev/toml-files/
[activate]: https://bkamins.github.io/julialang/2020/05/10/julia-project-environments.html
[versionconflicts]: https://bkamins.github.io/julialang/2020/05/11/package-version-restrictions.html
[pipejl]: https://github.com/oxinabox/Pipe.jl
