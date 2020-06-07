---
layout: post
title:  "Learning who is the author of the current state of the Julia language"
date:   2020-06-07 18:31:11 +0200
categories: julialang
---

# Development of the Julia language

During 20 years of my work as a researcher I have used numerous programming
languages to do scientific computing, chiefly R, Python, and Java.
However, when I learned Julia I immediately felt this is a to-go solution,
although I started using it when version 0.3 was released and the language
and its ecosystem was still immature.

Currently Julia has reached version 1.4.2 and in many fields its package
ecosystem provides best-in-class functionality.

A natural question to as is who has made this happen. It is easy enough to find
out on the GitHub page of the Julia project [here][julia_contributors].
However, the default GitHub interface allows you to only see contributions
by number of commits, additions or deletions. We can learn from this that
[Jeff Bezanson][jeffbezanson] is a leader by far in all these categories.

However, the statistics show you the whole history of the git repository.
I was always curious who is the author of the current state of the code.
Essentially, what I wanted to do is blame the whole repository and count
the distribution of the number of lines committed by the authors.

The problem is that by default `git` does not give you such an option.
There are ways to achieve this, which I discuss below. The project was
interesting for me, because I think it nicely shows what Julia offers you
when you have a scripting task at hand.

# Before we start

In order to follow the examples below you need to have `git` installed.
Also you should have `git-extras` installed. If you are on Ubuntu just write
`sudo apt install git-extras` and it should be added.

In order to analyze the repository we need to download it to our local machine
e.g. to `julia_src` folder.
This can be done using the following command (warning! it takes some time):

```
~$ git clone https://github.com/JuliaLang/julia.git julia_src
Cloning into 'julia_src'...
remote: Enumerating objects: 83, done.
remote: Counting objects: 100% (83/83), done.
remote: Compressing objects: 100% (78/78), done.
remote: Total 325678 (delta 31), reused 16 (delta 5), pack-reused 325595
Receiving objects: 100% (325678/325678), 181.28 MiB | 1.20 MiB/s, done.
Resolving deltas: 100% (244259/244259), done.
```

Now switch our working directory to the newly downloaded repository:
```
~$ cd julia_src
~/julia_src (master)$
```

# Using git

You can get the information we want using the `summary` command provided by
`git-extras`. Here is how you can do it:

{% highlight bash %}
~/julia_src (master)$ time git summary --line

 project  : julia_src
 lines    : 469506
 authors  :
  70542 Jeff Bezanson                     15.0%
  65223 Jameson Nash                      13.9%
  31294 Keno Fischer                      6.7%
  28562 Katharine Hyatt                   6.1%
  24794 Yichao Yu                         5.3%
  15353 Michael Hatherly                  3.3%
  11146 Stefan Karpinski                  2.4%
  10955 Kristoffer Carlsson               2.3%
  10216 Steven G. Johnson                 2.2%
   8976 Tim Holy                          1.9%
   8763 Rafael Fourquet                   1.9%
   8587 Andreas Noack Jensen              1.8%
   7992 Sacha Verweij                     1.7%
   7934 Fredrik Ekre                      1.7%
   6750 Matt Bauman                       1.4%
   6277 Simon Byrne                       1.3%
   5943 Amit Murthy                       1.3%
   5895 Jacob Quinn                       1.3%
   5392 Milan Bouchet-Valat               1.1%
   5373 Alex Arslan                       1.1%
   4792 Tony Kelman                       1.0%
   4712 Curtis Vogt                       1.0%
   4637 Viral B. Shah                     1.0%
[...]

real    8m31.993s
user    8m1.853s
sys 0m39.650s
{% endhighlight %}

The whole list is quite long so I have cut it down to show only people with at
least 1.0% contribution. As you can see from the distribution
[Jameson Nash][jamesonnash] is really close to [Jeff Bezanson][jeffbezanson]
in the ranking.

As you can see I have additionally added `time` in front of the command to see
how long the operation took. For a such large repository as this one
(note that it has almost 500,000 lines of code) it is quite time consuming.

The first thing I did was search over the Internet and I have found the
following proposal [here][bash_solition]:

{% highlight bash %}
git ls-files | while read f; do git blame --line-porcelain $f \
| grep '^author '; done | sort -f | uniq -ic | sort -n
{% endhighlight %}

