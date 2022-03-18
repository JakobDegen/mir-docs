First, a note on unwinding: Panics may occur during the execution of some terminators. Depending on
the `-C panic` flag, this may either cause the program to abort or the call stack to unwind. Such
terminators have a `cleanup: Option<BasicBlock>` field on them. If stack unwinding occurs, then once
the current function is reached, execution continues at the given basic block, if any. If `cleanup`
is `None` then no cleanup is performed, and the stack continues unwinding. This is equivalent to the
execution of a `Resume` terminator.

The basic block pointed to by a `cleanup` field must have its `cleanup` flag set. `cleanup` basic
blocks have a couple restrictions:
 1. All `cleanup` fields in them must be `None`.
 2. `Return` terminators are not allowed in them. `Abort` and `Unwind` terminators are.
 3. All other basic blocks (in the current body) that execution may reach from `cleanup` basic
    blocks must also be `cleanup`. This is a part of the type system and checked statically, so it
    is still an error to have such an edge in the CFG even if it's known that it won't be taken at
    runtime.

# By Terminator

## `Goto`

Continues execution at the specified basic block.

## `SwitchInt`

Switches based on the computed operand. First, evaluates the `discr` operand. The type of the
operand must be a signed or unsigned integer, char, or bool, and must match the given type. Then, if
the list of switch targets contains the computed value, continues execution at the associated basic
block. Otherwise, continues execution at the "otherwise" basic block.

Target values may not appear more than once. There must be an "otherwise" target.

## `Return`

Returns from the function. This assigns the value currently in the return place (`_0`) to the return
place specified in the associated `Call` terminator in the calling function, as if assigned via
`dest = move _0`.

If the body is a generator body, this has slightly different semantics; it instead causes a
`GeneratorState::Returned(_0)` to be created (by reading `_0`) and assigned to the return place.

## `Unreachable`

Indicates that this cannot be reached. Executing this terminator is UB.

## `Resume` & `Abort`

Indicates that the landing pad is finished and that the process should either continue unwinding or
abort, respectively. This is comparable to a return in that all locals are dead at this point.
However, unlike a return, the return place is also dead.

Only permitted in cleanup blocks. `Resume` is not permitted with `-C unwind=abort` after
deaggregation runs.

## `Call`

Roughly speaking, evaluates the `func` operand and the arguments, and starts execution of the
referred to function. The operand types must exactly match the argument types of the function. The
return place type must exactly match the return type. The type of the `func` operand must be
callable, meaning either a function pointer, a function type, or a closure type.

**NC**: The exact semantics of this, see [71117][71117].

[71117]: https://github.com/rust-lang/rust/issues/71117

## `Drop` & `DropAndReplace`

The behavior of these statements differs significantly before and after drop elaboration. After drop
elaboration, `Drop` executes the drop glue for the specified place, after which it continues
execution/unwinds at the given basic blocks. `DropAndReplace` is not permitted at this point. The
behavior of drop glue execution is undecided at the moment, and part of the memory model. (**TODO**:
due we have an issue tracking this in UCG?)

Before drop elaboration, `DropAndReplace` correspond to a `Drop`, followed unconditionally by an
assignment of the value to the place. This assignment happens both on the normal path and on the
unwind path.

`Drop` before drop elaboration is a *conditional* execution of the drop glue. Specifically, the
`Drop` will be executed if

**NC**: Clarify this. This in effect should document the exact behavior of drop elaboration. The following sounds vaguely right, but I'm not quite sure:

> The drop glue is executed if, among all statements executed within this `Body`, an assignment to
> the place or one of its "parents" occurred more recently than a move out of it. This does not
> consider indirect assignments.

## `Assert`

Evaluates the operand, which must have type `bool`. If it is not equal to `expected`, initiates a
panic. Initiating a panic corresponds to a `Call` terminator with some unspecified constant as the
function to call, all the operands stored in the `AssertKind` as parameters, and a `!` return type.
Keep in mind that the `cleanup` path is not necessarily executed even in the case of a panic, for
example in `-C panic=abort`. If there is no panic, execution continues at the specified basic block.

## `Yield`

Computes the operand and then a `GeneratorState::Yielded(value)`. That value is then assigned to the
return place of the function calling this one, and this function "returns." When next invoked,
execution of this function continues at the `resume` basic block, with the argument written to the
`resume_arg` place. If the generator is dropped before then, the `drop` basic block is invoked.

Not permitted in bodies that are not generator bodies, or after generator lowering.

**TODO**: What about the evaluation order of the `resume_arg` and `value`?

## `GeneratorDrop`

Semantically just a `return` (from the generators drop glue). Only permitted in the same situations
as `yield`.

**NC**: Is that even correct? The generator drop code is always confusing to me, because it's not
even really in the current body.

**NC**: Are there type system constraints on these terminators? Should there be a "block type" like
`cleanup` blocks for them?

## `FalseEdge` & `FalseUnwind`

Both disallowed after drop elaboration. Semantically gotos to the real target.

## `InlineAsm`

Executes some assembly. The assembly operands are evaluated in the given order.

**TODO**: I don't know enough about inline asm to fill in the rest of this.

**NC**: Are there restrictions on what places may appear?
