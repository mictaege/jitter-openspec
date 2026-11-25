# Design

## Context

The Kotlin source dejitterer (`KotlinDejitterer`) already prunes jitter API imports via `pruneImports`, but unlike the Java dejitterer (`Dejitterer.pruneImports`), it does not remove imports of classes that were removed in the active flavour. See proposal.md for the full motivation (issue #51).

## Goals / Non-Goals

- **Goal**: Make Kotlin dejittered output symmetric with Java dejittered output regarding import pruning — any import that is no longer used after removals must be dropped.
- **Goal**: Use the removal manifest (`StripResult.getRemovedClasses()`) so that imports of classes removed via package-level `@OnlyIf` (which are recorded as removed classes, not as edits in the importing file) are also caught.
- **Non-Goal**: Change the Java dejitterer — it already prunes unused imports correctly.
- **Non-Goal**: Change stripping, verification, or the annotation API.

## Decisions

### Decision: Prune by usage check, mirroring the Java dejitterer

Extend `KotlinDejitterer.pruneImports` to drop any non-jitter import whose simple name no longer appears in the file body (outside import lines), the same approach `Dejitterer.pruneImports` uses for Java. Additionally, cross-check against `StripResult.getRemovedClasses()` to catch imports of classes removed via package gates, where the simple name might coincidentally appear in a comment or string.

**Rationale**: The Java dejitterer's usage-based approach works and is already tested. Adding the removal-manifest cross-check makes the Kotlin path robust against false positives (e.g. a simple name appearing in a comment).

**Alternatives considered**:
- Only cross-check against the removal manifest (no usage check): would miss imports that become unused because the referencing declaration was removed in the same file but the class itself is not in the removal manifest (e.g. a `@OnlyIf` constructor parameter type that is not itself flavour-gated). The usage check catches these.

### Decision: Pass the removal manifest into pruneImports

`pruneImports` currently receives only the text and the import list. It will additionally receive the `StripResult` (or just the `Set<String>` of removed class internal names) so it can resolve an import's fully qualified name to an internal name and check membership.

**Rationale**: The removal manifest is already built up during dejittering; passing it in is a minimal change. The import's FQN is available from the import directive text.

## Risks / Trade-offs

- [Risk] A simple name that coincidentally appears in a string literal or comment keeps an otherwise-unused import → Mitigation: cross-check against the removal manifest for classes known to be removed; the usage check alone is a superset of what the Java dejitterer does, so this is no worse than the status quo.
- [Risk] Star imports (`import foo.*`) are not handled — they are already not handled by the Java dejitterer either → Mitigation: out of scope; consistent with existing behaviour.

## Migration Plan

No migration needed — this is a bug fix in the dejitter task output. Existing builds that worked around it via `removedClasses` in `verifyCriticalTerms` can drop the workaround.

## Open Questions

None.