The solution finished in 4 minutes and 14 seconds, so it was two times faster
(the downside is that it does not produce a nice percentage information).

In general it lead me to thinking about writing a Julia script that would do the
job and check its speed. In the next section you can find my take on it.

# Using Julia

In the solution I use FreqTables.jl, ProgressMeter.jl, and Pipe.jl in the
following versions:
```
(@v1.4) pkg> status FreqTables ProgressMeter Pipe
Status `~/.julia/environments/v1.4/Project.toml`
  [da1fdf0e] FreqTables v0.4.0
  [b98c9c47] Pipe v1.2.0
  [92933f4c] ProgressMeter v1.3.0
```

Here is the code that does the job of listing authors of all lines in the git
repository:

{% highlight julia %}
using FreqTables, ProgressMeter, Random

function get_git_data()
    println("Using ", Threads.nthreads(), " threads")
    files = readlines(`git ls-files`)
    shuffle!(files)
    auths = String[]
    l = Threads.SpinLock()
    p = Progress(length(files))
    Threads.@threads for f in files
        isempty(f) && continue
        lines = readlines(`git blame --line-porcelain $f`)
        filter!(k -> startswith(k, "author "), lines)
        Threads.lock(l)
        append!(auths, chop.(lines, head=7, tail=0))
        Threads.unlock(l)
        next!(p)
    end
    println()
    return auths
end
{% endhighlight %}

As you can see I am using `Threads.@threads` to use multiple threads for
my computations. In variable `p` I keep a progress meter that helps me
to visually track how the computations go.

In the code a line that looks innocent but is actually quite relevant is
`shuffle!(files)`. You might wonder why do I randomly reorder files for
processing. The reason is that the files most probably (and in fact also
actually) do not have the same cost of processing using `git blame`. Therefore
I do not want to have *expensive* files clumped together. This has two benefits:

* ProgressMeter.jl is able to quickly give me a good estimate of ETA (e.g. if
  cheap files were clumped together at the beginning of processing the
  estimate would be overly optimistic);
* `Threads.@threads` does static allocation of jobs to threads; this against
  means that we do prefer to shuffle jobs in order to reduce the risk that
  all expensive jobs go to a single thread, which would negatively affect the
  overall processing time.

Finally note that I wrap `append!` to `auths` vector in a lock to avoid
race condition (different threads potentially might try to update `auths` at
the same time). This is not needed for `next!(p)` operation as ProgressMeter.jl
is thread-safe.

Now let us test the above code. First start Julia using four threads
(you can change it of course to other number of threads) using the command:
```
~/julia_src (master)$ JULIA_NUM_THREADS=4 julia
```
(on Windows do `set JULIA_NUM_THREADS=4` before running Julia)

Next load the script I have given above. You are now ready for the test. Here
is the code I have run on my machine:

{% highlight julia %}
julia> using Pipe
julia> @time @pipe get_git_data() |>
                   freqtable |>
                   prop |>
                   sort!(_, rev=true) |>
                   filter(>=(0.01), _)
Using 4 threads
Progress: 100%|██████████████████████████████████████████████████|  ETA: 0:00:00
 97.533423 seconds (8.14 M allocations: 1.353 GiB, 0.11% gc time)
22-element Named Array{Float64,1}
Dim1                 │
─────────────────────┼──────────
Jeff Bezanson        │  0.149081
Jameson Nash         │  0.141561
Keno Fischer         │ 0.0661324
Katharine Hyatt      │ 0.0602627
Yichao Yu            │ 0.0523127
Michael Hatherly     │ 0.0323932
Stefan Karpinski     │ 0.0235169
Kristoffer Carlsson  │ 0.0231223
Steven G. Johnson    │ 0.0215547
Tim Holy             │ 0.0189384
Rafael Fourquet      │  0.018489
Andreas Noack Jensen │ 0.0181176
Fredrik Ekre         │ 0.0169466
Sacha Verweij        │ 0.0168623
Matt Bauman          │ 0.0142418
Simon Byrne          │ 0.0132501
Amit Murthy          │ 0.0125391
Jacob Quinn          │ 0.0124378
Milan Bouchet-Valat  │ 0.0113765
Alex Arslan          │ 0.0113364
Curtis Vogt          │ 0.0101254
Tony Kelman          │ 0.0101148
{% endhighlight %}

As you can see I am well under 2 minutes now.

