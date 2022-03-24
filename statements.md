# By Statement

## `Assign`

Assign statements roughly correspond to an assignment in Rust proper (`x = ...`) except without the
possibility of dropping the previous value (that must be done separately, if at all). The *exact*
way this works is undecided. It probably does something like evaluating the LHS and RHS, and then
doing the inverse of a place to value conversion to write the resulting value into memory. Various
parts of this may do type specific things that are more complicated than simply copying over the
bytes depending on the type of the LHS.

**NC**: The implication of the above idea would be that assignment implies that the resulting value
is initialized. I believe we could commit to this separately from committing to whatever part of the
memory model we would need to decide on to make the above paragragh precise. Do we want to?

Assignments in which the types of the place and rvalue differ are not well-formed.

**NC**: Do we ever want to worry about non-free (in the body) lifetimes for the typing requirement
in post drop-elaboration MIR? I think probably not - I'm not sure we could meaningfully require this
anyway. How about free lifetimes? Is ignoring this interesting for optimizations? Do we want to
allow such optimizations? 

**NC**: We currently require that the LHS place not overlap with any place read as part of
computation of the RHS. This requirement is under discussion in [#68364][68364]. As a part of this
discussion, it is also unclear in what order the components are evaluated.

[68364]: https://github.com/rust-lang/rust/issues/68364

See the [rvalues](rvalues) page for details for each of those.

[rvalues]: rvalues.md

## `FakeRead`

Disallowed after drop elaboration. Semantically a nop.

## `SetDiscriminant`

Sets the discriminant of the place to the one corresponding to the given variant. For generators,
the behavior of this statment is documented below. For ADTs, the behavior is currently under
discussion [on Zulip][set-discriminant]. This statement is only allowed after deaggregation.

[set-discriminant]: https://rust-lang.zulipchat.com/#narrow/stream/189540-t-compiler.2Fwg-mir-opt/topic/SetDiscriminant.20and.20aggregate.20initialization.20.2394590

For generators, this writes to some of the bytes in the place that are not also within any of the
variant's fields.

## `StorageLive` & `StorageDead`

`StorageLive` and `StorageDead` statements mark the live range of a local. Using a local before a
`StorageLive` or after a `StorageDead` is either UB or not well-formed. These statements are not
required. If the entire MIR body contains no `StorageLive`/`StorageDead` statements for a particular
local, the local is always considered live.

More precisely, the MIR validator currently does a `MaybeLiveLocals` analysis to check validity of
each use of a local. I believe this is equivalent to requiring that there exist at least one path
from the root to each use of a local that contains a `StorageLive` more recently than a
`StorageDead`.

**NC**: Is that the actual rule we want, or just what the validator currently checks?

**NC**: Is it permitted to `StorageLive` a local for which we previously emitted `StorageDead`. How
about two `StorageLive`s without an intervening `StorageDead`? Two `StorageDead`s without an
intervening `StorageLive`? LLVM says yes, poison, yes.

## `Retag`

Only emitted when `-Zmir-emit-retag` is enabled, which is done by miri. The meaning of these
statements is specific to Stacked Borrows. For things that are not specific to Stacked Borrows, you
should consider these to be opaque modifications to the value.

Only allowed on references, pointers, and `Box`.

**NC**: Shouldn't retags only ever be emitted in the very last MIR pass? Doing otherwise might allow
other passes to remove them, thereby hiding some UB that would could be caught.

## `AscribeUserType`

Disallowed after drop elaboration.

**TODO**: The rest of this.

## `Coverage`

**TODO**: I don't know anything about this.

## `CopyNonOverlapping`

This is effectively an implementation of the associated intrinsic. First, all three operands are
evaluated. `src` and `dest` must each be a reference, pointer, or `Box` pointing to the same type
`T`. `count` must evaluate to a `usize`. Then, `src` and `dest` are dereferenced, and
`count * size_of::<T>()` bytes beginning with the first byte of the `src` place are copied to the
continguous range of bytes beginning with the first byte of `dest`.

**NC**: In what order are these things done? It should probably match the order for assignment, but
that is also undecided.

**NC**: Is this typed or not, ie is there a place to value and back conversion involved? I vaguely
remember Ralf saying somewhere that he thought it should not be.

## `Nop`

Does nothing.
