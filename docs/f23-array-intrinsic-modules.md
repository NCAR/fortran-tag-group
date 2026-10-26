# New Array Features, Intrinsic Procedures and Modules

## New Array Features

### Multiple Subscripts with `@`

A rank-one integer array can now provide several consecutive subscripts.

```fortran
integer :: location(2) = [3, 5]
real    :: a(10, 10)

print *, a(@location)
```

This is equivalent to:

```fortran
print *, a(3, 5)
```

Multiple subscripts can be mixed with ordinary subscripts:

```fortran
integer :: middle(2) = [3, 5]

value = field(2, @middle, 7)
```

This is equivalent to:

```fortran
value = field(2, 3, 5, 7)
```

A multiple subscript is different from a vector subscript:

* A vector subscript selects several elements from one dimension.
* A multiple subscript supplies subscripts for several consecutive dimensions.
* The number of dimensions supplied is `size(integer_array)`.

---

### Multiple Subscript Triplets

Rank-one integer arrays can provide the lower bounds, upper bounds, and strides for several array dimensions.

```fortran
integer :: lower(2)  = [3, 5]
integer :: upper(2)  = [9, 10]
integer :: stride(2) = [2, 3]

section = a(@lower:upper:stride)
```

This is equivalent to:

```fortran
section = a(3:9:2, 5:10:3)
```

Rules:

* At least one part of the triplet must be a rank-one integer array.
* Array components must have the same size.
* Scalar components are applied to every generated dimension.
* An omitted lower bound, upper bound, or stride is omitted for every generated dimension.

For example:

```fortran
section = a(@lower:upper:2)
```

is equivalent to:

```fortran
section = a(lower(1):upper(1):2, lower(2):upper(2):2)
```


---

### Integer Arrays for Rank and Bounds

Rank-one integer arrays can now be used in:

* Array declarations
* `allocate` statements
* Pointer remapping

The general rule is:

> The size of the bounds array determines the rank, and its elements determine the bounds of each dimension.

#### Assumed-shape declarations

```fortran
integer, parameter :: lower(3) = [0, 0, 1]

subroutine process(x)
    real :: x(lower:)
end subroutine
```

Because `lower` has three elements, `x` has rank three.

This is equivalent to:

```fortran
subroutine process(x)
    real :: x(0:, 0:, 1:)
end subroutine
```

The size of the bounds array must be known at compile time because it determines the declared rank.

#### Explicit-shape declarations

Both lower and upper bounds can be arrays:

```fortran
integer, parameter :: lower(3) = [0, 0, 1]
integer, parameter :: upper(3) = [9, 19, 4]

real :: field(lower:upper)
```

This is equivalent to:

```fortran
real :: field(0:9, 0:19, 1:4)
```

A scalar bound can be applied to every dimension:

```fortran
real :: field(0:upper)
```

This uses zero as the lower bound for every dimension.

An especially useful example is:

```fortran
real :: copy(shape(original))
```

This declares `copy` with the same extents as `original`.

Its lower bounds default to one.

#### Allocation

Bounds arrays can be used in `allocate` statements:

```fortran
real, allocatable :: x(:,:,:)
integer :: lower(3), upper(3)

allocate(x(lower:upper))
```

A scalar bound can also be applied to every dimension:

```fortran
allocate(x(0:upper))
```

The size of each bounds array must match the rank of the allocatable object.

#### Pointer remapping

Bounds arrays can be used in pointer remapping:

```fortran
real, target  :: storage(1000)
real, pointer :: cube(:,:,:)
integer       :: lower(3), upper(3)

cube(lower:upper) => storage
```

This feature does not apply to coarray cobounds.

---

### The `rank(n)` Declaration Clause

The rank of an assumed-shape or deferred-shape entity can now be written explicitly.

```fortran
subroutine process(a)
    real, rank(2) :: a
end subroutine
```

This is equivalent to:

```fortran
subroutine process(a)
    real :: a(:,:)
end subroutine
```

Rules:

* `rank(0)` declares a scalar.
* The rank must not exceed the processor's maximum supported rank.
* Without `allocatable` or `pointer`, it declares an assumed-shape entity.
* With `allocatable` or `pointer`, it declares a deferred-shape entity.
* The argument must be an integer constant expression.

The rank can be derived from another array:

```fortran
real :: foo(10, 20, 30)

real, rank(rank(foo)), allocatable :: work
```

Here, `work` is a deferred-shape rank-three array.

This avoids repeating rank information manually.

#### Review: Assumed Shape vs. Deferred Shape

