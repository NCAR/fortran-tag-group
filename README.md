# Fortran TAG Group
This group has been created to discuss and learn about new features in the
  Fortran Standard.
## Documents
The version of the standard that will be used is the newest 2023 draft
standard.
Draft versions will be used because they have the major benefit of being free!
The current F28 draft standard is also listed for the curious, this document
is a living document and will be updated as the committee edits it.

* [Fortran 2023 Standard](https://j3-fortran.org/doc/year/24/24-007.pdf)
* [The new features of Fortran 2018](https://wg5-fortran.org/N2151-N2200/ISO-IECJTC1-SC22-WG5_N2161_The_New_Features_of_Fortran_2018.pdf)
  by John Reid
* [The new features of Fortran 2023](https://wg5-fortran.org/N2201-N2250/N2212.pdf)
  by John Reid
* [New Patterns in Fortran](https://cass.community/events/hpcbp-095-fortran)
  by Damian Rouson
* [Fortran 2028 Working Draft](https://j3-fortran.org/doc/year/26/26-007r1.pdf)


## Schedule
Last Monday of the month, unless otherwise noted.

| date       | standard      | topic                                                       |
|------------|---------------|-------------------------------------------------------------|
| 2026.06.29 | F23           | Introduction, [F23 Feature Overview](#f23-feature-overview) |
| 2026.07.27 | F23           | New Array Features, Intrinsic Procedures and Modules        |
| 2026.08.31 | F23           | Input/Output; F23/F18: Obsolete Features                    |
| 2026.09.28 | F23, F18, F08 | Interoperability: C, Python, Etc                            |
| 2026.10.26 | F23, F18, F95 | Simple, Elemental, Impure, Pure                             |
| 2026.11.30 | F23, F18, F08 | Parallel Features, Do Concurrent, Reduction                 |
| 2027.01.25 | F23, F18, F08 | Parallel Coarrays                                           |
| 2027.02.22 | F23, F18, F08 | Teams                                                       |
| 2027.03.29 | F23, F18, F08 | Events                                                      |
| 2027.04.26 | F18           | Conformance with ISO/IEC/IEEE 60559:2011                    |
| 2027.06.28 | F18           | Features that address deficiencies and discrepancies        |
| 2027.07.26 | F28           | Overview, Generics and Templates                            |


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
  penumerator :: red, orange, green
end enumeration type
type(colour) light
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
