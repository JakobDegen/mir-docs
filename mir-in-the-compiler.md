Places where I know there is something missing that I just haven't written yet are marked TODO. If I
tried to fill these in, but did not come to a clear conclusion about what to put (sometimes after
only limited research), I've instead marked them  with "NC" (needs clarification). It's quite
possible that many of these things do have clear decisions, and I just don't know about them.

In many cases, MIR semantics depend on Rust semantics. For example, the rules that apply to aliasing
in MIR are the same ones that apply to aliasing in Rust. However, much of Rust's semantics
(including the aliasing model) are undecided. Appearences of these kinds of uncertainties are not
marked "NC" because they do not represent clarifictions that MIR needs - they instead represent
clarifications that Rust needs. Where possible I have linked to UCG or rust-lang/rust issues that
are relevant.

"Rules" for MIR come in two flavors: 1) The specification of behavior of MIR on the AM, and 2) the
rules that MIR must obey in addition the rules inherent to the AM. The second of these is generally
called the "MIR type system." To give some concrete examples:

Consider an assignment like `_2 = _3`, where `_2` has type `i32`. The Rust AM (let's pretend anyway,
more on this later) requires that integer operations, including reading and writing them, be done
with initialized integers; doing otherwise is UB. As such, the same applies to MIR, and executing
such an assignment when `_3` is uninit is UB. This type of rule is of the first kind mentioned
above.

At the same time, Rust allows you to read the memory of a `u32` as an `i32`, for example like this:

```rust
let x: u32 = 100;
let p: *const i32 = &x as *const u32 as *const i32;
let y: i32 = unsafe { *p };
```

You might be tempted to optimize such code to `y = x` in the MIR. However, the MIR type system
requires that assignments be between matching types, and so such MIR would be invalid. This is an
example of the second type of rule; this operation would be legal in the AM (under it's natural
interpretation), but the MIR type system forbids it.

In general, when we say that something is "undefined behavior," we mean that it breaks the first
type of rule. A program exhibiting undefined behavior has no particular behavior when executed.  On
the other hand, a program that is "not well-formed" or contains an "error" breaks the second type of
rule. It is - in principle - acceptable to produce MIR that exhibits undefined behavior - of course,
you should not introduce UB where there was none in the past, as that will cause the user's code to
be miscompiled. You should never produce MIR that is not well-formed, as other parts of the compiler
might ICE if that happens. Along the same vein, you may receive MIR that possibly exhibits UB. You
should not turn such UB into MIR that is not well-formed.
