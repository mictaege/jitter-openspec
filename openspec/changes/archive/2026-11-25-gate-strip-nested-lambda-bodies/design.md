# Design

## Context

A `Gate.onlyIf(flavours, block)` call compiles (Java and Kotlin alike) to a flavour-list constant,
an `invokedynamic` for the lambda body, and the `invokestatic` of the gate. `GateRemover` removes
that span for non-listed flavours and then removes the synthetic `lambda$...` method the
`invokedynamic` points at (`removeLambdaBody`). Anything the lambda body itself references through
further `invokedynamic` instructions (nested lambdas) or `new`/`invokespecial <init>` of an
anonymous class stays in the bytecode, so `ReferenceVerifier` reports dangling references to
flavour-removed members from those leftover bodies.

## Decision

Make the removal transitive instead of teaching the verifier to tolerate the leftovers:

1. **Recursive lambda removal.** When `removeLambdaBody` drops a lambda method, it first scans
   the method's instructions for `InvokeDynamicInsnNode`s whose bootstrap-method arguments contain
   a `Handle` owned by the same class with a `lambda$` name, and removes those methods the same
   way (transitively). This mirrors what `javac`/`kotlinc` produce for nested lambdas: each nesting
   level is its own synthetic method referenced from the enclosing one.
2. **Gate-local anonymous classes.** While removing the lambda bodies, collect `NEW` instructions
   whose type is an anonymous inner class of the enclosing top-level class
   (`<Enclosing>$<n>...`). After all gates of all classes are processed, each candidate is removed
   as a whole (via `OnlyIfRemover.removeClass`, which also drops its nested classes and records it
   in `StripResult.removedClasses`) iff no surviving class references it anymore — checked by
   scanning all remaining methods for method/field owners, `TypeInsnNode`, `LDC` `Type`,
   `MultiANewArrayInsnNode` and `invokedynamic` handles mentioning the candidate's internal name.
   The safety scan is what keeps the removal conservative: if any surviving code still references
   the anonymous class, it is kept (and verification reports it, as before).

## Consequences

- Removed gate blocks leave no trace: no call site, no direct or nested synthetic lambda method,
  no gate-local anonymous class — matching the spec's "as if the statement had never been written".
- The verifier needs no changes; it already skips classes recorded in `removedClasses`.
- Kotlin metadata rewriting already handles removed members via the `removedMembers` manifest; the
  recursively removed lambdas are recorded there exactly like direct ones.
