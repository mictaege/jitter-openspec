# Delta for block-gates

## ADDED Requirements

### Requirement: Synthetic lambda detection covers constructor-lambda naming

Synthetic lambda methods SHALL be recognized by name `lambda$...` as well as any name containing
`$lambda$` (kotlinc names lambdas nested in a constructor lambda `_init_$lambda$...`), so gated
blocks inside constructor lambdas (e.g. `Spec({ spec { ... } })`) are stripped like any other.

#### Scenario: Gate inside a constructor lambda
- **WHEN** a `Gate.onlyIf` block sits inside a lambda passed to a super constructor (arete-ktx
  `Spec({ spec { ... } })`) and the active flavour is not listed
- **THEN** the gate span, all its synthetic lambda bodies (any nesting depth), and gate-local
  instantiated classes are removed, and verification passes

### Requirement: Orphan sweeps ignore self-references

The reference check that decides whether a gate-local class or lambda may be removed SHALL ignore
references from the candidate itself (e.g. its own field stores), so gate-local classes are
actually removed.

#### Scenario: Gate-local class with fields
- **WHEN** a removed gate block instantiates a class whose own `<init>` stores fields on itself
- **THEN** the class is still recognized as unreferenced and removed

### Requirement: Kotlin metadata cleanup matches full nested-class names

The `@kotlin.Metadata` rewrite SHALL remove `nestedClasses` entries of removed classes whether they
are stored as simple names or as full internal names.

#### Scenario: Removed nested class in metadata
- **WHEN** a nested class is removed by gate stripping and the surviving outer class's
  `@kotlin.Metadata` lists it
- **THEN** the entry is removed from the metadata and metadata verification passes
