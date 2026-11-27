# Design

## Context

`Gate.onlyIf(List<String>, Runnable)` (see proposal.md — Why) can only run side-effecting
blocks. In Java a lambda cannot assign to a captured local, so "the gated block produces the
value of a local" needs a field or a holder (`AtomicReference`). The change adds a
value-returning gate `Gate.getOnlyIf(List<String>, Supplier<T> block, Supplier<T> orElse)`.

Verified bytecode shape of a `getOnlyIf` call site (javac, both branches):

```
[LDC flavours..][INVOKESTATIC List.of]   <- flavour list
[INVOKEDYNAMIC #block]                   <- Supplier for block   (ctor ref: handle to <init>, no lambda$ method;
[INVOKEDYNAMIC #orElse]                  <- Supplier for orElse    lambda: synthetic lambda$... method)
[INVOKESTATIC Gate.getOnlyIf (List,Supplier,Supplier)Object]
[CHECKCAST T][ASTORE / ARETURN / ...]
```

Existing machinery that carries over unchanged: the orphan-lambda fixpoint sweep in
`GateRemover` (removes a synthetic `lambda$` body once no surviving `invokedynamic` references
it), the reference-tolerance of `ReferenceVerifier` for content inside removed spans, and the
Kotlin `@kotlin.Metadata` rewrite for removed synthetic functions.

## Goals / Non-Goals

- **Goals**: value-returning gate for Java and Kotlin; exactly one branch survives stripping per
  flavour; the discarded branch leaves no trace (no instructions, no indy, no synthetic method,
  no dangling references); dejittered source contains the selected branch inline; 150% model
  unchanged (returns `block.get()`, never invokes `orElse`).
- **Non-Goals**: no own holder type in jitter-api (`AtomicReference` suffices for the rare
  mutation case); no 2-arg `getOnlyIf` without `orElse`; no plain-value default overload
  `getOnlyIf(List, Supplier<T>, T)` (eager evaluation of the default in every flavour would
  leave dangling references for flavour-specific defaults); no changes to XML resource gates.

## Decisions

1. **Distinct name `getOnlyIf`, not an `onlyIf` overload.** Verified with javac/kotlinc: adding
   `<T> T onlyIf(List, Supplier<T>)` silently rebinds existing expression-lambda call sites
   (`() -> text = "EU"`) from the `Runnable` descriptor to the `Supplier` descriptor, and in
   Kotlin a trailing lambda `{ "EU" }` binds to the `Runnable` overload and returns `Unit` — a
   silent type error. A distinct name keeps existing call sites byte-identical, lets the
   stripper/dejitterer recognize the gate by name, and gives Kotlin a clean trailing lambda for
   the last (`orElse`) parameter: `Gate.getOnlyIf(listOf("ESA"), Supplier { "EU" }) { "NONE-EU" }`.

2. **Three arguments, `orElse` as `Supplier<T>`.** The fallback must be lazy: a plain `T` default
   is evaluated eagerly at the call site in every flavour, so a flavour-specific default
   (`OtherPolicy::new`) would dangle after its class is stripped. With `Supplier<T>`, the
   discarded branch is removed with its indy and leaves no reference. `T` guarantees a type
   (both branches constrain inference, so `var policy = getOnlyIf(list, EsaPolicy::new,
   OtherPolicy::new)` infers the common supertype `Policy` — verified), not non-null; non-null
   holds only if the surviving branch supplies a non-null value.

3. **Stripping rewrites the call to the selected branch's `Supplier.get()`.** The span from the
   flavour-list argument start through the `INVOKESTATIC getOnlyIf` is removed, except the
   selected branch's `invokedynamic`, which is kept and followed by
   `INVOKEINTERFACE java/util/function/Supplier.get ()Ljava/lang/Object;`. The discarded branch's
   captured-argument loads sit inside the removed span; constructor-method-reference branches
   have no synthetic method (the ctor handle lives in the removed indy's bootstrap args), lambda
   branches leave an orphaned `lambda$` method that the existing fixpoint sweep removes.
   `GateRemover` recognizes the gate by owner `com/github/mictaege/jitter/api/Gate`, name
   `getOnlyIf`, descriptor `(Ljava/util/List;Ljava/util/function/Supplier;Ljava/util/function/Supplier;)Ljava/lang/Object;`.

4. **Dejittering replaces the whole call expression with the selected branch's source.** For an
   assignment `t = getOnlyIf(...)` only the call expression range is replaced (the existing
   `gateCall` already unwraps `AssignExpr` to find the call); the replacement text is the branch
   body — expression lambda `() -> "EU"` → `"EU"`, constructor method reference `EsaPolicy::new` →
   `new EsaPolicy()`. `KotlinDejitterer` mirrors this by call name. No `getOnlyIf` survives in
   dejittered source; the 150% variant is untouched.

5. **150% semantics: return `block.get()`, never invoke `orElse`.** The orElse supplier is still
   constructed (its indy survives), so its lambda body stays referenced and is not swept — the
   150% bytecode is exactly what javac/kotlinc produced.

## Risks / Trade-offs

- [Risk] Kotlin users must wrap the non-trailing `block` argument in an explicit `Supplier { ... }` →
  documented in the API javadoc and the spec scenario.
- [Risk] `T` gives type safety but not non-null; an `orElse` returning null still NPEs later →
  javadoc states the non-null contract is the caller's choice of branches.
- [Risk] jitter-api and jitter-plugin must be version-bumped in coordination (stripper must know
  the new descriptor) → additive change, coordinated release as per store conventions.
- [Risk] Statement-position `getOnlyIf(...)` with discarded result → allowed; dejitters to the
  selected branch as an expression statement, strips like any other gate call.

## Migration Plan

Purely additive: `Gate.getOnlyIf` is a new method; no existing member changes. Consumers adopt it
per call site; nothing to migrate. Rollback is trivial (remove the new method; no stripped
artefact depends on it unless used).

## Open Questions

- Whether a 2-arg `getOnlyIf(List, Supplier)` (non-listed → `null`) is ever needed; deferred —
  `() -> null` as orElse covers it today.
- Whether a plain-value default overload is needed for constant defaults; deferred — eager
  evaluation is safe only for side-effect-free constants.
