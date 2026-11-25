# Spec Delta

## Purpose

Source-level dejittering produces compilable, flavour-pure source variants: flavour-gated declarations are removed and imports referencing removed classes are pruned, so the dejittered output compiles and passes critical-terms verification.

## ADDED Requirements

### Requirement: Kotlin dejittering drops imports of removed classes

When the Kotlin source dejitterer removes a declaration in the active flavour, it SHALL also remove `import` statements whose imported class was removed in the same flavour (as listed in the removal manifest), and any import whose simple name no longer occurs anywhere in the remaining file body.

#### Scenario: Import of a package-gated class is dropped
- **WHEN** a Kotlin file imports a class from a package gated with `@OnlyIf(Flavour.B)` and the active flavour is A
- **THEN** the dejittered output for flavour A contains no `import` of that class

#### Scenario: Import of a class-gated class is dropped
- **WHEN** a Kotlin file imports a class annotated with `@OnlyIf(Flavour.B)` and the active flavour is A
- **THEN** the dejittered output for flavour A contains no `import` of that class

#### Scenario: Import still used by surviving code is kept
- **WHEN** a Kotlin file imports a class that is also referenced by a declaration that survives in the active flavour
- **THEN** the import remains in the dejittered output

#### Scenario: Java dejittering already prunes unused imports
- **WHEN** a Java file's flavour-gated declaration is removed and its import becomes unused
- **THEN** the import is dropped (existing behaviour, no regression)
