<!-- Hide h3+ from toc  -->
<style>.md-nav--secondary .md-nav__list .md-nav__list { display: none }</style>

# Compiling libigl as a static library

!!! warning
    Compiling libigl as a static library is considerably more difficult
    than using it as a header-only library (see [installation instruction](https://libigl.github.io/#installation) instead). Do
    it only if you are experienced with C++, cmake and make, and you want to
    improve your compilation times.

Libigl is developed most often on Mac OS X, though has current users in Linux and Windows.

### Linux/Mac OS X/Cygwin

Libigl may also be compiled to a static library. This is advantageous when
building a project with libigl, since when used as an header-only library can
slow down compile times.

To build the entire libigl library producing at least `lib/libigl.a` and
possible other (automatically detected) extras, e.g. `lib/libiglcgal.a` issue:

```bash
cmake ../ -DCMAKE_BUILD_TYPE=Release\
          -DLIBIGL_USE_STATIC_LIBRARY=ON\
          -DCMAKE_INSTALL_PREFIX=/path/to/custom/installation
```
```bash
make
make install
```
This will install libigl into the directory specified by `CMAKE_INSTALL_PREFIX`, which is set to `/usr/local` by deafault on macOS and Linux.
The installation process does not only create `${CMAKE_INSTALL_PREFIX}/lib/libigl.a`, it copies the libigl headers into `${CMAKE_INSTALL_PREFIX}/include/igl/` too.

Or if you base your project on the [libigl-example-project](https://github.com/libigl/libigl-example-project)
you can add

```cmake
option(LIBIGL_USE_STATIC_LIBRARY     "Use libIGL as static librarie" ON)
```

before the following line

```cmake
find_package(LIBIGL REQUIRED QUIET)
```

in the `CMakeLists.txt` to always build libigl as static library for your project.
See [example-project](./example-project.md) aswell.

!!! tip
    If you have changed the value for `LIBIGL_USE_STATIC_LIBRARY` in your CMakeLists.txt make sure to remove or update the **CMakeCache.txt** in your build directory.

!!! warning
    You should expect to see a few linker warnings of the form:

    ```bash
    /opt/local/bin/ranlib: file: libigl.a(*.cpp.o) has no symbols
    ```

    These are (admittedly unpopular) functions that have never been used by us
    statically so we haven't explicit instantiations (yet).


## Development
Further documentation for developers is listed in
[style-guidelines](./style-guidelines.md).

## License
See `LICENSE.txt`

## Zipping
Zip this directory without .git litter and binaries using:

```bash
git archive -prefix=libigl/ -o libigl.zip master
```

## Explicit instantiations of templated functions

Special care must be taken by the developers of each function and
class in the libigl library that uses C++ templates. If this function
is intended to be compiled into the statically linked libigl library
then function is only compiled for each *explicitly* instantiated
declaration. These should be added at the bottom of the corresponding
.cpp file surrounded by a

```cpp
#ifdef IGL_STATIC_LIBRARY
```

Of course, a developer may not know ahead of time which
instantiations should be explicitly included in the igl static lib.
One way to find out is to add one explicit instantiation for each
call in one's own project. This only ever needs to be done once for
each template.

The process is somewhat mechanical using a linker with reasonable error
output.

Supposed for example we have compiled the igl static lib, including the
cat.h and cat.cpp functions, without any explicit instantiation. Say
using the makefile in the `libigl` directory:

```bash
cd $LIBIGL
make
```

Now if we try to compile a project and link against it we may get
an error like:

```bash
Undefined symbols for architecture x86_64:
"Eigen::Matrix<int, -1, -1, 0, -1, -1> igl::cat<Eigen::Matrix<int, -1, -1, 0, -1, -1> >(int, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&)", referenced from:
uniform_sample(Eigen::Matrix<double, -1, -1, 0, -1, -1> const&, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&, int, double, Eigen::Matrix<double, -1, -1, 0, -1, -1>&)in Skinning.o
"Eigen::SparseMatrix<double, 0, int> igl::cat<Eigen::SparseMatrix<double, 0, int> >(int, Eigen::SparseMatrix<double, 0, int> const&, Eigen::SparseMatrix<double, 0, int> const&)", referenced from:
covariance_scatter_matrix(Eigen::Matrix<double, -1, -1, 0, -1, -1> const&, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&, ArapEnergy, Eigen::SparseMatrix<double, 0, int>&)in arap_dof.o
arap_rhs(Eigen::Matrix<double, -1, -1, 0, -1, -1> const&, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&, ArapEnergy, Eigen::SparseMatrix<double, 0, int>&)in arap_dof.o
```

This looks like a mess, but luckily we don't really need to read it
all. Just copy the first part in quotes

```cpp
Eigen::Matrix<int, -1, -1, 0, -1, -1> igl::cat<Eigen::Matrix<int, -1, -1, 0, -1, -1> >(int, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&)
```

, then append it
to the list of explicit template instantiations at the end of
`cat.cpp` after the word
**template** and followed by a semi-colon.
Like this:

```cpp
#ifdef IGL_STATIC_LIBRARY
// Explicit template instantiation
template Eigen::Matrix<int, -1, -1, 0, -1, -1> igl::cat<Eigen::Matrix<int, -1, -1, 0, -1, -1> >(int, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&, Eigen::Matrix<int, -1, -1, 0, -1, -1> const&);
#endif
```

Then you must recompile the IGL static library.

```bash
cd $LIBIGL
make
```

And try to compile your project again, potentially repeating this
process until no more symbols are undefined.

!!! note
    It may be useful to check that you code compiles with
    no errors first using the headers-only version to be sure that all errors are from missing template
    instantiations.

If you're using make then the following command will
reveal each missing symbol on its own line:

```bash
make 2>&1 | grep "referenced from" | sed -e "s/, referenced from.*//"
```

### Benefits of static library

- **Faster compile time**: Because the libigl library
    is already compiled, only the new code in ones project must be
    compiled and then linked to IGL. This means compile times are
    generally faster.

- **Debug or optimized**: The IGL static
    library may be compiled in debug mode or optimized release mode
    regardless of whether one's project is being optimized or
    debugged.

### Drawbacks of static library

- **Hard to use templates**: Special care (by the developers of the library) needs to be taken when exposing templated functions.
