# Proposal

## Why

Jitter can strip flavour-specific code only at declaration level (`@OnlyIf` on packages, types,
fields, constructors, methods). DSLs that build behaviour with type-safe builder calls inside a
method or lambda body — e.g. the new arete-ktx Kotlin DSL
(`class BookingProcessSpec : Spec({ spec("...") { scenario("...") { ... } } })`) — have no way to
mark an individual `scenario(...)`/`given { ... }` block as flavour-specific, because there is no
declaration to annotate. Statement/block-level gating is a general need for both Java and Kotlin
consumers, not just arete.

## What Changes

- **jitter-api**: new block-gate API, e.g. `com.github.mictaege.jitter.api.Gate.onlyIf(List<String>
  flavours, Runnable block)` so a block of statements
  can be marked flavour-specific inside any method or lambda. In the 150% model the block executes
  normally; the annotation/retention is CLASS so it never leaks into runtime.
- **jitter-plugin**: stripping removes, per flavour, the whole gated call site (argument setup +
  `invokedynamic`/`invokestatic` + the synthetic lambda/method body it points to) from the bytecode
  of every class, in addition to today's declaration-level stripping. Works for Java and Kotlin
  lambdas alike because both compile to a synthetic body method plus a call site.
- **jitter-plugin verification**: references that occur only inside a stripped block SHALL be
  tolerated (not reported as dangling) in flavours where the block is removed, mirroring how
  declaration-level removals are handled today.
- **jitter-plugin dejittering**: dejittered sources SHALL keep the gated block visible (like other
  gated code) so the 150% sources stay readable.
- **eval.jitter**: a demo usage showing a gated block in Java and Kotlin.
- No change to the XML resource gates (`jitter:onlyIf`/`jitter:set`) or the XSD.

## Capabilities

### New Capabilities

- `block-gates` — statement/block-level flavour gating of method-call and lambda bodies across
  jitter-api and jitter-plugin (stripping, verification, dejittering).

### Modified Capabilities

- (repo-local, same-named changes in `jitter-plugin`): `stripping` and `verification` gain
  block-level removal and tolerance rules.
