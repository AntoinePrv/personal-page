+++
title = "Why Conda?"
date = "2020-08-29"
description = """Exploring what is the conda package manager and how it solves the binary
requirements of scientific software."""
+++


If you have written or used scientific software, especially in Python, you have probably been told
to use Anaconda without much explanations, only to find yourself frustrated by conda slowness and
inflexibility.
Why is that? And what is the difference between `pip install numpy` and `conda install numpy`
anyways?

Conda is a general purpose cross language package manager tailored to deal with the extensive binary
compatibility requirements of scientific software.
In this blog post, I will detail what those requirements are and what Conda is doing different as a
package manager to meet them.


## Some definitions
First let us define some terms you may have heard that I will be using in this post.
These terms are not written in stone, rather this is my own interpretation of what they usually
refer to.

Software package
: A set of files used to distribute a given version of a library or executable to its users.
  There exists many types, with different specifications telling tools what to find inside, such as
  version number, dependencies, data files, and of course the actual software.
  Using the Python naming conventions, a package is either:
    - A [*source*](https://packaging.python.org/glossary/#term-source-distribution-or-sdist) package
      if it contains source code and a script to install the package on the user's system (for
      instance the compilation instructions);
    - A [*built*](https://packaging.python.org/glossary/#term-built-distribution) package if the
      files only needs to be placed in the final install location (with no installation script).
      If a built package contains any compiled files, it is called a
      [*binary*](https://packaging.python.org/glossary/#term-binary-distribution) (built)
      package.[^built_package]

Package manager
: A program that download, install, and manage packages and their dependencies.
  It knows what packages are installed, where, how to remove them *etc*.

Pip
: The official Python package manager.
  By default it fetches packages from the Python Package Index [PyPI](https://pypi.org/).

Anaconda
: A Python distribution, that is an installation of Python with a currated list of packages.
  It is also the the name of the [company](https://www.anaconda.com/) who provide related services.

Conda
: The open-source package manager for its own language agnostic binary packages.
  It can also install Python packages.
  By default, it download packages from [Anaconda servers](https://anaconda.org/).


## Language Agnostic
Contrary to popular believe, conda is not (only) a Python package manager.
It can distribute all types of files and programs.
For instance, you might find there:
- Pure Python packages, such as the http library [Requests](https://anaconda.org/anaconda/requests);
- Python packages with compiled extensions, such as the array library
  [NumPy](https://anaconda.org/anaconda/numpy);
- `R` package, such as the [StringR](https://anaconda.org/r/r-stringr) package for common string
  operations;
- Shared (compiled) libraries, such as the optimized
  [OpenBLAS](https://anaconda.org/anaconda/openblas) basic linear algebra library;
- The [GCC](https://anaconda.org/anaconda/gcc_linux-64) compiler;
- The [CMake](https://anaconda.org/anaconda/cmake) build tool for C and C++ (in particular);
- Other sort of programs, such as the [Git](https://anaconda.org/anaconda/git) version control
  system.

Scientific software is growing extremly complex.
Scientists need high performance code and the ability to explore idea fast in high level languages
such as Python.
However, scientists cannot spend their time rewritting high performing basic operations constantly,
nor should they have to.
We rely on software that has been carefully developped tested and optimized by thousands of experts.
For instance, [scikit-learn](https://scikit-learn.org), the go-to Python package for implementing
machine learning models, relies on [SciPy](https://www.scipy.org), which relies on
[NumPy](https://numpy.org), which in turn rely on C anf Fortran libraries such as
[LAPACK](https://www.netlib.org/lapack/),
[BLAS](https://en.wikipedia.org/wiki/Basic_Linear_Algebra_Subprograms), and
[QuadMath](https://gcc.gnu.org/onlinedocs/libquadmath/).
This is only an overview, and the details are even more complicated.
For instance scikit-learn also depends on NumPy and SciPy depnds on some the same shared libraries
as NumPy.

Fortunaltely, we do not need to know all of this.
We use scikit-learn out of the box and it works great!
With Conda, the fact that one can upload package of any kind means developper can depend directly on
shared libraries such as BLAS.
Shared (compiled) libraries are not relics of the past that need to be migrated to something else.
They need to exist for many reason, such as performance, and hardware compatibility.
For example, on Intel processors, a dedicted BLAS implementation called
[MKL](https://software.intel.com/content/www/us/en/develop/tools/math-kernel-library.html) is often
used.
Another example is that of Graphic Processing Units (GPU) getting traction in scientific computing
(including, but not limited to, deep learning).
The GPUs has its own set of shared libraries that is need to function, and more to function
efficiently.
Doing Deep Learning on an Nvidia GPU, one would typically indirectly need
[CUDA](https://developer.nvidia.com/cuda-zone) and [CuDNN](https://developer.nvidia.com/cudnn).

In summary, scientific software is hybrid in nature: high level interfaces are needed to explore
ideas unconstrained and with the concepts that the community has already established, low level
libraries are needed to get the best computing efficiency out of our hardware.
Conda solve the problem of hybrid dependencies by serving language agnostic packages that can depend
on one another.

As a perk, if you also need some utilities tools that you would normally get on system package
managers (_e.g._ `apt-get`) and like conda, you can also get them from there.
I personally get `git`, `htop`, `fzf`, `ripgrep` `fswatch` (and more) on the
[Conda-Forge](https://conda-forge.org/) channel.

One question remains.
_If NumPy relies on shared libraries such as BLAS, how can one `pip install numpy` and still get a
fully working package?_
Short answer, they are distributed inside the NumPy package.
In the next section, we will understand how, and the different approach taken by conda.


## Distributing Binary packages
To understand the challenges that come with distributing binary packages, we will study Python
packages.
While the solutions used in Python might not be the same in other languages, the complications
remains the same.
There are multiple formats supported by pip.
For clarity, I will keep the terms used so far and link to the Python specific definitions.

### Compiling source packages
Source packages ([sdist](https://packaging.python.org/glossary/#term-source-distribution-or-sdist))
contain the source code and installation instructions (_e.g._ the [setup script]() and compilation
instruction for extensions and shared libraries).
It is roughly a striped down version of the code in the library repository, archived and compressed.

Relying soley on this format is not feasible.
It would mean that upon installing our usual packages we might get hours of compilation.
This may reasonable at first, as we think that we will install the packages only once, but in
practice, we reinstall packages every time we update, every time we use a new environment/computer,
and everytime we mess everything up and need a fresh start.
Furthermore, not everyone has a compiler.
There is most certainly one in our personal computers, but many minimal systems do not.
Even if a compiler is present, it might not be properly supported by in the package source code, the
computer may not have sufficient capabilities to compile and optimized high demanding libraries.
Finally it is a huge waste of energy.
A package like NumPy is downloaded
[over one millon times per day](https://pypistats.org/packages/numpy).
Imagine recompiling it and all the libraries it depends on so many times, when compiling a few
binary packages could serve all these users.

### Binary wheel packages
Another format that pip can install are the [wheel](https://packaging.python.org/glossary/#term-wheel)
(a [bdist](https://packaging.python.org/glossary/#term-built-distribution)), only needs for the
files it contained to be copied to the installation location.
It is of relevance to us because it is binary (compiled) package format, that is, it contains the
metadata for pip to select the correct version for the CPU and operating system.

#### OS tag
When compiling a library, the resulting binary cannot run on any computer.
At minimum, it is compiled for a given CPU architecture, and a given operating system.
It it then natural to wonder _how can wheel packages support users with different CPU and OS?_
The package itself only support what it has been compiled for, but we can compile it multiple time
to create a binary wheel package for every case.
Let us use the public [PyPI API](https://warehouse.pypa.io/) to look the packages uploaded for the
release `1.19.0` of NumPy.
If your browser display the resulting JSON nicely, you can open the
[URL](https://pypi.org/pypi/numpy/1.19.0/json), otherwise, using the command line, you can execute
```bash
curl -s https://pypi.org/pypi/numpy/1.19.0/json | python -m json.tool
```
Under the sectionn `urls`, we see 25 uploads.
The first one is
```json
{
  "info": {...},
  "last_serial": 7524196,
  "releases": {...},
  "urls": [
    {
      ...
      "filename": "numpy-1.19.0-cp36-cp36m-macosx_10_9_x86_64.whl",
      ...
    },
    ...
  ]
```
This file is a wheel package (`.whl`), for MacOS (`macosx_10_9`) with
[x86_64](https://en.wikipedia.org/wiki/X86-64) CPU architecture.
We also find `numpy-1.19.0-cp37-cp37m-manylinux2014_aarch64.whl` for Linux with
[ARM64](https://en.wikipedia.org/wiki/AArch64) CPU, and finally the source distribution package
`numpy-1.19.0.zip` in case no appropriate wheel can be found for the given user.

We will now manually download the first one (given the url given in the JSON) to inspect it.
```bash
wget https://files.pythonhosted.org/packages/d0/6f/26dd9a193c7292954c0afdaa9ed7c0f2595505388b7bd104dbe04669087a/numpy-1.19.0-cp36-cp36m-macosx_10_9_x86_64.whl
```
A wheel is actually a compressed archived (usually named `.tgz` or `.tar.gz`) of files ready to be
placed at the install location.
Let us extract it to look inside.
```shell
tar -xzf numpy-1.19.0-cp36-cp36m-macosx_10_9_x86_64.whl
```
There are many files, and the objective is not detail the NumPy package internals but rather to
notice the presence of certain types of files.
The file `numpy/random/_mt19937.cpython-36m-darwin.so` is a (compiled) shared library for a
Python extension that, I assume by its name, implements the
[Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister) pseudorandom number generator.
The file `numpy/.dylibs/libopenblas.0.dylib` is the OpenBLAS library (`.dylib` are similar to `.so`
on MacOS), added by the NumPy developper to face the limitations regarding non Python dependencies.
[^vendor]

#### ABI tag
So far, we have seen that for binary packages such as Python wheels, we need to build packages for
every configuration of the users, such as CPU architecture and operating system.
In the NumPy 1.19.0 wheels, we can also observe an additional tags like `cp36m` and `cp37m`.
These are telling us some sort of compatibility with Python 3.6 and Python 3.7.
It may not be suprising to see a Python version appearing somewhere, after all different versions
support different features.
However, this is _much more suttle_ than it looks.
This part is the _Python Application Binary Interface_ (ABI) tag.
The `cp36m` tag is _not_ claiming that NumPy 1.19.0 is compatible only with Python 3.6.
In fact, it is compatible with all current versions greater than 3.6.
It is giving a restriction to _this particular wheel_ stating that it has been compiled for
CPython 3.6 with the [Pymalloc](https://docs.python.org/2.3/whatsnew/section-pymalloc.html) option.

_Python what?_
Yes I wrote _CPython_, not _Python_ or _Cython_.
Python is the code we write.
For most users gets interpreted (_i.e._ executed) by another program written in C, called _CPython_.
It the reference and original implementation of Python, but anyone is free to create other
implementations.
There are quite a few! Some, like [Jython](https://www.jython.org/) and
[IronPython](https://ironpython.net/) aim to support different extension language (Java and .Net
respectively), while others, like [PyPy](https://www.pypy.org/) bring other new features (a just in
time compiler in this case).
When writting an compiled extension, as it is the case in NumPy, we need somthing to communicate
with the Python interpreter.
In this case, this is the [CPython C library](https://docs.python.org/3/c-api/index.html), a library
of C functions that can manipulate Python objects.
It is code like this that Python extensions tools like [Cython](https://cython.readthedocs.io) or
[Pybind11](https://pybind11.readthedocs.io) are generating.
When running NumPy, some of this function are calling CPython C functions, that are resolved in a
shared library and this is where ABI compatibility arise.
This idea is that the Python library used to compile the NumPy Python extension must be _binary_
compatible with the one used at runtime.
This has to do with conventions about the way data is represented and functions are called by the
CPU.
It is highly complex and can be impacted by many things that are outside the scope of this blog
post.
What is important to understand is that is really has to do the binaries.
The NumPy Cython code and the C code generated by Cython might very well be unchanged in the
when comling aginst different Python librairies.

As developpers of packages, we can ignore most of these details, and consider it the same way we
considered the OS and CPU architecture: the source code is left unchanged, yet we need to recompile
the wheel packages against all OS, CPU architecture, and now Python libraries, that we wish to
support.
Python and the packaging tools are keeping track of ABI compatibility, so all we need to do is
execute again the same commands with different version of Python.

### Conda packages and ABI
In the previous section on binary wheels, we have explained that when we want to distribute an
extension as binaries, we need to compile it for each combinaison of OS, CPU architecture, and
Python version that we want to support.

ABI is not something specific to Python, but somehting that arise when compiling against any
shared (dynamic) library.
It is a very complicated topic, so I have written [another post]({{< relref "2-abi-break.md" >}})
to convey intuition about it.
When we build NumPy 1.19.0 against one of its library dependency, _e.g._ BLAS version 0.3.10, the
binary package produced becomes only compatible with BLAS versions that are _binary compatible_
(ABI) with the version 0.3.10, even if NumPy _source code_ might[^numpy_blas_api] be (API)
compatible with all versions.
NumPy needs to be compiled for each range of ABI compatible versions of BLAS that it wish to
support, in the same way that is needs to be compiled for each version of Python.

So while we could use wheels to package a BLAS library (even if there is no Python code), there are
no way of stating for which ABI of BLAS, a given wheel of NumPy is build for.
What NumPy could do is pin the BLAS dependency to something like `blas>=0.3.10,<0.4.0` to ensure ABI
compatibility.[^numpy_blas_api]
But now, let's say SciPy, that also uses BLAS, is built for `blas>=0.2.1,<0.3.0`, then NumPy and
SciPy become incompatible!
And all this even if, prior to compiling, NumPy was (in our example) comaptible with all BLAS
versions.
Trying to coordinate the BLAS dependency for all packages would not only be massive work, it would
be futile and break eventually.

What we really need, is to be able to upload multiple NumPy wheels for different version of BLAS.
Something like
```text
numpy-1.19.0-cp36-cp36m-macosx_10_9_x86_64-blas_0.2.whl
numpy-1.19.0-cp36-cp36m-macosx_10_9_x86_64-blas_0.3.whl
...
```

This is not an easy.
First, we would have to generalize this to all shared library dependencies, not only BLAS.
Second, we need to know for each library what are the ABI compatible version ranges.
Here we have assumed all 0.2.x are binary compatible between them, similarily for all 0.3.x _etc_.
but this is oversimplified.

This is the challenge that Conda has taken upon itself: generalizing the ABI tag to all shared
library dependencies.
When compiling a Conda package, developers have to specify not only the libraries they depend on,
but also their ABI stable version range, and the matrix of OS, CPU architecture, and libraries
(including Python) that they widh to build for.


## Conda Environments
As a consequence, Conda must ensure between that all packages installed have a compatible versions,
but also a compatible ABI.
Conda needs to choose for each installed package, not only the version, but also the build.
On the contrary, with Pip, for each package version, only one build is appropriate for the current
user.
The problem for Conda becomes large and hard to solve, with potentially hard to track bug if
ABI incompatibilities are introduced.
The [approach taken by Conda](https://web.archive.org/web/20130925061157/https://continuum.io/blog/new-advances-in-conda)
is to model the dependencies versions selction as a
[SAT](https://en.wikipedia.org/wiki/Boolean_satisfiability_problem) and use a solver to find
compatible packages.

To ensure that a solution exsists and that it is found quickly, a best practice is to make use of
[environments](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html).
Environments are a way to install a set of packages together independently of other environemnts.
This let the users install incompatible versions of a libraries in different projects.
This concept exist in many package manager, including for Pip with Python's
[virtual environments](https://docs.python.org/3/library/venv.html).
With Conda, it is even more critical to do so to keep the solving time reasonable.

A good rule of thumbs is to do one environemnt per project, for which you install all the
dependencies needed to work on the project (neither less nor more).


## Other alternatives for Python
### Vendoring
Including shared library dependencies inside the package is what is currently being done on PyPI.
NumPy vendors BLAS, LAPACK, *etc*. PyTorch also vendors BLAS, and CUDA.
This is a pretty heavy solution that replicate libraries on file and in memory.
It also is a non trivial solution that requires susbtantial work from developpers to make sure the
library is properly found and loaded by the dynamic loader.
It is not well documented and requires much knowledge about shared libraries and Python packages.

A slightly similar alternative is to would be to vendor libraries as static libraries, that is
included in the binary of the Python extension.
Given that we can no longer find the library files in package, it is hard to know how much this is
done.
Changing a static library requires to recompile the whole extension, so this is also a downside to
consider.

### Dependencies pinning
This is the alternative presented earlier where a dynamic library such as BLAS would be wrapped in a
Python package.
A given version of NumPy would then pin as a requirement the specific version of BLAS it was build
for.
Unless there are very few libraries, this approach would very soon lead to unsatisfiable dependency
requirements.
As with the previous alternative, it also requires substantial technical work.

### Conan?
[Conan](https://conan.io/) is a C/C++ package manager.
As such, it is able to deal with all the complication of distributing binaries, including
everything related to ABI compatibilities.

Conan happens to be written in Python, so there is an opportunitiy to leverage it inside Python
packages to resolve the C/C++ libraries!
This is what the authors realized in their blog post
"*[Conan, a C, C++ and Python package manager](https://blog.conan.io/2016/12/12/Conan-python-package-manager.html)*"
where they upload a binary Python package to Conan servers.

The solution is still incomplete in my opinion, as the package are not managed by Pip, so it is
tricky to make sure the package is visible to Python, and it also not possible for other PyPI
package to use it as a dependency.

I believe there is a great product for Python packages to be built with Conan, but right now, I am
still learning to use Conan in its intended way.


There is much more that could be added all along this blog post, but it is already quite long and
technical.
For sure, no one holds the perfect solution to package management, and tools are constantly
improving.

In this post, we observed that scientific software is complex and depends on an heterogeneous
sets of libraries, some distributed as binaries.
Then, we analysed how binaries have requirements between them stronger than there API compatibility.
It's their ABI compatibility.
Finally, we learned how Conda is managing ABI  compatibility of varaious libraries by tracking their
ABI stability and compiling packages against multiple version of their library dependencies.



[^built_package]: In a compiled language, such as C++, all built packages would be binary because we
  cannot run C++ without compiling it.
  For a Python *only* library, a built pacakge cannot be binary because Python is interpreted
  (*i.e.* the source code is read directly and executed by an interpreter).
  In this case, there is really not much an installation script should do, and it built pacakges
  should be prefered.
  However, Python supports compiled extensions from other languages, so there also exists binary
  packages.

[^vendor]: We sometimes say that we _vendor_ a given dependency to mean that it is included with the
  current library, instead of expecting a package manager or the user to install it.

[^numpy_blas_api]: I do not know if this is the case, but it does not matter for the sake of this
  example.
