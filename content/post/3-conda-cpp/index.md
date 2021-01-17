+++
title = "C++ Development With Conda"
date = "2021-01-17"
description = """An example of how to use Conda to get a full C/C++ development environment and
discussing the advantages and drawbacks of using this thechnique.
We suggest a generalization to other languages as well as highlight a bright future for Conda."""
+++

[Conda](https://conda.io) is a general purpose package manager widely used in the scientific
community.
We have shown in [another post]({{< relref "1-why-conda.md" >}}) how it can install many assets,
including managing binary compatibility.
[Conda-Forge](https://conda-forge.org/) is a community driven Conda channel[^channel] and set of
tools to build and update packages.
Together, they make a wide variety of packages available that we can leverage for C and C++
development.
Without further ado, let us have a look at some of them.

## Useful Packages
Package managers usually focus on delivering the libraries that we use in our project, but there are
actually many more tools that we use in our development.
They are needed at different time and for different purposes (non exclusively):
 - to build (compile) our project,
 - to run our project (our executable our library),
 - to test our project,
 - to benchmark our project,
 - to generate the documentation for our project,
 - to perform quality analysis of our project (_e.g._ code check and formatting).

Following are useful packages, _all available on Conda-Forge_, that we may need to work on our C/C++
project.
  - A C, C++, and Fortran compiler, along with their standard libraries, and binary toolchain
    (linker, archiver...).
  - Various build tools, such as [CMake](https://cmake.org/),
    [Make](https://www.gnu.org/software/make/), [Ninja](https://ninja-build.org/),
    and [Bazel](https://bazel.build/).
  - [CCache](https://ccache.dev/) and [SCCache](https://github.com/mozilla/sccache) compiler caches.
  - Header only libraries, such as [Xtensor](https://xtensor.readthedocs.io), or
    dynamic[^conda-dynamic] libraries such as [TBB](https://software.intel.com/en-us/oneapi/onetbb),
    multiple [BLAS](https://www.netlib.org/blas/) and
    [LAPACK](https://performance.netlib.org/lapack/) implementations,[^blas]
    [OpenMP](https://www.openmp.org), [GMP](https://gmplib.org), [Zlib](https://www.zlib.net/),
    [Boost](https://www.boost.org), and [Fmtlib](https://fmt.dev).
  - The [Conan](https://conan.io/) C/C++ package manager, if you need an additional source of
    libraries, or if you are working to make yours compatible with it.
  - Testing frameworks, such as [GoogleTest](https://github.com/google/googletest) and
    [Catch2](https://github.com/catchorg/Catch2).
  - Benchmark framework such as [Google Benchmark](https://github.com/google/benchmark).
  - [Doxygen](https://www.doxygen.nl), possibly with [Sphinx](https://www.sphinx-doc.org) and
    [Breathe](https://breathe.readthedocs.io) if you prefer that toolset.[^breathe] 
  - The [Clangd](https://clangd.llvm.org/)[^clang-tools] C/C++ language server for your editor.
  - [Clang-Tidy](https://clang.llvm.org/extra/clang-tidy/)[^clang-tools] and
    [CppCheck](http://cppcheck.sourceforge.net/) to verify code;
    [Clang-Format](https://clang.llvm.org/docs/ClangFormat.html)[^clang-tools] to format it.
  - [Pre-Commit](https://pre-commit.com/) to run check automatically. Notice that this is a Python
    package, but because Conda is a general purpose package manager, it will autmatically get a
    compatible Python interpreter without additional input.
  - Many more utilities, such as [Git](https://git-scm.com), [Hub](https://hub.github.com/),
    [Wget](https://www.gnu.org/software/wget/), [AWSCLI](https://aws.amazon.com/cli/).
  - Scripting langages, including [Python](https://www.python.org/), [Perl](https://www.perl.org),
    [Bash](https://www.gnu.org/software/bash/) if you have custom scripts.
  - Language binding libraries, such as [Cython](https://cython.readthedocs.io),
    [Pybind11](https://pybind11.readthedocs.io), [Rcpp](http://rcpp.org/),
    and [LibCxxWrap](https://github.com/JuliaInterop/libcxxwrap-julia/).
  - The [Singularity](https://sylabs.io/singularity/) containerization solution.

If you are missing something, you can
[submit it](https://conda-forge.org/docs/maintainer/adding_pkgs.html) to Conda-Forge.
The process happens entirely on Github, and all the building infrastructure will be set up
automatically.


## Advantages and Drawbacks
Many of the packages mentioned above are redundant with what is already available on our computer or
easily installable with the OS package manager (`apt-get`, [HomeBrew](https://brew.sh),
[WinGet](https://github.com/microsoft/winget-cli), [Chocolatey](https://chocolatey.org/)...).
Why then choose Conda?

  - Conda is _cross platform_, which means you can easily share the package requirements with
    collaborators and contributors.
  - Conda provides _isolated environments_, which means you do not have to worry about conflicting
    dependencies in different projects (one environment per project is a good rule of thumb).
  - Conda is _reproducible_, with environment and lock files.
    You can share them with collaborators, use them to switch to another machine, or to rebuild
    your environment later on if you decide to free up some space.
  - Conda _does not need_ admin rights to install packages.
    This is useful if you (or you collaborators) need to work on a university or company server.
  - Over language specific package managers, Conda is _not limited_ to libraries or a single
    language.
  - Everything is pre-compiled, which means fast installtion and update times.[^slow]

However it is true that Conda has some downsides.
  - Conda can be slow to find compatible libraries.[^slow]
  - Conda is very picky about packages compatibility and can sometimes refuse to install
    packages,[^slow] all while giving cryptic error messages.
  - Conda environments can be hard to use at first.


## Example
We will be running the example inside the
[continuumio/miniconda3](https://hub.docker.com/r/continuumio/miniconda3) Docker image.
This image does not have much except the Miniconda installation.
In particular, there is no compiler, C/C++ standard library, or linker.

We run a shell inside the image with
```bash
docker run -it --rm continuumio/miniconda3
```
Then create our environment from the following `environment.yaml`.
It is a good practice to keep all our dependencies there, rather than installing them with
`conda install`, so that we can easily rebuild our environment if needed.

```yaml
channels:
  - conda-forge
dependencies:
  - cxx-compiler
  - make
  - cmake
  - boost-cpp
```
Finally, we create the environment and activate it.
```bash
conda env create --name myproject --file environment.yaml
conda activate --name myproject
```
We can now compile a simple C++ project and keep adding any other packages that we need.


## Note on the Compiler Packages
The packages `fortran-compiler`, `c-compiler`, and `cxx-compiler` set many environment variables to
control the compilation process.
Some, like `FC`, `CC`, `CXX`, `LD`, `AR` are necessary as they help tools like CMake find the
compilers and other utilities.
Other like `FFLAGS`, `CFLAGS`, `CXXFLAGS`, `CPPFLAGS`, and `LDFLAGS` control compilation options and
are also picked up by CMake.
These are the flags used by Conda when building the packages, so they are very reasonable.
However, they are not all well-suited for development.
For instance, even when configuring with `CMAKE_BUILD_TYPE=Debug`, CMake ends up compiling with the
`-O2` option. 

The first solution that we have is to not use the compilers packages but rather the one on our
machine if that is an option.

The second option is to leave it as it is.
The flags can make it a bit harder to debug, but they are not weakening the build in any way, on
the contrary.
If we are not using this environment for debugging (_e.g._ if this is used in continuous
integration, or to deploy on a university server), then we can use the flags as is.
It is also possible to unset these environment variables manually when we need to run the debugger.

Finally, we can adapt the way our build tool processes these environment varaible flags.
A CMake example is given in this toolchain file
[`conda-toolchain.cmake`](conda-toolchain.cmake).
It defines two addtional `CMAKE_BUILD_TYPE`, `CondaDebug` and `CondaRelease` that uses the Conda
flags, and reset the other build types to their default value.
We can use it wihtout modifying our source code:
```bash
cmake -B build CMAKE_TOOLCHAIN_FILE="path/to/conda-toolchain.cmake -D CMAKE_BUILD_TYPE=..."
```


## Other Languages
Is Conda still relevant for other languages? I belive so.
We can use a language's usual package manager to build our library, and still complete our
development environment with Conda to get all the other tools needed, including the language itself.

We can get [Rust](https://www.rust-lang.org), [Node.js](https://nodejs.org),
[Haskell](https://www.haskell.org), and [Julia](https://julialang.org)[^julia] on Conda-Forge.
Even for Python and C/C++ that have many libraries in Conda-Forge, we can also use them through
other package managers (Pip and [Conan](https://conan.io/)) also available on Conda-Forge.


## Looking in the Future: The Mamba Org
Many of the issues of Conda are being tackled in a community led project:
the [Mamba Organization](https://github.com/mamba-org).
It includes:

  - [Mamba](https://github.com/mamba-org/mamba), a fast replacement to Conda.
    It is also fully written in C++, which means that it can be installed very easily without
    Python (which Conda needs).
    Micromanba, a standalone distribution, is also under development as a Microconda replacement.
  - [Boa](https://github.com/mamba-org/boa), a replacement to `conda-build` the tool used to build
    Conda packages.
  - [Quetz](https://github.com/mamba-org/quetz), the first open-source server for hosting Conda
     packages.
     Up to now, hosting packages has been exclusively done by [Ananconda](https://anaconda.org).

All the details and acknowledgments are available in the
[release announcement](https://medium.com/@QuantStack/open-software-packaging-for-science-61cecee7fc23)

On top of making the Conda experience better for users, this also opens exciting possiblilies.
Mamba speed makes it a promising candidate to resolve dependencies in continuous integration.
This means that with a single requirement file, we can get the same dependencies in continuous
integration that we use for development, and this for all OS!

The open source Quetz server will make it possible to host Conda packages with fine-grained access,
including in private networks.
This is a possibility available with most major package managers, and which fits the more complex
needs of companies.


## Acknowledgements
I would like to thank Wolf Vollprecht, Chris Burr, and Isuru Fernando for helping me understand the
Conda-Forge compiler packages.


<!-- Footnotes -->

[^channel]: A collection of Conda packages.

[^conda-dynamic]: Conda encourages shipping dynamic libraries over static.

[^breathe]: Breathe connects Doxygen outputs to Sphinx.
  Check out [Xtensor](https://xtensor.readthedocs.io) for an example of the output.

[^slow]: Conda has strict requirements to manage (binary) packages compatibility, as explained in
  this [other post]({{< relref "1-why-conda.md" >}}).
  This can make it slow (or impossible) to find compatible versions of libraries.
  However, a package manager that would need to compile some libraries would take significantly more
  time.

[^julia]: At the time or writting Julia is stuck on version 1.1 in Conda-Forge, while the latest
  available is 1.5.
  This is due to a change in the Julia installation and is
  [being addressed](https://github.com/conda-forge/julia-feedstock/issues/81).

[^blas]: See the [knowledge base](https://conda-forge.org/docs/maintainer/knowledge_base.html#blas)
  to use the different libraries.

[^clang-tools]: In the `clang-tools` package.
