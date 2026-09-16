---
title: "Fortran Guide"
date: "2026-09-16"
draft: false
weight: 30
---

Enzyme can be used in Fortran via its bindings provided in the
[`enzyme`](https://github.com/EnzymeAD/Enzyme/blob/main/enzyme/Fortran/enzyme.f90)
module.

## Note on compilers

Before providing details on the Fortran bindings, it is worth noting that Enzyme
only supports the `2023.0.0` and `2023.2.4` versions of the Intel
IFX Fortran compiler. We strongly recommend using the
[Flang](https://flang.llvm.org) compiler, which is available as part of the
[LLVM project](https://github.com/llvm/llvm-project).

## Running Enzyme from flang

Configuring Enzyme with `-DENZYME_FLANG=ON` builds `FlangEnzyme-<LLVM version>`,
a pass plugin that flang can load with `-fpass-plugin`. Enzyme then runs as part
of the flang optimization pipeline, so a single command differentiates and
compiles:

```console
$ flang -fpass-plugin=/path/to/FlangEnzyme-21.so -I /path/to/enzyme/modules program.f90 -o program
```

The `-I` flag points at the directory holding the `enzyme.mod` module file,
which is built by `-DENZYME_FORTRAN=ON` (see the sections below).

Without the plugin the derivative has to be produced out of line, by emitting
LLVM IR from flang and running the Enzyme pass over it with `opt`:

```console
$ flang -flto -c -I /path/to/enzyme/modules program.f90 -o program.bc
$ opt -load-pass-plugin=/path/to/LLVMEnzyme-21.so -passes=enzyme program.bc -o program-enzyme.bc
$ flang -flto program-enzyme.bc -o program
```

Both routes are exercised by the tests in
[`enzyme/test/Fortran`](https://github.com/EnzymeAD/Enzyme/tree/main/enzyme/test/Fortran).
The plugin route is flang-only; with ifx use the `opt` pipeline above.

### Using Fortran Package Manager (fpm)

Enzyme provides an
[`fpm.toml`](https://github.com/EnzymeAD/Enzyme/blob/main/fpm.toml) file,
allowing it to be easily integrated into projects that use the
[Fortran Package Manager (fpm)](https://fpm.fortran-lang.org/).
To make use of Enzyme via fpm, ensure the Flang compiler is in your `PATH`.

#### Flang plugin

To make use of Enzyme's Flang plugin, set the `ENZYME_PLUGIN` environment
variable to be the appropriate path, i.e.,
```sh
$ export ENZYME_PLUGIN=/path/to/Enzyme/FlangEnzyme-${VN}.so
```
where `${VN}` is the LLVM version you are using. Add the following to the
`fpm.toml` for your project:
```toml
[features]
enzyme.flang.flags = "-fpass-plugin=$ENZYME_PLUGIN"
enzyme.preprocess.cpp.macros = ["HAS_ENZYME"]
```
If you are on a non-Unix system then you may need to hard-code `ENZYME_PLUGIN`
rather than using an environment variable.

Note that this approach will only work if you have a single source file. If your
project contains multiple Fortran files then use the following approach.

#### LLD plugin

To make use of Enzyme's LLD plugin, set the `ENZYME_LLD_PLUGIN` environment
variable to be the appropriate path, i.e.,
```sh
$ export ENZYME_LLD_PLUGIN=/path/to/Enzyme/LLDEnzyme-${VN}.so
```
where `${VN}` is the LLVM version you are using. Add the following to the
`fpm.toml` for your project:
```toml
[features]
enzyme-lto.flang.flags = "-flto"
enzyme-lto.flang.link-time-flags = "-fuse-ld=lld -Wl,--load-pass-plugin=$ENZYME_LLD_PLUGIN"
enzyme-lto.preprocess.cpp.macros = ["HAS_ENZYME"]
```
If you are on a non-Unix system then you may need to hard-code
`ENZYME_LLD_PLUGIN` rather than using an environment variable.

## Function hooks for differentiation

We provide bindings for the `__enzyme_fwddiff` and `__enzyme_autodiff` function
hooks using implicit interfaces. Some Fortran compilers disallow procedure names
starting with an underscore so we rename the function hooks to remove the
leading double underscore.

To make use of the `enzyme_autodiff` function hook in your code, import it from
the bindings module and call it as a subroutine or function as appropriate. For
example, consider the Fortran equivalent of the example of differentiating a
function that squares a scalar argument in the following code snippet:
```fortran
program main
  use enzyme, only: enzyme_autodiff
  implicit none
  real :: x, dx

  x = 3
  print *, square(x)
  dx = 0
  call enzyme_autodiff(square, x, dx)
  print *, dx

contains

  real function square(x)
    implicit none
    real, intent(in) :: x
    square = x**2
  end function square

end program main
```

Similarly for `enzyme_fwddiff`. Thanks to the implicit interface, arbitrary
signatures are supported, with the following caveats.

* Implicit interfacing is that it only works for arguments that are passed by
  reference, which is the default in Fortran. If you want to pass any arguments
  by value using the `value` attribute then you will need to write an explicit
  interface block to the function hook yourself. See the
  [`square_with_explicit_interface`](https://github.com/EnzymeAD/Enzyme/blob/main/enzyme/test/Fortran/ReverseMode/square_with_explicit_interface.f90)
  test for an example.
* The implicit interfacing approach is not supported by the Intel Fortran
  compiler ifx when running without optimizations, i.e., running with `-O0`. If
  you want to use ifx with `-O0` then you will need to write an explicit
  interface block, even if you are only passing arguments by reference.
* Differentiation with respect to procedures with assumed shape arrays is not
  currently supported when compiling with Flang. It should work with ifx,
  however.

## Activity descriptors

We provide bindings for the activity descriptors `enzyme_const`, `enzyme_dup`,
`enzyme_dupnoneed`, and `enzyme_out`, as well as the descriptors
`enzyme_scalar`, `enzyme_width`, and `enzyme_vector`. To make use of these in
your code, import them from the bindings module and use them in calls to
function hooks in the same way you would do in C or C++. We can write the
example of squaring a scalar argument using the `enzyme_dup` activity descriptor
as follows:
```fortran
program main
  use enzyme, only: enzyme_autodiff, enzyme_dup
  implicit none
  real :: x, dx

  x = 3
  print *, square(x)
  dx = 0
  call enzyme_autodiff(square, enzyme_dup, x, dx)
  print *, dx

contains

  real function square(x)
    implicit none
    real, intent(in) :: x
    square = x**2
  end function square

end program main
```

## Function hook for batching

We do not currently provide bindings for the `__enzyme_batch` function hook
because it requires `enzyme_width` to be passed-by-value as an integer and this
is not supported by the implicit interfacing approach used for the other
function hooks. As such, you will need to write your own explicit `interface`
block to handle the batching. The following code snippet demonstrates how to
apply batching to the square example considered above:
```fortran
module squareBatch
  implicit none
  public

  interface
    subroutine square__enzyme_batch(sr, width_desc, width, &
                                    vec_desc, x1, x2, x3, x4, y1, y2, y3, y4)
      implicit none
      interface
        subroutine sr_decal(xx, yy)
          implicit none
          real, intent(in)  :: xx
          real, intent(out) :: yy
        end subroutine sr_decal
      end interface
      procedure(sr_decal)        :: sr
      integer, value, intent(in) :: width_desc
      integer, value, intent(in) :: width
      integer, value, intent(in) :: vec_desc
      real, intent(in)           :: x1, x2, x3, x4
      real, intent(out)          :: y1, y2, y3, y4
    end subroutine square__enzyme_batch
  end interface

contains

  subroutine square(x, y)
    implicit none
    real, intent(in)  :: x
    real, intent(out) :: y
    y = x ** 2
  end subroutine square

end module squareBatch

program main
  use enzyme, only: enzyme_vector, enzyme_width
  use squareBatch, only: square, square__enzyme_batch
  implicit none
  real :: x1, x2, x3, x4
  real :: y1, y2, y3, y4

  x1 = 23.1
  x2 = 10.0
  x3 = 100.0
  x4 = 3.14

  call square__enzyme_batch(square, enzyme_width, 4, &
                            enzyme_vector, x1, x2, x3, x4, &
                            y1, y2, y3, y4)

  print *, y1, y2, y3, y4
end program main
```

Notes:

* In C, the batched output is provided using a simple `struct`. The required
  syntax is different in Fortran - you should instead provide each entry of the
  output batch individually.
* You will likely find that batching works more straightforwardly with
  subroutines than with Fortran functions.