In the last part of code I have used [Pipe.jl][pipe] which greatly facilitates
using pipes in Julia (there is also a very nice package
[Underscores.jl][underscores] which I recommend you
to investigate; it has more functionality but this comes at the cost of being
a bit more complex to master).

What Pipe.jl does is best described by a section of its manual, so I just reuse
it here:

> if after `@pipe` you place a underscore in the right hand of `|>`,
> it will be replaced with the left hand side. So:
> ```
> @pipe a |> b(x, _) # == b(x, a)
> ```

I hope you enjoyed this little exercise (and now we know exactly whose code we
run when using Julia).

# P.S. Setting up your environment

As you probably know I am obsessed with proper environment setup. In an [earlier
post][env] I discussed that you should always make sure you run proper versions
of the packages. What is a quick way to set up the environment for the project
described in this post?

When you are in Julia REPL (e.g. started as instructed above in the `julia_src`
directory) switch to the package manager mode by pressing `]` and execute the
following commands (I am showing the whole output which is a bit long but allows
you to check which packages got recursively added to Manifest.toml):
```
(@v1.4) pkg> activate .
 Activating new environment at `~/julia_src/Project.toml`

(julia_src) pkg> add FreqTables@0.4.0 Pipe@1.2.0 ProgressMeter@1.3.0
   Updating registry at `~/.julia/registries/General`
   Updating git-repo `https://github.com/JuliaRegistries/General.git`
  Resolving package versions...
  Installed Parsers ─ v1.0.5
   Updating `~/julia_src/Project.toml`
  [da1fdf0e] + FreqTables v0.4.0
  [b98c9c47] + Pipe v1.2.0
  [92933f4c] + ProgressMeter v1.3.0
   Updating `~/julia_src/Manifest.toml`
  [324d7699] + CategoricalArrays v0.8.1
  [861a8166] + Combinatorics v1.0.2
  [9a962f9c] + DataAPI v1.3.0
  [864edb3b] + DataStructures v0.17.17
  [e2d170a0] + DataValueInterfaces v1.0.0
  [da1fdf0e] + FreqTables v0.4.0
  [41ab1584] + InvertedIndices v1.0.0
  [82899510] + IteratorInterfaceExtensions v1.0.0
  [682c06a0] + JSON v0.21.0
  [e1d29d7a] + Missings v0.4.3
  [86f7a689] + NamedArrays v0.9.4
  [bac558e1] + OrderedCollections v1.2.0
  [69de0a69] + Parsers v1.0.5
  [b98c9c47] + Pipe v1.2.0
  [92933f4c] + ProgressMeter v1.3.0
  [ae029012] + Requires v1.0.1
  [3783bdb8] + TableTraits v1.0.0
  [bd369af6] + Tables v1.0.4
  [2a0f44e3] + Base64
  [ade2ca70] + Dates
  [8bb1440f] + DelimitedFiles
  [8ba89e20] + Distributed
  [9fa8497b] + Future
  [b77e0a4c] + InteractiveUtils
  [8f399da3] + Libdl
  [37e2e46d] + LinearAlgebra
  [56ddb016] + Logging
  [d6f4376e] + Markdown
  [a63ad114] + Mmap
  [de0858da] + Printf
  [9a3f8284] + Random
  [ea8e919c] + SHA
  [9e88b42a] + Serialization
  [6462fe0b] + Sockets
  [2f01184e] + SparseArrays
  [10745b16] + Statistics
  [8dfed614] + Test
  [cf7118a7] + UUIDs
  [4ec0a83e] + Unicode

(julia_src) pkg> status
Status `~/julia_src/Project.toml`
  [da1fdf0e] FreqTables v0.4.0
  [b98c9c47] Pipe v1.2.0
  [92933f4c] ProgressMeter v1.3.0
```
Now you are sure all will work as expected. Just press backspace to leave the
package manager mode and you are ready to run the examples.

[julia_contributors]: https://github.com/JuliaLang/julia/graphs/contributors
[jeffbezanson]: https://github.com/JeffBezanson
[jamesonnash]: https://github.com/vtjnash
[bash_solition]: https://gist.github.com/amitchhajer/4461043
[pipe]: https://github.com/oxinabox/Pipe.jl
[underscores]: https://github.com/c42f/Underscores.jl
[env]: https://bkamins.github.io/julialang/2020/05/18/project-workflow.html
