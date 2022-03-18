# Places

Places in MIR have three components:

 1. The range of bytes it references in memory.
 1. The type of the place and possibly a variant index. See [`PlaceTy`][placety]

[placety]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/tcx/struct.PlaceTy.html

 1. The provenance with which the place is being accessed.

For a place that has no projections, ie `Place { local, projection: [] }`, these values are the
obvious ones: The range of bytes is the local's full allocation, the type is the type of the local,
and the provenance is whatever the aliasing model says. For any other place, we define the values as
a function of the parent place, that is the place with its last [`ProjectionElem`][projections]
stripped. The way this is computed of course depends on which `ProjectionElem` was the last one. The
details below specify this computation for the range of bytes and the type of the place; how each
projection influences provenance is dependent on the aliasing model, which is undecided, and so that
is ommited.

[projections]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/enum.ProjectionElem.html

 - `Downcast`: This projection sets the place's variant index to the given one. A `Downcast`
   projection on a place with its variant index already set is not well-formed.
 - `Field`: `Field` projections take their parent place and create a place referring to one of the
   fields of the type. They are only legal for tuples, ADTs, closures, and generators. If the ADT or
   generator has more than one variant, the parent place's variant index must be set, indicating
   which variant is being used. If it has just one variant, the variant index may or may not be
   included.
 - `ConstantIndex`: Indexes an array or slice. This is only legal if the parent place has type
   `[T;  N]` or `[T]` (*not* `&[T]`). The type of the resulting place is `T`.
 - `Subslice`: Much like `ConstantIndex`. It is also only legal on `[T; N]` and `[T]`. However, this
   yields a `Place` of type `[T]`. **NC**: Also legal for `str`? 
 - `Index`: Like `ConstantIndex`, only legal on `[T; N]` or `[T]`. However, `Index` additionally
   takes a local from which the value of the index is computed at runtime. Computing the value of
   the index involves interpreting the `Local` as a `Place { local, projection: [] }`, and then
   performing a place to value conversion to determine the index. The array/slice is then indexed
   with the resulting value. The local must have type `usize`.
 - `Deref`: Derefs are the last type of projection, and the most complicated. They are only legal on
   parent places that are references or pointers (**NC**: And `Box`?). A `Deref` projection begins
   by performing a place to value conversion on the parent place, to read the pointer/reference. It
   then dereferences this pointer, creating a place of the pointed to type. This dereferencing may
   be UB, depending on the aliasing model.

# Place to Value Conversions

Place to value conversions accept a place as defined above, and turn it into a value. This operation
may be UB, if the aliasing model says that the provenance of the place is insufficient for the
operation.

It is undecided when this UB kicks in. I believe that is the question being discussed in
[UCG#319][ucg-319]. Summarizing from that thread, I believe the options are:

 1. Each intermediate place must have provenance for all of the referred to bytes. This is the
    status quo.
 2. Only for intermediate place where the last projection was *not* a deref. This corresponds to
    "Check inbounds on place projection".
 3. Only on place to value conversions, assignments, and referencing operation. This corresponds to
    "remove the restrictions from `*` entirely."
 4. On each intermediate place if the place is used for a place to value conversion, assignment, or
    referencing operation. For a raw pointer computation, never. This corresponds to "magic?".

Hopefully I am not misrepresenting anyone's opinions - please let me know if I am. Currently, Rust
chooses option 1. This is checked by MIRI and taken advantage of by codegen (via `gep inbounds`).
That is possibly subject to change.

[ucg-319]: https://github.com/rust-lang/unsafe-code-guidelines/issues/319

A place to value conversion on a place that has its variant index set is not well-formed. However,
note that this rule only applies to places appearing in MIR bodies, or other situations in which the
well-formedness of places may be checked. Many functions, such as [`Place::ty`][place_ty], still
accept such a place. If you write a function for which it might be ambiguous whether such a thing is
accepted, make sure to document your choice clearly.

[place_ty]: https://doc.rust-lang.org/stable/nightly-rustc/rustc_middle/mir/struct.Place.html#method.ty

**NC**: Are place to value conversions typed? An interesting example here is this one:

```rust
pub fn main() {
    let mut r: &i32 = &5;
    let mut x = 3;
    let rp = &mut r as *mut _ as *mut &mut i32;
    unsafe { *rp = &mut x; }
    // `r`s memory now contains a `&mut x`.

    // The `r as *const i32` is lowered to `&raw *r` in MIR
    let p = r as *const i32 as *mut i32;
    unsafe { *p = 10 };
}
```

MIRI reports UB for this program. That means that MIRI is of the impression that at some point in
the `&raw *r` operation, we are losing write provenance. The question is, where? As far as I can
tell, we have four options:

 1. We are losing the provenance at the place to value conversion that happens immediately before
    the dereference. In order for this to be possible, this must be a typed operation - this allows
    the place to value conversion to retag the pointer with new, reduced provenance.
 2. We are losing the write provenance at the dereference. For this to be possible, the value
    yielded by the place to value conversion needs to be typed (so that we know what). The behavior
    of the dereference then depends on the type of the value, in addition to its bytes.
 3. Both of the above.
 4. Neither of the above, and MIRI is wrong. If we want the program to be UB, we need to codegen it
    differently.

# Operands

**NC**: Definition of an operand. It seems like operands are the same thing as "values" in "place to
value" conversion. That would mean they are a sequence of bytes, possibly with a type (depending on
the result of the above discussion).

Besides the previous uncertainty, computing an operand works as follows:

 - `Copy`: Performs a place to value conversion for the place. The type of the place must be `Copy`.
 - `Move`: Performs a place to value conversion for the place. This *may* additionally overwrite the
   place with `uninit` bytes, depending on how we decide in [UCG#188][ucg-188].

[ucg-188]: https://github.com/rust-lang/unsafe-code-guidelines/issues/188

 - `Constant`: Copies the bytes of the constant to produce a value.