* **Assumed-shape arrays** receive their shape from the actual argument when a procedure is called. They are declared with `:` and are not allocatable by the procedure.

  ```fortran
  real :: a(:,:)
  ```

* **Deferred-shape arrays** do not have a shape until they are allocated or pointer-associated. They must have the `allocatable` or `pointer` attribute.

  ```fortran
  real, allocatable :: a(:,:)
  real, pointer     :: p(:,:)
  ```


---

### Reductions in `do concurrent`

Just doing a short note of this feature, `do concurrent` will be covered in a future session.
A `do concurrent` construct can now identify reduction variables using the `reduce` locality specifier.
The reduce clause tells the processor that updates to these variables form reductions and may be parallelized.

```fortran
do concurrent (i = 1:n)
  reduce(+:total)
  reduce(max:largest_val)

  total = total + x(i)**2
  largest_val = max(largest_val, x(i))
end do
```

---

## New Intrinsic Procedures and Modules

### `split`

The new intrinsic subroutine `split` locates separator characters in a string.

```fortran
call split(string, set, position [, back])
```

Arguments:

* `string`: Input character scalar.
* `set`: Characters treated as separators.
* `position`: Integer search position.
* `back`: Optional logical controlling the search direction.

#### Forward search

For a forward search:

* Begin with `position` between zero and `len(string)`.
* The procedure returns the position of the first separator after `position`.
* It returns `len(string) + 1` when no separator remains.

#### Backward search

For a backward search:

```fortran
call split(string, set, position, back=.true.)
```

The procedure:

* Searches before the current position.
* Returns the last separator before `position`.
* Returns zero when no separator is found.

`split` is a low-level procedure for iterating through separators one at a time.

---

### 2.2 `tokenize`

The new intrinsic subroutine `tokenize` separates a string into tokens.

There are two overloaded subroutines, one with character array arguments and one with integer array arguments.

#### Returning token strings

```fortran
character(:), allocatable :: words(:)
character(:), allocatable :: separators(:)

call tokenize(input_text, " ,;", words, separators)
```

The `words` array is automatically allocated.

Properties:

* It is rank one.
* Its size is the number of tokens.
* Its character length is the length of the longest token.
* Shorter tokens are padded with spaces.

The optional `separators` result records the separator characters that were encountered.

Consecutive separators can produce zero-length tokens.

A separator at the end of the string can also produce a zero-length token.

#### Returning token positions

```fortran
integer, allocatable :: first(:), last(:)

call tokenize(text, " ,;", first, last)
```

This form returns the beginning and ending positions of each token without copying the token text.

For a zero-length token:

```fortran
last(i) == first(i) - 1
```

This allows an empty token to be represented naturally.

Both `split` and `tokenize` are classified as simple procedures.

Their `intent(out)` and `intent(inout)` arguments cannot be coarrays or coindexed objects.


---

#### Simple, Pure, and Impure Procedure Review
Fortran procedures (subroutines and functions) can be classified by how they interact with program state.
* *Impure procedures* are the default. They may read or modify variables outside their local scope, perform input/output, and have other side effects.
* *Pure procedures* may modify data outside their scope only through their arguments. Module variables are a case of data inside their scope. This makes them safer for parallel code.
* *Simple procedures* was introduced in F23 and is a more restrictive than pure procedures. It can only access data passed through its arguments.
#### Elemental Procedure Review
Elemental procedures are written for scalar arguments, but can also be called with conformable arrays.

```fortran
elemental real function square(x)
  real, intent(in) :: x
  square = x * x
end square

...

real :: a(4), foo(4), res
res = square(2.0)
foo = square(a)
```


---

### Trigonometric Functions Using Degrees

Fortran 2023 adds elemental trigonometric functions that use degrees.

```fortran
sind(x)
cosd(x)
tand(x)

asind(x)
acosd(x)
atand(x)
atan2d(y, x)
```

Examples:

```fortran
sind(30.0)        ! Approximately 0.5
acosd(0.0)        ! Approximately 90.0
atan2d(1.0, 1.0)  ! Approximately 45.0
```

The two-argument form:

```fortran
atand(y, x)
```

is equivalent to:

```fortran
atan2d(y, x)
```

The functions are elemental, so they work with both scalars and arrays.

The inverse functions return angles in degrees.

---

### 2.4 Trigonometric Functions Using Multiples of π

Another set of elemental trigonometric functions uses units of π radians.

```fortran
sinpi(x)
cospi(x)
tanpi(x)

asinpi(x)
acospi(x)
atanpi(x)
atan2pi(y, x)
```

