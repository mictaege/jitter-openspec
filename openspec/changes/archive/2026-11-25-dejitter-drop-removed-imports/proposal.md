# Proposal

## Why

`dejitter<SrcSet>` removes flavour-gated Kotlin declarations correctly, but keeps the `import` statements of classes that only the removed code used. When those classes are themselves removed in the active flavour (e.g. via a package-level `@OnlyIf`), the dejittered source does not compile and `verifyCriticalTerms` fails on source deliveries. The Java dejitterer already prunes unused imports; the Kotlin dejitterer only prunes jitter API imports.

## What Changes

- `KotlinDejitterer.pruneImports` SHALL also drop imports whose imported class was removed in the active flavour (as recorded in the `StripResult` removal manifest), and imports that are no longer referenced anywhere in the remaining file body.
- The behaviour SHALL be symmetric with the Java dejitterer's existing `pruneImports`, which already removes any import whose simple name no longer occurs in the file.

## Capabilities

### New Capabilities
- `source-dejittering`: Source-level dejittering of Java and Kotlin files for the active flavour, including removal of flavour-gated declarations and pruning of imports that reference removed classes.

### Modified Capabilities

## Impact

- `jitter-plugin`: `KotlinDejitterer.java` (the `pruneImports` method and its call sites to pass the removal manifest).
- Tests in `jitter-plugin`: `KotlinDejittererTest.java` and/or `JitterDejitterTaskTest.java` get new cases covering imports of package-gated and class-gated removed classes.
- No changes to `jitter-api`, `jitter` (schema repo), or `eval.jitter`.
