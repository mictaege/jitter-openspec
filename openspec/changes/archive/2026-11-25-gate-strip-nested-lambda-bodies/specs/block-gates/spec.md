# Delta for block-gates

## ADDED Requirements

### Requirement: Stripping removes nested lambda bodies of a removed gate block

When a `Gate.onlyIf` block is removed for the active flavour, the stripping SHALL also remove
every synthetic lambda method that the removed block's lambda body references through further
`invokedynamic` instructions, transitively (lambdas nested at any depth inside the gated block).

#### Scenario: Nested lambda inside a removed gate block
- **WHEN** a gated block contains a nested lambda whose body references a class that is
  `@OnlyIf`-stripped in the active flavour
- **THEN** the stripped class contains neither the nested synthetic lambda method nor any
  dangling reference to the removed class, and verification passes

### Requirement: Stripping removes gate-local anonymous classes

When a removed `Gate.onlyIf` block instantiates an anonymous (inner) class of the enclosing
top-level class, and no surviving bytecode references that anonymous class anymore, the stripping
SHALL remove the anonymous class as a whole (recorded in the removal manifest). If surviving code
still references it, the class SHALL be kept.

#### Scenario: Anonymous class created only inside a removed gate block
- **WHEN** a gated block contains `new SomeAnonymous() { ... }` whose method body references a
  member removed in the active flavour, and nothing outside the gate references the anonymous
  class
- **THEN** the anonymous class file is removed and verification passes

#### Scenario: Anonymous class still referenced from surviving code
- **WHEN** an anonymous class instantiated inside a removed gate block is still referenced from
  code that survives the stripping
- **THEN** the anonymous class is kept unchanged