Examples:

```fortran
sinpi(0.5)  ! sin(0.5*pi), approximately 1
cospi(1.0)  ! cos(pi), approximately -1
```

Inverse functions return results in multiples of π:

```fortran
asinpi(1.0)  ! Approximately 0.5
acospi(0.0)  ! Approximately 0.5
```

These functions avoid explicitly multiplying or dividing by an approximation to π.

---

### 2.5 `selected_logical_kind`

The new selected kind instrinsic for logical kinds.
Added to match the selected kind functions for all the other types.

```fortran
selected_logical_kind(bits)
```

returns a logical kind whose storage size is at least the requested number of bits.

Example:

```fortran
integer, parameter :: logical_kind = selected_logical_kind(32)

logical(kind=logical_kind) :: flags
```

Selection rules:

* Select a logical kind with sufficient storage.
* Prefer the smallest sufficient storage size.
* Return `-1` if no suitable logical kind exists.

Programs should check for a negative result when requesting a kind that may not be available.

---

### Changes to `system_clock`

Fortran 2023 clarifies the relationship between integer kinds and the clocks returned by `system_clock`.

Within one call, all integer arguments must have the same kind.

This removes ambiguity that previously existed when different integer kinds were mixed in one call.


```fortran
use iso_fortran_env, only : int64

integer(int64) :: count, rate, maximum

call system_clock(count, rate, maximum)
```

Important points:

* All integer arguments in one call must have the same kind.
* A processor may provide different clocks for different integer kinds.
* A processor may technically provide no usable clock.

---

### Additions to `ieee_arithmetic`

The intrinsic module `ieee_arithmetic` adds four elemental functions:

```fortran
ieee_max(x, y)
ieee_min(x, y)
ieee_max_mag(x, y)
ieee_min_mag(x, y)
```

Meanings:

* `ieee_max`: Returns the greater value.
* `ieee_min`: Returns the lesser value.
* `ieee_max_mag`: Returns the value with the greater absolute magnitude.
* `ieee_min_mag`: Returns the value with the lesser absolute magnitude.

The arguments must be real and have the same kind.

These functions define IEEE behavior for:

* Quiet NaNs
* Signaling NaNs
* Positive zero
* Negative zero

For example, when comparing positive and negative zero, the result is positive zero.

The existing functions were also revised:

```fortran
ieee_max_num
ieee_min_num
ieee_max_num_mag
ieee_min_num_mag
```

Their behavior was updated to match ISO/IEC 60559:2020, particularly for NaNs and signed zeros.

---

### New Constants in `iso_fortran_env`

The following constants are added to `iso_fortran_env`:

```fortran
logical8
logical16
logical32
logical64
real16
```

Example:

```fortran
use iso_fortran_env, only : logical32, real16

logical(logical32) :: mask
real(real16)       :: compact_value
```

The constants identify kinds according to storage size in bits.

If the exact requested storage size is unavailable:

* The constant is `-2` if a larger kind exists.
* The constant is `-1` if no kind of that size or larger exists.

If several kinds have the requested storage size, the processor chooses which kind is represented by the constant.

These constants specify storage size. They do not necessarily guarantee:

* A particular numerical precision
* A particular IEEE representation
* A particular internal encoding



### Kind Parameter Review

A **kind parameter** identifies a particular representation of an intrinsic type, such as an integer range or real precision.

A particular representation means one of the processor’s available ways of storing and interpreting values of a type.

For integers, different kinds may differ in:

* Storage size
* Range of representable values
* Internal binary format


An example of the older, less preferred way:

```fortran
integer(kind=8) :: i
real(kind=8)    :: x
```

Literal kind numbers such as `8` are processor dependent, so portable code should use named constants from `iso_fortran_env`:

```fortran
use, intrinsic :: iso_fortran_env, only : int64, real64

integer(int64) :: i
real(real64)   :: x
```

Kinds can also be selected according to required capabilities:

```fortran
integer, parameter :: rk = selected_real_kind(p=15, r=300)
integer, parameter :: ik = selected_int_kind(r=12)
```

* `selected_real_kind` selects a real kind with the requested decimal precision and exponent range.
* `selected_int_kind` selects an integer kind capable of representing the requested decimal range.
* `selected_logical_kind` selects a logical kind with at least the requested storage size.

A negative result means that no suitable kind is available.

The kind of an expression or variable can be queried with:

```fortran
kind(x)
```

Kinds improve portability, but they specify representation capabilities rather than guaranteeing an exact hardware type.
