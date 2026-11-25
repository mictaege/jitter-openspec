# Tasks

## 1. Extend KotlinDejitterer.pruneImports

- [x] 1.1 Modify `KotlinDejitterer.pruneImports` to also drop non-jitter imports whose simple name no longer appears in the file body outside import lines, and whose fully qualified name (converted to internal form) is in the `StripResult.getRemovedClasses()` set. Pass the `StripResult` (or the removed-class set) into `pruneImports` from both call sites in `dejitterUnderPsi`. Verify: `./gradlew compileJava` in `jitter-plugin` succeeds.

- [x] 1.2 Add a unit test in `KotlinDejittererTest` reproducing issue #51: a Kotlin file with two `@OnlyIf`-gated constructors referencing classes from package-gated packages, dejittered for flavour A — assert the `repro.b.BDao` import is absent from the output. Verify: `./gradlew test --tests KotlinDejittererTest` in `jitter-plugin` passes.

- [x] 1.3 Add a unit test for a class-level `@OnlyIf` (annotation on the class itself, not the package) whose import becomes unused after dejittering — assert the import is dropped. Verify: `./gradlew test --tests KotlinDejittererTest` in `jitter-plugin` passes.

- [x] 1.4 Add a test that an import still used by surviving code is kept. Verify: `./gradlew test --tests KotlinDejittererTest` in `jitter-plugin` passes.

## 2. Integration verification

- [x] 2.1 Run the full `jitter-plugin` test suite: `./gradlew test` in `~/projects/jitter-plugin`. Verify: all tests pass, no regressions in `DejittererTest` or `JitterDejitterTaskTest`.

- [x] 2.2 Run `./gradlew build` in `~/projects/eval.jitter` with `-Djitter.active.flavour=ESA` to confirm end-to-end dejittering still works. Verify: build succeeds and dejittered sources contain no imports of removed flavour classes.
