# Input/Output
## Basics
### Write vs. Print

- `print` always writes to the default output unit (usually the terminal) and
  takes only a format
- `write` takes a control list: a unit, a format, and optional
  specifiers

This code does the same thing:
```fortran
print *, 'x = ', x              ! same as  write(*, *) 'x = ', x
print '(a, f0.3)', 'x = ', x    ! same as  write(*, '(a, f0.3)') 'x = ', x
```

Only `write` can:

* write to a file, to a specific unit such as `error_unit` from
  `iso_fortran_env`, or to a character variable (an internal file)
* do unformatted output
* catch errors with `iostat=` and `iomsg=`
* use `advance='no'`, `pos=`, `rec=`, `asynchronous=`, and the other
  data transfer specifiers. A data transfer specifier is one of the optional
  keyword=value items you can put inside the parentheses of a read or write
  statement

```fortran
use, intrinsic :: iso_fortran_env, only : error_unit

write(error_unit, '(a)') 'warning: dt reduced'
write(u, '(es12.4)', iostat=ios, iomsg=msg) x
write(*, '(a)', advance='no') 'Enter n: '
```

Use `print` for quick output to the screen. Use `write` for everything else.

### Read

`read` mirrors `write`. It has a short form like `print` that reads from the
default input unit, and a full form with a control list:

```fortran
read *, n                   ! same as  read(*, *) n
read(u, '(a)') line         ! read one line from unit u
```

List-directed input (`*` as the format) accepts values separated by blanks,
commas, or new lines. A formatted read uses fixed columns given by the format.

A read from a character variable (an internal file) converts a string to a
number:

```fortran
character(len=8) :: str = '42'
integer :: n

read(str, *) n              ! n = 42
```

Use `iostat=` to detect the end of a file instead of letting the program stop:

```fortran
use, intrinsic :: iso_fortran_env, only : iostat_end

do
    read(u, '(a)', iostat=ios) line
    if (ios == iostat_end) exit         ! end of file
    if (ios /= 0) error stop 'read error'
    ! ... process line ...
end do
```

### ISO Fortran Environment Units

The intrinsic module `iso_fortran_env` provides named constants for the
preconnected units. Use them instead of hard-coding unit numbers like `6`,
because the actual numbers depend on the compiler.

| Unit                   | Stream | Use with         | Typical value | Notes                                             |
|------------------------|--------|------------------|---------------|---------------------------------------------------|
| `*`                    | stdout | `write`, `print` | n/a           | Same unit as `output_unit` for output             |
| `output_unit`          | stdout | `write`          | `6`           | Normal program output                             |
| `error_unit`           | stderr | `write`          | `0`           | Warnings and errors, can be redirected separately |
| `input_unit`           | stdin  | `read`           | `5`           | `read(*, ...)` reads from the same unit           |
| `newunit=` (on `open`) | a file | `write`, `read`  | negative      | Compiler picks a free unit, F2008                 |

```fortran
use, intrinsic :: iso_fortran_env, only : input_unit, output_unit, error_unit

integer :: u

write(output_unit, '(a)') 'results follow'
write(error_unit,  '(a)') 'warning: dt reduced'
read(input_unit, *) n

open(newunit=u, file='out.txt', action='write')
write(u, '(i0)') n
close(u)
```

### Derived Type I/O Example (aka Defined Type I/O) Fortran 2003
```fortran
module shapes
  implicit none
  type :: circle
    real :: r
  contains
    procedure :: write_circle
    generic :: write(formatted) => write_circle
  end type
contains
  subroutine write_circle(dtv, unit, iotype, v_list, iostat, iomsg)
    class(circle),    intent(in)    :: dtv
    integer,          intent(in)    :: unit
    character(*),     intent(in)    :: iotype    ! 'LISTDIRECTED', 'NAMELIST', or 'DT...'
    integer,          intent(in)    :: v_list(:) ! integers from dt(...) in the format
    integer,          intent(out)   :: iostat
    character(*),     intent(inout) :: iomsg
    write(unit, '(a, f0.2, a)', iostat=iostat, iomsg=iomsg) 'circle(r=', dtv%r, ')'
  end subroutine
end module


type(circle) :: c = circle(2.5)
print *, c                  ! circle(r=2.50)
write(*, '(dt)') c          ! same, through the dt
```



# New Input/Output Features Since Fortran 2008

