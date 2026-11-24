# block-gates Specification

## Purpose

Block-level flavour gating: mark a block of statements inside a method or lambda body as
belonging to specific flavours, so the stripped flavour model removes the whole block (call site
and synthetic body) from the bytecode, for Java and Kotlin alike.

## Requirements

### Requirement: Block-gate API in jitter-api

jitter-api SHALL provide a static gate entry point `Gate.onlyIf(List<String> flavours, Runnable
block)` usable inside any method or lambda body. In the 150% model the block SHALL execute
normally. The gate marker SHALL have CLASS retention so it is visible to the stripping tool but
never to the runtime.

#### Scenario: Gate executes in the 150% model
- **WHEN** code calls the gate with a flavour list and a block
- **THEN** in the unstripped (150%) build the block runs exactly once, in place

#### Scenario: Gate call compiles in Java and Kotlin
- **WHEN** the gate is used with a Java lambda and with a Kotlin lambda
- **THEN** both compile against jitter-api unchanged (no Kotlin-specific artifact required)

### Requirement: Stripping removes gated blocks per flavour

For each flavour that is NOT in the gate's flavour list, the stripping task SHALL remove from the
compiled bytecode: the gated call site including its argument-loading instructions, and the
synthetic method that implements the block (lambda body / anonymous class). Surrounding code
SHALL remain valid bytecode (stack map frames recomputed, no dangling labels).

#### Scenario: Gated block removed in non-listed flavour
- **WHEN** a method contains `Gate.onlyIf(List.of("ESA"), () -> ...)` and the build runs flavour NASA
- **THEN** the stripped NASA class contains no trace of the block: no call site, no synthetic
  lambda method, and the method behaves as if the statement were never written

#### Scenario: Gated block kept in listed flavour
- **WHEN** the same class is stripped for flavour ESA
- **THEN** the block is present and executes normally

#### Scenario: Kotlin DSL builder call gated
- **WHEN** an arete-ktx-style spec lambda contains a gated `scenario("...") { ... }` call
- **THEN** in a flavour not listed, the scenario does not appear in the executed spec tree nor in
  the stripped class

### Requirement: Verification tolerates references inside removed blocks

The dangling-reference verification SHALL NOT report references that occur only inside a block
removed for the active flavour (e.g. a flavour-only class used solely within a gated block).

#### Scenario: Flavour-only type used only inside gated block
- **WHEN** a gated block references a class that is itself `@OnlyIf`-stripped in the same flavour
- **THEN** verification passes for that flavour

### Requirement: Dejittering resolves gated blocks per flavour

The dejittered source is the source of the active flavour alone: a gate whose flavour list does
not include the active flavour SHALL be removed as a whole, and a gate that includes it SHALL be
unwrapped to its block body, so no `Gate.onlyIf` call survives in the dejittered source. With no
active flavour (the 150% variant) the gates stay unchanged.

#### Scenario: Active gate unwrapped in the dejittered source
- **WHEN** sources with a `Gate.onlyIf(listOf(Flavour.ESA)) { ... }` block are dejittered for ESA
- **THEN** the dejittered source contains the block body without the `Gate.onlyIf` wrapper

#### Scenario: Inactive gate removed from the dejittered source
- **WHEN** the same sources are dejittered for NASA
- **THEN** the gated statement is gone from the dejittered source

### Requirement: Kotlin metadata stays consistent

When stripping removes a gated block whose lambda body is referenced from a Kotlin class's
`@kotlin.Metadata`, the metadata rewrite SHALL remove the corresponding synthetic function
reference, consistent with the existing metadata-rewrite rules for removed declarations.

#### Scenario: Stripped lambda body dropped from metadata
- **WHEN** a Kotlin class contains a gated lambda and is stripped
- **THEN** `@kotlin.Metadata` of the surviving class no longer references the removed synthetic
  lambda function, and verification of the metadata passes
