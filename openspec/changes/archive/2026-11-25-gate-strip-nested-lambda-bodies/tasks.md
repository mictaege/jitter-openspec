# Tasks

## 1. Recursive lambda-body removal in GateRemover

- [x] 1.1 In `GateRemover.removeLambdaBody`, before removing a synthetic lambda method, collect
  nested `invokedynamic` lambda handles from its instructions and remove those methods
  transitively (recording each in `removedLambdas` / `removedMembers` as today).
  Verify: `./gradlew compileJava` in `~/projects/jitter-plugin` succeeds.

- [x] 1.2 Add a `GateRemoverTest` case: a gated block containing a nested lambda that references
  an `@OnlyIf`-removed class; after stripping for a non-listed flavour,
  `ReferenceVerifier.findDanglingReferences` is empty and both `lambda$` methods are gone.

## 2. Gate-local anonymous class removal

- [x] 2.1 In `GateRemover`, collect anonymous inner classes instantiated (`NEW`) inside removed
  gate lambda bodies; after `removeGates`, remove each candidate that no surviving bytecode
  references anymore (via `OnlyIfRemover.removeClass`).

- [x] 2.2 Add a `GateRemoverTest` case: a gated block instantiating an anonymous class whose
  method references an `@OnlyIf`-removed member; after stripping, the anonymous class is gone
  and verification is clean. Add a counter-case: an anonymous class still referenced from
  surviving code is kept.

## 3. Integration verification

- [x] 3.1 Run the full `jitter-plugin` suite: `./gradlew test` in `~/projects/jitter-plugin`.
- [x] 3.2 Run `./gradlew :acceptancetest:process:jitterVerify` in `~/projects/ologis` (flavour Bmx):
  the `DefaultOrderFlowKtTest` dangling references are gone.
