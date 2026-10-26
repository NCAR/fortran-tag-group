### F23 Feature Overview
For a more in-depth discussion of the new features in Fortran 2023, see John
  Reid's [The new features of Fortran 2023](https://wg5-fortran.org/N2201-N2250/N2212.pdf).
We will be discussing

* line length and continuation limit

The line length limit has been raised to
 10,000 characters. There is now no limit on number of continuation lines,
 it used to be 255, and the maximum statement length is a million characters.


 * `typeof` and `classof`

```fortran
subroutine work(a)
  type(foo) :: bar(:)
  typeof(bar) :: A(size(bar))
```

|         | GNU | Intel | NVIDIA | Cray |
|---------|-----|-------|--------|------|
| support | ❌   | ❌     | ?      | ?    |

* Enumeration Type

```fortran
enumeration type :: colour
  enumerator :: red, orange, green
end enumeration type
type(colour) :: light
...
if (light==red) ...
```

* Conditional Expressions

See Section [`10.1.2.3`](https://j3-fortran.org/doc/year/24/24-007.pdf) of the standard.

```fortran
foo = (err < 0.01 ? .true. : .false.)
! is equivalent to
if (err < 0.01) then
    foo = .true.
else
    foo = .false.
end if
```

* Extracting tokens from strings with `split` and `tokenize`

* Trig functions that work in degrees and half revolutions
  * degrees: add `d` to the instrinsic name, such as `cosd(x)`
  * half revolutions: add `pi` to instrinsic name, such as `cospi(x)`

* New kinds: `logical8, logical16, logical32, logical64, real16`

* Extend the intrinsic procedure `c_f_pointer` to allow its pointer result to
  have specified lower bounds

* Procedures for converting between Fortran and C strings: `f_c_string,
  c_f_strpointer, c_f_strpointer`

* IO: control over leading zeros of real values

* `Simple` procedure: cannot access outside variables. A stricter `pure` procedure

* Using integer arrays to specify subscripts and section subscripts
  * `A(@[2,7]) ! equivalent to A(2, 7)`
  * And more!

* Reduction specifier for `do concurrent`: using instrinsic operators `+, *, min, max, .and., .or., .eqv., .neqv.`
