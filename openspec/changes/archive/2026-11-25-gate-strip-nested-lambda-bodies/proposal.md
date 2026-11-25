# Proposal

## Why

`GateRemover.removeLambdaBody` removes only the *direct* synthetic `lambda$...` method of a
removed `Gate.onlyIf` block. Lambdas nested inside the gated block (e.g. a `step { ... }` lambda
inside a gated `journey { ... }` lambda in an arete-ktx spec, or a `matching { ... }` lambda inside
a gated step body) compile to their own synthetic `lambda$...` methods, which survive the
stripping together with their references to flavour-removed classes/fields. `jitterVerify` then
fails with dangling references such as
`DefaultOrderFlowKtTest references removed class OrderFlowMatchers$Mfr2OlogisJob`, even though
every reference sits inside a `Gate.onlyIf` block that does not list the active flavour.
Anonymous classes instantiated inside a removed gate block (e.g. a Kotlin `object :` expression
in a gated journey) survive as well and keep their references to removed members.

## What Changes

- `GateRemover` SHALL strip a removed gate block *recursively*: when a gated call is removed,
  the synthetic lambda methods reachable from the removed lambda body via further
  `invokedynamic` instructions SHALL be removed as well, transitively.
- `GateRemover` SHALL also remove anonymous (inner) classes that are instantiated only inside
  removed gate blocks: after all gates are processed, an anonymous class of the enclosing
  top-level class that is no longer referenced from any surviving bytecode SHALL be removed as a
  class (recorded in the `StripResult` removal manifest like `@OnlyIf`-removed classes).
- Verification SHALL then pass for code whose flavour-specific references live exclusively
  inside (possibly nested) removed gate blocks.

## Capabilities

### Modified Capabilities
- `block-gates`: stripping of `Gate.onlyIf` blocks must cover nested lambda bodies and gate-local
  anonymous classes, not just the direct lambda method.

## Impact

- `jitter-plugin`: `GateRemover.java` (recursive lambda-body removal, orphaned anonymous-class
  sweep) and `GateRemoverTest.java` (regression tests for nested lambdas and anonymous classes in
  removed gates).
- No changes to `jitter-api`, `jitter` (schema repo), or `eval.jitter`.
- Unblocks the arete-ktx Kotlin DSL tests in ologis (`DefaultOrderFlowKtTest`), whose gated
  journeys/steps nest lambdas inside `Gate.onlyIf` blocks.
