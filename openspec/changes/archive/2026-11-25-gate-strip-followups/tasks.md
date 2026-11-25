# Tasks

- [x] 1.1 `GateRemover`: `isSyntheticLambdaName` matches `lambda$...` and `...$lambda$...` (constructor-lambda shape); used by gate-lambda removal and the orphan sweep. Verify: `./gradlew test` in `jitter-plugin` passes.
- [x] 1.2 Orphan-lambda sweep runs to a fixpoint and also collects classes instantiated (`NEW`) in the removed lambdas. Verify: `GateRemoverTest` cases pass.
- [x] 1.3 Orphan-class sweep ignores self-references of the candidate class. Verify: `namedNestedClassInsideDoubleNestedLambdaOfRemovedGateIsRemoved` passes.
- [x] 1.4 `JvmKotlinMetadataRewriter.mutateKmClass`: nestedClasses cleanup matches full internal names too. Verify: `./gradlew test` in `jitter-plugin` passes.
- [x] 2.1 `./gradlew :acceptancetest:process:jitterVerify` in ologis (Bmx) passes.
