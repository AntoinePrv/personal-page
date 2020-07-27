+++
title = "Minimal ABI Example"
date = "2020-07-27"
description = """
    A small example example illustrating what is an ABI and why it is so hard to maintain.
  """
+++


In compiled languages, such as C++, developpers have the possibility to use and build
[_shared libraries_](https://en.wikipedia.org/wiki/Library_(computing)#Shared_libraries), also known
as a _dynamic libraries_.
To depend on a shared library (in an exectuable or another shared library), said library needs
to be present at compile time, [^link-time] but really symbols from the library (functions, types,
global variables _etc_.) are resolved at runtime (when something is executed) by the
[dynamic linker](https://en.wikipedia.org/wiki/Dynamic_linker).

Shared libraries have many advantages.
They can be _shared_ between multiple programs, reducing memory and disk usage.
Because they are resolved at runtime, they can be changed without impacting any program that depend
on them.
This is convenient to deliver updates to user without requiring much tools on their machine (which
can be hard to get right).
It also means we can wait until program execution to choose the most appropriate library.
For instance, a library such as [NumPy](https://numpy.org) is compiled against
[BLAS](https://en.wikipedia.org/wiki/Basic_Linear_Algebra_Subprograms).
BLAS has many implementations, for instance [OpenBLAS](https://www.openblas.net/),
[ATLAS](http://math-atlas.sourceforge.net/), [BLIS](https://github.com/flame/blis), that we can
choose from.
One of them, [Intel MKL](https://software.intel.com/mkl), is built specificly for Intel CPUs, and we
can therefore expect it to be more efficient on these machines.
But for NumPy, it does not change anything.
It only needs to compile against _a_ BLAS library, and it will work with _all_ BLAS libraries.

_All of them? What if we take something completly different and called it BLAS?_\
As you can expect that will not work.
Some information (_e.g._ function names) have been looked up when compiling against the shared
library, and this information need to be the same in the library used at runtime.
This is kown as the library interface: the observable boundary of componnents (functions, classes,
_etc_.) that can be used to interact with the library.
In non compiled languages (_e.g._ Python), it is the same as the
[_Application Programming Interface_](https://en.wikipedia.org/wiki/Application_programming_interface)
(API), _i.e._ the way we (humans) can write code using a library.
For shared library however, there also exsits an
[_Application Binary Interface_](https://en.wikipedia.org/wiki/Application_binary_interface) to
define the way computers can interact with it (in binary form).
Understanding ABI is highly technical.
It can change even if the shared library API (or even all of it source code) remains
unchanged.[^abi-break]
ABI is what make a shared library compatible with another.
The first thing that impacts ABI is the CPU architecture.
If we build a library for a given type, say [x86-64](https://en.wikipedia.org/wiki/X86-64), we
cannot expect it to be interchangable with another like
[ARM64](https://en.wikipedia.org/wiki/AArch64).

If we want to update a library without having to recompile its dependent programs, the ABI has to
remain stable.
This is why in this post we will build our intuition by studying an example that breaks the ABI.

_Should you care about ABI stability in your library?_\
If you are wondering this, then chances are the answer is no.
Libraries that are ABI stable across many versions and implementations are usually cornerstones of
our computing ecosystem and require dramatically more work and knowledge than what is in this post.
With this article, you will however better understand the struggle of package managers.


## libcomplex 0.1.0 - Hello world
Complex numbers are at the foundation of many algorithms and applications, so we have decided to put
much of our knowledge into a library `libcomplex` so that other developpers can build it.
Watch out for bugs, it is still in in beta! 😉

We build a class to represent complex numbers and put it in our `include/complex.hpp` header.
It uses the carthesian representation, but can compute a complex's modulus and argument.
We also added a `to_string` method to get human readable representation.

```cpp
#pragma once

#include <ostream>


struct Complexe {
  Complexe(double real, double imaginary) noexcept :
    real(real), imaginary(imaginary) {}

  double modulus() const noexcept;
  double argument() const noexcept;

  friend std::ostream& operator<<(std::ostream& out, const Complex& c);

private:
  double imaginary;
  double real;
};
```

And we put the implementation of the method in the associated `src/complex.cpp` file

```cpp
#include <cmath>

#include "complex.hpp"


double Complexe::modulus() const noexcept {
  return std::sqrt(real*real + imaginary*imaginary);
}

double Complexe::argument() const noexcept {
  if(modulus() == 0.){
    return std::nan("");
  } else if(imaginary < 0.){
    return - std::asin(real / modulus());
  }
  return std::asin(real / modulus());
}

std::ostream& operator<<(std::ostream& out, const Complex& c) {
  out << c.real << " + " << c.imaginary << 'i';
  out << " = " << c.modulus() << "e^(" << c.argument() << " i)";
  return out;
}
```

Our library is an overnight success, and many projects use it as a dependency!
One of them, `awesome-app`, has the following code

```cpp
#include <iostream>
#include <string>

#include "complex.hpp"


int main(int argc, char** argv) {
  const Complex z{std::stod(argv[1]), std::stod(argv[2])};
  std::cout << z << '\n';
}
```

Because compiling a library can be tricky to get right, we write a short [CMake](https://cmake.org/)
file to do it.
For simplicity, we will add `awesome-app` to the same file, but in practice this would be two
separate projects.

```cmake
cmake_minimum_required(VERSION 3.0)

project(Complex VERSION 0.1.0 LANGUAGES CXX DESCRIPTION "Complex number library")

add_library(complex SHARED src/complex.cpp)
target_include_directories(complex PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_compile_features(complex PUBLIC cxx_std_14)

add_executable(awesome-app src/awesome-app.cpp)
target_link_libraries(awesome-app PRIVATE complex)
```

And now to compile and run all
```bash
cmake -B build
cmake --build build
./build/awesome-app 3 4
```
Yields
```
3 + 4i = 5e^(0.643501 i)
```


## libcomplex 0.2.0 - Pardon my French
Too excited about our library, we named our class `Complexe` in French instead of the English
`Complex`.
We make the change and try to recompile only our library (_i.e._ not `awesome-app`).
```bash
cmake --build build --target complex
```
Now if our dependency `awesome-app` updates the library without recompiling
```bash
./build/awesome-app 3 4
```
It will get the error similar to (this is on MacOS)
```
dyld: Symbol not found: __ZlsRNSt3__113basic_ostreamIcNS_11char_traitsIcEEEERK8Complexe
  Referenced from: ./build/awesome-app
  Expected in: ./build/libcomplex.dylib
```
This means that the dynamic loader cannot find the (binary) function to print a `Complexe` (the
names in the output are [mangled](https://en.wikipedia.org/wiki/Name_mangling)).
Indeed, when recompling the library we replaced the function to print a `Complexe` with a function
to print a `Complex`.
The ABI is no longer compatible.
`awseome-app` needs to be recompiled
```bash
cmake --build build --target awesome-app
```
And this time we have the compilation error
```
./src/awesome-app.cpp:8:8: error:
      unknown type name 'Complexe'; did you mean 'Complex'?
        const Complexe z{std::stod(argv[1]), std::sto...
              ^~~~~~~~
              Complex
```
Indeed, the name has changed.
The API has also been modified and `awesome-app` needs to modify its code to replace `Complexe`
with `Complex`.

Changing the API will always change the ABI so evaluating if the ABI was stable here was a lost
cause anyways.


## libcomplex 0.2.1 - Bug fix
In our enthusiasm, we made a mistake in the formula to compute a complex's argument.
We used `asin` instead of `acos`.
To be fair, there is an equivalent formula using `asin`, and also one using `atan`.
There is actually a function in the standard library to compute directly this angle:
[`std::atan2`](https://en.cppreference.com/w/cpp/numeric/math/atan2).[^atan2]
Because it is made just for this, we can expect it to be numerically more accurate.
It also reduce the amount of code we have to maintain and test, so we make the switch.

```cpp
double Complex::argument() const noexcept {
  return std::atan2(imaginary, real);
}
```

If we recompile only the library
```bash
cmake --build build --target complex
```
And use it directly without recompiling `awesome-app`,
```bash
./build/awesome-app 3 4
```
We get
```
3 + 4i = 5e^(0.927295 i)
```
The ABI remains unchanged so we are able to deliver a bugfix to our users (and the users of
`awesome-app`) by simply swaping in the new library.
This could happen without any action from the developpers of `awesome-app`!


## libcomplex 0.3.0 - ABI break
While putting some documentation in our code, we notice the following attributes in the `Complex`
class

```cpp
struct Complex {
  ...
private:
  double imaginary;
  double real;
};
```

Complex numbers are usually represented with their real part first, so we would like our code to
look the same.
We inverse the attributes

```cpp
struct Complex {
  ...
private:
  double real;
  double imaginary;
};
```

These are private attributes.
They cannot be accessed outside of the class, so our API has not changed.
If we try again to only swap the new library

```bash
cmake --build build --target complex
```
And use it directly without recompiling `awesome-app`,
```bash
./build/awesome-app 3 4
```
This time, we get
```
4 + 3i = 5e^(0.643501 i)
```
Notice how `3` and `4` (and the phase) have been swapped, just like we swapped the attributes in the
class?
This happens because when the `complex.hpp` header get included in `awesome-app.cpp`, some code for
`Complex` gets generated in `awesome-app`.
In particluar, the compiler aligns all its attributes (like if it was a tuple) and loses the notion
of name.
The constructor of `Complex` is defined in the headers, so when the compiler compiles `awesome-app`,
it puts the imaginary part first and the real part afterwards.
When we use a function from the updated library, the library expects on the contrary to find the
real attribute first and the imaginary second, hence the flip.

We made a modification that left the API unchanged but changed the ABI.
This is a particluarily nasty one, because it does not give any errors.
The program is completly well behaved for computers but it does not compute what we expect it.
As before, the solution is to recompile `awesome-app`.

The topic of ABI stability is extremely complex.
For instance, renaming `imaginary` to any other name would have preserved the ABI.
Similarily, if the constructor was only _declared_ in `include/complex.hpp`

```cpp
struct Complex {
  Complex(double real, double imaginary) noexcept;
  ...
};
```

And _defined_ in `src/complex.cpp`

```cpp
Complex::Complex(double real, double imaginary) noexcept :
  real(real), imaginary(imaginary) {}
```

Then, _in this limited example_, we could swap the attributes without changing the ABI.


I hope this example helped you understand how and when package managers can decide to replace a
shared library for another, and hopefully it will help you figure out related bugs in the future.
This post was meant to be illustrative and is not an incitation to develop ABI stable library.
In particular, the examples in this post may behave differently in a more complete library.


[^link-time]: [Link](https://en.wikipedia.org/wiki/Linker_(computing)) time to be precise.
[^abi-break]: For instance,
  [GCC changed](https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html) its C++
  standard library ABI in GCC 5.
  Therefore the same shared library compiled with GCC < 5 and GCC >= 5 may not be ABI compatible.
[^atan2]: Ok, there is also a [`std::complex`](https://en.cppreference.com/w/cpp/numeric/complex)
  type, but 🤫.
