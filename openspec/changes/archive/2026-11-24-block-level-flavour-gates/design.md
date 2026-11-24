# Design

## Context

Today `@OnlyIf` targets declarations only (PACKAGE/TYPE/FIELD/CONSTRUCTOR/METHOD), and stripping
removes whole declarations from the ASM tree. Builder-style DSLs (arete-ktx:
`Spec({ spec("...") { scenario("...") { ... } } })`) express behaviour as method calls inside a
lambda, so there is no declaration to annotate. Both Java and Kotlin compile such a lambda to a
synthetic private method (`lambda$...$0` / `invoke$something`) plus an `invokedynamic` call site
— which makes block removal a well-defined bytecode operation independent of the source language.

## Goals / Non-Goals

**Goals:**
- Gate arbitrary statement blocks inside methods/lambdas per flavour, in Java and Kotlin, via a
  small jitter-api entry point.
- Remove call site + synthetic body in stripped flavours; keep surrounding bytecode valid.
- Verification tolerates references that only occur inside removed blocks.
- Dejittered sources and Kotlin `@kotlin.Metadata` stay consistent.

**Non-Goals:**
- Value-returning / expression-level gating — no ternary-style rewriting of arbitrary
  expressions; the gate is statement-level only.
- Changes to XML resource gates / XSD.
- Source rewriting during the build (stripping stays bytecode-only).

## Decisions

- **API shape: marker call, not annotation.** Java annotations cannot target statements, so the
  gate is a static method `Gate.onlyIf(List<String> flavours, Runnable)` in jitter-api.
  Alternative considered: `@OnlyIf` on `LOCAL_VARIABLE` — rejected, it cannot express "remove this
  block" and would not cover builder calls.
- **Removal unit: call site + synthetic body.** In the ASM tree, find `invokedynamic` (lambda
  metafactory) or `invokestatic`/`new`+lambda-body call sites whose bootstrap/callee is the gate
  method, delete the argument-setup instructions feeding the call, and delete the synthetic
  lambda method (`lambda$*` / `*-lambda$*` / Kotlin `invoke` synthetic) from the class. This is
  language-agnostic because it operates on the compiled shape, not the source.
- **CLASS retention** for the gate marker: visible to ASM stripping (via the call target, no
  runtime retention needed at all — the gate is identified by its method signature, so retention
  can even be CLASS on the API method itself; no runtime annotation is required).
- **Verification:** the removal manifest already lists removed classes/members; extend it to
  record removed synthetic lambda bodies, and treat references originating only from those bodies
  as satisfied, mirroring declaration-level tolerance.
- **150% semantics:** in the full model the gate simply executes the block — zero overhead beyond
  one static call.

## Risks / Trade-offs

- **Stack-frame recomputation:** deleting a mid-method call site requires recomputing stack map
  frames; mitigated by using ASM `ClassWriter(COMPUTE_FRAMES)` for affected methods (already the
  pattern for stripped classes).
- **Inline lambdas vs. capturing lambdas:** capturing lambdas still compile to one synthetic
  method; the call-site argument span differs (captured vars are extra invokedynamic args), so
  the instruction-span analysis must walk back over exactly the consumed arguments, not a fixed
  count.
- **Kotlin metadata:** removed synthetic `invoke`/`lambda` functions must be dropped from
  `@kotlin.Metadata` like other removed declarations; the existing metadata rewriter handles
  removed members, needs extension for removed synthetic functions.
- **False positives:** a user method that happens to call `Gate.onlyIf` reflectively or via
  method reference would not be stripped — acceptable, documented as direct-call only.


