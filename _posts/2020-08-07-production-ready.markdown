---
layout: post
title:  "JuliaCon2020: Julia is production ready!"
date:   2020-08-07 12:44:12 +0200
categories: julialang
---

# Introduction

Last time I have [posted][lastpost] about the take-aways for DataFrames.jl from
[JuliaCon 2020][juliacon]. This time I wanted to share my general conclusions
from attending different talks at this extremely successful event.

I have spent 20 years now deploying data science related projects in corporate
environments (back then it was not called data science, but we were already
training neural networks to make predictions) and have many colleagues who are
deeply into enterprise software development. Quoting a discussion with
Tomasz Olczak I had some time ago, who is a genuine one-man army in
delivering complex enterprise projects:

> Julia is fast, and has a very nice syntax, but its ecosystem is not mature
> enough for use in serious production projects.

For many years I would agree with it, but after [JuliaCon 2020][juliacon]
I believe we can confidently announce that

<p align="center" style="font-size:24px">
  <b><i>Julia is production ready!</i></b>
</p>

Let me now give a list of key (in my opinion) presentations given during
[JuliaCon 2020][juliacon] that make me draw this conclusion.

I will not comment here on functionalities related to number crunching, as it
is clear that Julia shines here, but rather I want to focus on the things that
make Julia a great tool for deployment in production (still I skip many
interesting talks in this area --- check out the detailed [agenda][agenda] to
learn more).

# Building microservices and applications

In [this talk][microservices] [Jacob Quinn][qj] gives an end to end tutorial how
to build and deploy in an enterprise setting a microservice using Julia.
He gives ready recipes how to solve typical tasks that need to be handled in
such contexts: logging, context management, middleware setup, authentication,
caching, connection pooling, dockerization, and many other, that
are bread and butter of enterprise projects.

As an addition be sure to check out:

* the [shippable apps][apps] talk, where [Kristoffer Carlsson][kc] guides you
  through creating executables which can be run on machines that do not have Julia
  installed.
* the [Julia for scripting][scripting] presentation, during which
  [Fredrik Ekre][fe] discusses the best practices for using Julia in contexts
  where you need to execute short code snippets many times.
* the [Genie.jl][genie] talk, in which [Adrian Salceanu][as] shows that it is currently
  a mature, stable, performant, and feature-rich Julia web development framework.

# Dependency management

The two talks [Pkg.update()][pkgupdate] and [What's new in Pkg][pkgnew] show that
currently Julia has best in class functionalities for enterprise grade dependency
management for your projects. The list of provided functionalities is so long
that it is hard to list them all here.

Let me just mention one particular tool in this ecosystem, that is presented in
[BinaryBuilder.jl][bb] talk that explains how to take software written in compiled
languages such as C, C++, Fortran, Go or Rust, and build precompiled artifacts
that can be used from Julia packages with ease (which means that no compilation
has to take place on client side when you install packages having such dependencies).

# Integration with external libraries

A natural topic related to dependency management is how to integrate Julia with
external tools. This area of functionality is really mature. Here is a list of
talks that cover this topic:
* [Make your Julia code faster and compatible with non-Julia code][nonJulia]
* [Wrapping a C++ library using CxxWrap.jl][wcpp]
* [Julia and C++: a technical overview of CxxWrap.jl][wcpp2]
* [Introducing the @ccall macro][ccall]
* [Integrating Julia in R with the JuliaConnectoR][rconn]
* [Integrate Julia and Javascript using Node.js extensions][nodejs]

Here it is worth to add Julia has had a great integration with Python for many
years now, see [JuliaPy][juliapy].

A good end to end example of doing some real work in Julia that requires integration
is [Creating a multichannel wireless speaker setup with Julia][wirelessspeaker]
talk that show how to easily stitch things together (and in particular featuring
ZMQ.jl, Opus.jl, PortAudio.jl, and DSP.jl).

Another interesting talk showcasing integration capabilities is
[JSServe: Websites & Dashboards in Julia][jsserve]
that shows a high performance framework to easily combine interactive plots,
markdown, widgets and plain HTML/Javascript in Jupyter / Atom / Nextjournal and
on websites.

# Developer tooling

The two great talks [Juno 1.0][juno] and [Using VS Code][vscode] that current IDE
support for Julia in VS Code is first class. You have all tools that normally you
would expect to get: code analysis (static and dynamic), debugger, workspaces,
integration with Jupyter Notebooks, and remote capabilities.

# Managing ML workflows

I do not want to cover many different ML algorithms that are available in Julia
natively, as there are just too many of them (and if something is missing you
can easily integrate it --- see the integration capabilities section above).

However, on top of particular models you need frameworks that let you manage
ML workflows. In this area there are two interesting talks, one about
[MLJ: a machine learning toolbox for Julia][mlj] and the other showing
[AutoMLPipeline: A ToolBox for Building ML Pipelines][amlp]. From my experience
such tools are crucial when you want to move with your ML models from data scientist's
sandbox to a real production usage.

# Conclusion

Obviously, I have omitted many interesting things that were shown during
[JuliaCon 2020][juliacon]. However, I hope that the aspects I have covered here,
that is:

* enterprise grade patterns to create microservices and applications in Julia,
* robust dependency management tools,
* very flexible and powerful capabilities to integrate Julia with existing code
  bases that were not written in Julia,
* excellent developer tooling in VSCode,
* mature packages that help you to create production-grade code for ML solutions
  deployment,

show that already now Julia can (and should) be considered as a serious option
for your next project in enterprise environment.

What I believe is crucially important is that not only we have required tools
ready but also we have great practical showcases how they can be used to build
robust production code with.

[lastpost]: https://bkamins.github.io/julialang/2020/08/02/post_juliacon_1.html
[juliacon]: https://juliacon.org/2020/
[microservices]: https://live.juliacon.org/talk/Y3H7FG
[qj]: https://github.com/quinnj
[apps]: https://live.juliacon.org/talk/Z8TE39
[kc]: https://github.com/KristofferC
[scripting]: https://live.juliacon.org/talk/N39HSX
[fe]: https://github.com/fredrikekre
[genie]: https://live.juliacon.org/talk/DTLBM9
[as]: https://github.com/essenciary
[agenda]: https://live.juliacon.org/agenda/
[juno]: https://live.juliacon.org/talk/DWCJUK
[vscode]: https://live.juliacon.org/talk/HBTFT7
[pkgupdate]: https://live.juliacon.org/talk/PLFURQ
[pkgnew]: https://live.juliacon.org/talk/RXAN8F
[bb]: https://live.juliacon.org/talk/9WMAVU
[wcpp]: https://live.juliacon.org/talk/NNVQQF
[nonJulia]: https://live.juliacon.org/talk/7MALUA
[rconn]: https://live.juliacon.org/talk/PHGCKB
[nodejs]: https://live.juliacon.org/talk/Q88P8U
[ccall]: https://live.juliacon.org/talk/Y7ERM9
[wcpp2]: https://live.juliacon.org/talk/XGHSWW
[juliapy]: https://github.com/JuliaPy
[wirelessspeaker]: https://live.juliacon.org/talk/KK9S9V
[jsserve]: https://live.juliacon.org/talk/GFE3DB
[mlj]: https://live.juliacon.org/talk/DMHZCC
[amlp]: https://live.juliacon.org/talk/AD7TQW
