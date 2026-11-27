# Delta for block-gates

## ADDED Requirements

### Requirement: Value-returning gate API in jitter-api

jitter-api SHALL provide a static gate entry point `Gate.getOnlyIf(List<String> flavours,
Supplier<T> block, Supplier<T> orElse)` returning `T`, under a name distinct from the void
`Gate.onlyIf` so existing `Runnable` call sites keep their compiled descriptor and Kotlin
trailing lambdas bind to the intended overload. In the 150% model the method SHALL return
`block.get()` and SHALL NOT invoke `orElse`.

#### Scenario: Gate returns the block's value in the 150% model
- **WHEN** code calls `getOnlyIf` with a flavour list, a block and an orElse supplier
- **THEN** in the unstripped (150%) build the block runs exactly once, its value is returned,
  and the orElse supplier is never invoked

#### Scenario: Gate call compiles in Java and Kotlin
- **WHEN** the gate is used with a Java lambda / constructor method reference and with a
  Kotlin lambda (`Gate.getOnlyIf(listOf("ESA"), Supplier { "EU" }) { "NONE-EU" }`)
- **THEN** both compile against jitter-api unchanged, and the common result type is inferred
  from both branches (e.g. `var policy = getOnlyIf(list, EsaPolicy::new, OtherPolicy::new)`
  infers the common supertype `Policy`)

### Requirement: Stripping selects one branch of a value gate per flavour

For each `Gate.getOnlyIf` call, the stripping SHALL keep exactly one branch per flavour: the
`block` branch when the active flavour is listed, otherwise the `orElse` branch. The discarded
branch SHALL leave no trace in the stripped bytecode (no argument instructions, no
`invokedynamic`, no synthetic lambda method), and the surviving branch SHALL produce the gated
value.

#### Scenario: orElse branch stripped in a listed flavour
- **WHEN** a method contains `getOnlyIf(List.of("ESA"), () -> "EU", () -> "NONE-EU")` and the
  build runs flavour ESA
- **THEN** the stripped class returns "EU" and contains no trace of the orElse branch: no
  second `invokedynamic`, no synthetic lambda method for it

#### Scenario: block branch stripped in a non-listed flavour
- **WHEN** the same class is stripped for flavour NASA
- **THEN** the stripped class returns "NONE-EU" and contains no trace of the block branch

#### Scenario: Constructor method reference branch needs no synthetic method
- **WHEN** a value gate uses constructor references (`getOnlyIf(List.of("ESA"), EsaPolicy::new,
  OtherPolicy::new)`)
- **THEN** stripping removes the unselected branch's `invokedynamic` (which holds the only
  reference to the unselected constructor) without needing to remove any synthetic lambda
  method

#### Scenario: Nested value gate in the orElse branch
- **WHEN** the orElse argument is itself a `getOnlyIf` call (three-way flavour split)
- **THEN** each gate call site is resolved independently, so in every flavour exactly one
  leaf branch survives

### Requirement: Verification tolerates references inside the stripped branch

The dangling-reference verification SHALL NOT report references that occur only inside the
branch of a value gate that is stripped for the active flavour (e.g. a flavour-only policy
class referenced only from the discarded constructor-method-reference branch).

#### Scenario: Flavour-only type referenced only from the stripped branch
- **WHEN** `getOnlyIf(List.of("ESA"), EsaPolicy::new, OtherPolicy::new)` is stripped for ESA
  while `OtherPolicy` is itself `@OnlyIf`-stripped in ESA
- **THEN** verification passes for that flavour

### Requirement: Dejittering resolves value gates to the selected branch

The dejittered source is the source of the active flavour alone: a `getOnlyIf` call SHALL be
replaced by the selected branch's source — `block` when the active flavour is listed, `orElse`
otherwise — so no `getOnlyIf` call survives in the dejittered source. With no active flavour
(the 150% variant) the gates stay unchanged.

#### Scenario: Listed branch unwrapped in the dejittered source
- **WHEN** sources with `getOnlyIf(listOf(Flavour.ESA), { "EU" }, { "NONE-EU" })` are dejittered
  for ESA
- **THEN** the dejittered source contains `"EU"` in place of the whole call

#### Scenario: Non-listed branch replaced by orElse in the dejittered source
- **WHEN** the same sources are dejittered for NASA
- **THEN** the dejittered source contains `"NONE-EU"` in place of the whole call

#### Scenario: Constructor method reference unwrapped in the dejittered source
- **WHEN** sources with `getOnlyIf(listOf(Flavour.ESA), EsaPolicy::new, OtherPolicy::new)` are
  dejittered for ESA
- **THEN** the dejittered source contains `new EsaPolicy()` in place of the whole call
