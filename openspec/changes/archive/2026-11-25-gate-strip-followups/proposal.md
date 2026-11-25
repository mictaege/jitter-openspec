# Proposal

## Why

Follow-up fixes to `2026-11-25-gate-strip-nested-lambda-bodies`, found while running the arete-ktx
port (`DefaultOrderFlowKtTest`) in ologis against flavour Bmx:

1. Lambdas nested in a *constructor* lambda are named `_init_$lambda$...` by kotlinc, not
   `lambda$...`; the gate stripping only matched `lambda$`, so gated blocks inside
   `Spec({ spec { ... } })` were not stripped.
2. The reference-checked orphan sweep for gate-local classes counted the candidate's own
   fields/methods (e.g. `PUTFIELD` with the candidate as owner) as references, so gate-local
   classes were never removed.
3. The Kotlin metadata rewrite cleaned `nestedClasses` only under the assumption of simple names;
   full internal names survived and `jitterVerify` flagged the metadata of surviving classes.

## What Changes

- `GateRemover` SHALL treat a synthetic lambda method as any method whose name starts with
  `lambda$` or contains `$lambda$`, and SHALL remove *all* synthetic lambda methods that no
  surviving bytecode references anymore (fixpoint sweep), not only those reachable by following
  the `invokedynamic` chain of the gated call.
- The orphan-class sweep SHALL ignore references from the candidate class itself.
- `JvmKotlinMetadataRewriter` SHALL remove `nestedClasses` entries matching removed classes by
  full internal name as well as by simple name.

## Capabilities

### Modified Capabilities
- `block-gates`: as above.

## Impact

- `jitter-plugin`: `GateRemover.java`, `JvmKotlinMetadataRewriter.java`, `GateRemoverTest.java`.
- No changes to `jitter-api`, `jitter`, or `eval.jitter`; nothing in ologis.
