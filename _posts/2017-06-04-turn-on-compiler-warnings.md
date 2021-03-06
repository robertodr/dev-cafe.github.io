---
layout: post
title: Turn On Compiler Warnings!
author: Roberto Di Remigio
summary: Compilers save lives
---

Compiler warnings can be a nuisance, clogging the output with unnecessarily
verbose information.
This is especially true of C++, even more so when the code uses template magic.
However, turning them off can be rather harmful.

### TL;DR

Set `-Wall -Wextra`, or equivalent, as your basic compiler flags, both in debug
and in release mode.
If you are using CMake, the [Autocmake
project](http://autocmake.readthedocs.io/en/latest/) does it [by
default](https://github.com/coderefinery/autocmake/pull/203).

## An example from the [`cnpy` library](https://github.com/robertodr/cnpy)

Consider the following C++11 example, but it applies equally well to earlier
standards.
The
[`cnpy` library](https://github.com/robertodr/cnpy) for saving/loading C++ arrays
to/from NumPy binary format has a basic data structure, called `NpyArray`.
An object of this type is constructed by supplying:

- the shape of the array, i.e. a `std::vector<size_t>`. For example, a
  2-dimensional array with dimensions 10 and 11 will have: `std::vector<size_t>
  shape({10, 11});`
- the _word size_ of the data type to be dumped to NumPy array. This is
  either read from the `.npy` when loading the array, or determined by the
  result of `sizeof(T)`, where `T` is the data type, when saving the array.

The constructor will then compute how large a memory buffer is needed and
allocate a `std::vector<char>`.
The number of values in the array is computed from the shape array:
{% highlight c++ %}
size_t nValues = 1;
for (size_t i = 0; i < shape.size(); ++i)
  nValues *= shape[i];
{% endhighlight %}
or more compactly using `std::accumulate`:
{% highlight c++ %}
nValues = std::accumulate(shape.begin(), shape.end(),
                          1, std::multiplies<size_t>());
{% endhighlight %}
The type information is encoded in the `.npy`
file format header. When loading the array the user will have to perform a
`reinterpret_cast` to get the correct data type.

## Runtime error!

Stripped to its barebones, the `NpyArray` class looks like this:
{% highlight c++ %}
    {% include _snippets/wrong-vector_wtf.cpp %}
{% endhighlight %}
Let's now try to compile it:
{% highlight bash %}
g++ -std=c++11 -O0 main.cpp && ./a.out
{% endhighlight %}
The live example on [Coliru](http://coliru.stacked-crooked.com/a/bcf64023319e2f5e) shows that we get a runtime error
because the assertion in the constructor fails.
{% highlight bash %}
nValues_ 110
w 16
nValues_ * w 1760
a.out: main.cpp:18: VectorWtf::VectorWtf(const std::vector<long unsigned int>&, size_t): Assertion `buffer_.size() == nValues_ * w' failed.
bash: line 7: 13432 Aborted                 (core dumped) ./a.out
{% endhighlight %}
What's happening? Well, a very, very stupid mistake. **The `buffer_` data member is initialized using the `nValues_` data member**
This shouldn't be a problem, since it's **initialized first** in the constructor, right?
Wrong! [According to the standard, 12.6.2 Section 13.3](http://open-std.org/JTC1/SC22/WG21/docs/papers/2016/n4594.pdf)
data members are initialized in the order they were declared in the class. Thus `buffer_` gets initialized first, using an undefined value for `nValues_`.

## Fixing it

The **correct** `struct` declaration is thus:
{% highlight c++ %}
    {% include _snippets/vector_wtf.cpp %}
{% endhighlight %}
which also honors the tenet of ordering data members in your classes and structs by their size in memory.
However, this is something very easily forgotten. How to avoid these kinds of errors?

1. **Do not initialize data members based on other data members**. This is, in
   my opinion, overly restrictive.
2. **Insert assertions in the constructors**. Very useful, but assertions only
   work when `-DNDEBUG` is not given to the compiler. Most of the times this is not the case
   when compiling with optimization.
3. **Turn on compiler warnings**. `-Wall` catches this mistake and many others. For an extra layer
   of warnings, I also turn on `-Wextra`. This is the [output on Coliru](http://coliru.stacked-crooked.com/a/8eecbafde77f1d23)
   {% highlight bash %}
   main.cpp: In constructor 'VectorWtf::VectorWtf(const std::vector<long unsigned int>&, size_t)':
   main.cpp:27:10: warning: 'VectorWtf::nValues_' will be initialized after [-Wreorder]
      size_t nValues_;
             ^~~~~~~~
   main.cpp:26:21: warning:   'std::vector<char> VectorWtf::buffer_' [-Wreorder]
      std::vector<char> buffer_;
                        ^~~~~~~
   main.cpp:9:3: warning:   when initialized here [-Wreorder]
      VectorWtf(const std::vector<size_t> & s, size_t w) :
      ^~~~~~~~~
   {% endhighlight %}
