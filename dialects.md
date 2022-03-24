# Dialects

MIR goes through a number of [phases][phases] in the compiler, and the semantics of MIR at each
phase may change. The section below summarizes the changes, but does not document them thoroughly.
The full documentation is found in the appropriate section for the thing the change is affecting.

[phases]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/enum.MirPhase.html

## Pre drop-elaboration

This is the dialect of MIR that is used during `Build`, `Const`, and `ConstPromotion`. This is also
the MIR that analysis such as borrowck uses.

One important thing to remember about the behavior of this section of MIR is that drop terminators
(including drop and replace) are *conditional*. The elaborate drops pass will then replace each
instance of a drop terminator with a nop, an unconditional drop, or a drop conditioned on a drop
flag. Of course, this means that it is important that the drop elaboration can accurately recognize
when things are initialized and when things are de-initialized. That means any code running on this
version of MIR must be sure to produce output that drop elaboration can reason about. See the
section on the drop terminatorss for more details.

## Drop elaboration

At this point, the post borrow check cleanup has run and a bunch of things are removed:

* `TerminatorKind::DropAndReplace`
* `TerminatorKind::FalseUnwind`
* `TerminatorKind::FalseEdge`
* `StatementKind::FakeRead`
* `StatementKind::AscribeUserType`
* `Rvalue::Ref` with `BorrowKind::Shallow`

## Deaggregation

Deaggregating removes `Rvalue::Aggregates` that are not arrays or generators and potentially
replaces them with `SetDiscriminant`s.

## Generator lowering

Before this phase, generators are in the "source code" form, featuring `yield` statements, and such.
With this phase change, they are transformed into a proper state machine. Running optimizations
before this change can be potentially dangerous because the source code is to some extent a "lie."
In particular, `yield` terminators effectively make the value of all locals visible to the caller.
This means that dead store elimination before them, or code motion across them, is not correct in
general. This is also exasperated by type checking having pre-computed a list of the types that it
thinks are ok to be live across a yield point - this is necessary to decide whether autotraits are
implemented. Introducing new types across a yield point will lead to ICEs becaues of this.

Once this has run, generator aggregates are also removed, and yield statements are gone.