The I/O changes in Fortran 2018 and Fortran 2023 are mostly small: new edit
descriptors, better control over how numbers are printed, and fixes for
long-standing restrictions. The summaries below are based on John Reid's
[The new features of Fortran 2018](https://wg5-fortran.org/N2151-N2200/N2161.pdf)
and [The new features of Fortran 2023](https://wg5-fortran.org/N2201-N2250/N2212.pdf).
The last section covers the next revision, Fortran 2028, and is based on the
Committee Draft J3/26-007r2 (June 2026), so it may still change.


## Fortran 2018

### Zero-width `d`, `e`, `es`, `en`, and `g` edit descriptors

Fortran 2008 allowed a zero field width only for `i0`, `f0.d`, and some `g`
output. In Fortran 2018, a field width of zero works for all of these
descriptors. The processor then uses the smallest width that does not fill
the field with asterisks.

```fortran
real :: x = 12345.678

write(*, '(es0.3)') x      ! 1.235E+04, no leading blanks (gfortran prints 1.235E+4)
```

`g0.d` can now also be used for integer, logical, and character output. These
items are written as if `i0`, `l1`, and `a` had been used, so one format works
for a mixed list:

```fortran
write(*, '(*(g0.3, 1x))') 42, .true., 'label', 3.14159
```

A zero width is not allowed on input.

---

### Minimal exponent digits with `ew.dE0`

Setting the exponent width to zero tells the processor to use only as many
exponent digits as the value needs. This removes leading zeros in the exponent,
which can cause problems when another language reads the output.

```fortran
write(*, '(es12.4e0)') 1.0e5     ! 1.0000E+5
```

---

### Hexadecimal output of reals: `ex`

The new `exw.d` and `exw.dEe` descriptors write a real value with a
hexadecimal significand and a power-of-2 exponent. Because no decimal
rounding happens, this is useful for writing exact floating-point values.

| Internal value | Edit descriptor | Possible output   |
|----------------|-----------------|-------------------|
| `1.375`        | `ex0.1`         | `0X1.6P+0`        |
| `-15.625`      | `ex14.4e3`      | `-0X1.F400P+003`  |
| `1048580.0`    | `ex0.0`         | `0X1.00004P+20`   |

* `w = 0` lets the processor choose the field width.
* `d = 0` writes all of the hexadecimal digits.
* On input, `ex` behaves the same as `fw.d`.

---

### `inquire` enhancements

* Record Length: `recl=` used to become undefined when there was no connection
  or the connection was for stream access. Now it returns `-1` when the unit
  is not connected and `-2` when it is connected for stream access.
* `pos=` and `size=` now return values as if all pending asynchronous data
  transfers had finished. Useful because it is now deterministic and there is
  no need to add waiting.

```fortran
inquire(unit=u, recl=rl)
if (rl == -2) print *, 'stream access, no record length'
```


#### Fortran Access Modes

| Access                   | File is treated as                                          |
|--------------------------|-------------------------------------------------------------|
| `'sequential'` (default) | a series of records read in order (e.g. lines of text)      |
| `'direct'`               | fixed-length records you can jump to by number (`rec=`)     |
| `'stream'` (F2003)       | a plain sequence of bytes or characters, like a file in C   |

---

### `size=` with advancing input

`size=` returns the number of characters read. In Fortran 2008 it required
`advance='no'`. That restriction has been removed:

```fortran
character(len=256) :: line
integer :: nchars

read(u, '(a)', size=nchars) line   ! advancing read
```

---

### Formatted input errors

If a field read with numeric or logical editing is not in a standard form and
the processor does not accept it, this is now an error. It can be caught with
`iostat=`, so programs can detect bad input instead of getting undefined
values.

---

### Connecting a file to more than one unit

The rule that a file may be connected to only one unit at a time has been
removed. For example, two units could read different parts of the same
sequential file. Compilers are not required to support this, so whether it
works is processor dependent.


## Fortran 2023

### The `at` edit descriptor

`at` writes a character value as if `trim` had been applied to it. This
removes the need for `trim` calls in output lists:

```fortran
character(len=32) :: name = 'temperature'

write(*, '(at, " = ", f0.2)') name, 273.15   ! temperature = 273.15
```

`at` can only be used for output.

---

### Control over leading zeros

The output of a value such as `0.5` with `f`, `e`, `d`, or `g` editing could
be `0.50` or `.50`, depending on the compiler. A new *leading zero mode*
controls this:

| Mode                 | Edit descriptor | Effect                        |
|----------------------|-----------------|-------------------------------|
| `'print'`            | `lzp`           | Always write the optional `0` |
| `'suppress'`         | `lzs`           | Never write the optional `0`  |
| `'processor_defined'`| `lz`            | Compiler's choice (default)   |

The mode for a connection is set with `open` and can be checked with
`inquire`:

```fortran
open(newunit=u, file='out.txt', leading_zero='print')
inquire(unit=u, leading_zero=lz_mode)
```

The edit descriptors change the mode only for the current output statement:

```fortran
write(*, '(lzp, f5.2)') 0.5    !  0.50
write(*, '(lzs, f5.2)') 0.5    !   .50
```

They have no effect on input.

---

### Automatic allocation in `iomsg=` and internal writes

Deferred-length allocatable character variables are now allocated to the
correct length when they get a value from:

* an `iomsg=` specifier
* an internal `write` to a scalar character variable

```fortran
character(:), allocatable :: msg, str
integer :: ios

open(newunit=u, file='missing.dat', status='old', iostat=ios, iomsg=msg)
if (ios /= 0) print *, msg       ! msg has exactly the length of the message

write(str, '(i0)') 12345         ! str is allocated with len(str) == 5
```

Before this change, the variable had to be declared with a fixed length that
was long enough, and the result often needed to be trimmed.

---

### Namelist

A `public` namelist group may now contain `private` variables. This lets a
module make a namelist available for reading or writing without exposing the
variables it contains.

```fortran
module config
  implicit none
  private
  integer :: nsteps = 100
  real    :: dt = 0.1
  namelist /run_params/ nsteps, dt
  public :: run_params
end module config
```

---

## Fortran 2028 (Committee Draft)

The draft lists only two input/output extensions to Fortran 2023. A third
change, to the `generic` statement, also applies to defined input/output.

### I/O of enumeration types

Fortran 2023 added enumeration types, but their values could only be read or
written with `i`, `b`, `o`, or `z` editing, which use the ordinal position
(`1`, `2`, `3`, ...). Fortran 2028 lets enumeration values be read and written
**by name**:

* in list-directed I/O (`print *`, `read(u, *)`)
* in namelist I/O
* with the `g` edit descriptor in an explicit format

```fortran
enumeration type :: colour
  enumerator :: red, orange, green
end enumeration type

type(colour) :: light = orange

print *, light               ! ORANGE
write(*, '(g0)') light       ! ORANGE
write(*, '(g8, "|")') light  ! ORANGE  |
write(*, '(i0)') light       ! 2  (ordinal position, already allowed in F2023)

read(*, *) light             ! accepts green, Green, or GREEN
```

Rules for `g` editing of enumeration values (section 13.7.5.4 of the draft):

* Output is the enumerator name in **upper case**. With `g0` the field is the
  same width as the name. With `gw`, the name is left-justified and followed by
  blanks, or the field is filled with `w` asterisks if the name does not fit.
* On input, the field must contain one of the type's enumerator names, with
  optional blanks before and after it. Case does not matter. Any other value
  is an error condition.
* The enumerator name does not need to be accessible where the I/O statement
  is.
* List-directed and namelist output act like `g0`. List-directed and namelist
  input act like `gw`.

In the r2 draft, 12.6.3 paragraph 6 still says that an enumeration type item
cannot appear in list-directed I/O and must use `i`, `b`, `o`, or `z`. This
conflicts with the introduction and with 13.7.5.4, 13.10, and 13.11, and is
probably left over from the Fortran 2023 text.

---

### `Dw.d` is obsolescent

`Dw.d` was meant for double precision values when there were only two real
kinds. It is now obsolescent because:

* it can write an `E` exponent letter instead of `D`
* it has no form with an exponent width (`Ew.dEe` does)
* `Ew.d` and `Ew.dEe` work with every real kind

```fortran
write(*, '(d15.7)') x      ! obsolescent
write(*, '(e15.7e3)') x    ! use this instead
```

---

### Adding a whole generic to defined I/O

A `generic` statement, or a `procedure` statement in a generic interface
block, can now list a **generic name**. Every specific procedure of that
generic is then added to the generic identifier. The identifier can be a
defined operator, defined assignment, or **defined input/output**:

```fortran
interface write_shape
  module procedure write_circle, write_square
end interface

generic :: write(formatted) => write_shape   ! adds write_circle and write_square
```

Before this change, every specific procedure had to be listed again.
