# By Rvalue

All rvalues begin by evaluating their place/operand params in the order they appear in the type.
Then, they produce a value in a way that is specific to which rvalue they are.

## `Use`

Yields the operand unchanged.

## `Repeat`

Creates an array where each element is the value of the operand. This currently does not drop the
value even if the number of repetitions is zero, see [#74836][74836].

[74836]: https://github.com/rust-lang/rust/issues/74836

## `Ref` & `AddressOf`

Generates an appropriately typed reference or pointer to the place. There is not much to document
here, because besides the obvious parts the semantics of this are essentially entirely a part of the
aliasing model. There are too many UCG issues to link.

`Shallow` borrows are disallowed after drop lowering.

## `ThreadLocalRef`

Returns a pointer/reference to the given thread local. The return type is a `*mut T` if the static
is mutable, otherwise if the static is exter a `*const T`, and if neither of those apply a `&T`.

**Note:** This is a runtime operation that actually executes quite a bit of code and is in this
sense more like a function call. Also, DSEing these causes `fn main() {}` to SIGILL for some reason
that I never got a chance to look into.

**NC**: Are there weird additional semantics here related to the runtime nature of this operation?

## `Len`

Yields the length of the place, as a `usize`. If the type of the place is an array, this is the
array length. This also works for slices (`[T]`, not `&[T]`) through some mechanism that depends on
how exactly places work (see there for more details).

## `Cast` & `ShallowInitBox`

Performs essentially all of the casts that can be performed via `as`. The types on either end must
be numeric or pointer types, with the output of `ShallowInitBox` being `Box`. Those casts that
involve pointers on either end are generally affected by alias analysis. The remaining casts are
between numeric types.

**TODO**: Figure out why `ArrayToPointer` and `MutToConstPointer` are special and what the exact
combinations of types are that are allowed.

`ShallowInitBox` is disallowed after drop elaboration.

## `BinaryOp`

`BinOp::Offset` has exactly the behavior of the [`ptr::offset`][offset] method. Other binary ops
perform two's-complement arithmetic on integers. Division by zero (including modulo operations) are
UB. Besides that:

* The comparison operations accept `bool`s, `char`s, signed or unsigned integers, or floats and
  return a `bool`. Of course the types of the sides of the comparison must match.
* Left and right shift operations accept signed or unsigned integers not necessarily of the same
  type and return a value of the same type as their LHS.
* The remaining operations accept signed integers, unsigned integers, or floats of any matching type
  and return a value of that type.

[offset]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset

## `CheckedBinaryOp`

Same as `BinaryOp`, but yields `Option<T>` instead of `T`. Furthermore, instead of performing
two's-complement arithmetic, checks if the infinite precison result would be unequal to the
two's-complement result and returns `None` if so. `BinOp::Offset` is not allowed here.

**NC/TODO**: What about division/modulo? Are they allowed here at all? Are zero divisors still UB?
Also, which other combinations of types are disallowed?

## `UnaryOp`

Essentially exactly like `BinaryOp`, but less operands. Also does two's-complement arithmetic.
Negation requires a signed integer or a float; not requires a signed integer, unsigned integer, or
bool. Both operation kinds return a value with the same type as their operand.

## `NullaryOp`

Yields the size or alignment of the type as a `usize`.

## `Aggregate`

Assigns the operands in order to the fields for the aggregate kind. For arrays, these are the
elements of the array; for tuples, these are the fields of the tuple; for ADTs and generators, these
are the fields of the variant (or just the one field for unions); and for closures these are the
upvars.

This may additionally initialize other bytes besides the fields in the case of ADTs/generators if
the layout of the variant requires it. The remaining bytes are left uninit.

Disallowed after deaggregation for all aggregate kinds except `Array`.

## `Discriminant`

Computes the discriminant of the ADT or generator, returning it as an integer of type
[`discriminant_ty`][discrim-ty]. The validity requirements for the underlying value are undecided
for this rvalue, see [#91095][91095]. Note too, that the value of the discriminant is not the same
thing as the variant index; see [`discriminant_for_variant`][discrim-vs-variant] for converting.

[discrim-ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.Ty.html#method.discriminant_ty
[91095]: https://github.com/rust-lang/rust/issues/91095
[discrim-vs-variant]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.Ty.html#method.discriminant_for_variant
