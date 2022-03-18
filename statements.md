# By Statement

## `Assign`

Assign statements roughly correspond to an assignment in Rust proper (`x = ...`) except without the
possibility of dropping the previous value (that must be done separately, if at all). The *exact*
way this works is undecided. Various parts of this may do type specific things that are more
complicated than simply copying over the bytes depending on the type of the LHS.

Assignments in which the types of the LHS and RHS differ are not well-formed.

**NC**: Do we ever want to worry about non-free lifetimes for the typing requirement in post
drop-elaboration MIR? I think probably not - I'm not sure we could meaningfully require this anyway.
How about free lifetimes? Is ignoring this interesting for optimizations? Do we want to allow such
optimizations? 

**NC**: We currently require that the LHS place not overlap with any place read as part of
computation of the RHS. This requirement is under discussion in [#68364][68364]. As a part of this
discussion, it is also unclear in what order the components are evaluated.

[68364]: https://github.com/rust-lang/rust/issues/68364

**NC**: Closely related to the question about typed-ness in the places and operands section: are the
assignment statements *themselves* typed, or do they just copy memory? Note that this would not
imply that assignments in Rust are untyped, since we still have to evaluate the operands, and that
step might be typed even if this one is not.

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
variant's fields. **NC**: Which bytes? Only the necessary ones, or all of them?

## `StorageLive` & `StorageDead`

`StorageLive` and `StorageDead` statements mark the live range of a local. Using a local before a
`StorageLive` or after a `StorageDead` is either UB or not well-formed (see below). These statements
are not required. If the entire MIR body contains no `StorageLive`/`StorageDead` statements for a
particular local, the local is always considered live.

**NC**: What does "before"/"after" mean? Three options as far as I can see:

 1. Storage statements are "static," meaning that there may be no path in the CFG from the root to a
    use of the local that does not contain a `StorageLive` statement, and similarly there may be no
    path from the root to a use of the local that does contain a `StorageDead` statement. In this
    case, breaking such a rule is checkable by the validator and should be considered not
    well-formed.
 2. Storage statements are "dynamic," ie they are executed at runtime. Breaking the liveness rules
    imposed by the statements is UB. This means that dead code may "break" the requirements that
    would be imposed by the static rules.
 3. Some combination of the above? i.e. storage statements are dynamic and executed at runtime, but
    we additionally require them to pass some reduced amount of static analysis.

We could also make the semantics dependent on MIR phase, eg option 1 before drop-elaboration, and
option 3 after.

Additionally, if we choose option 2 or 3: Is it permitted to `StorageLive` a local for which we
previously emitted `StorageDead`. How about two `StorageLive`s without an intervening `StorageDead`?
Two `StorageDead`s without an intervening `StorageLive`? LLVM says yes, poison, yes.

## `Retag`

Only emitted when `-Zmir-emit-retag` is enabled, which is done by miri. The meaning of these
statements is specific to Stacked Borrows. For things that are not specific to Stacked Borrows, you
should consider these to be both reads and writes to the value.

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

**NC**: Is this a typed copy or not? I vaguely remember Ralf saying somewhere that he thought it
should not be.

## `Nop`

Does nothing.
