# Proposal

## Why

Block gates (`Gate.onlyIf(List<String>, Runnable)`) can only run side-effecting blocks. In Java a
lambda cannot assign to a captured local (effectively-final rule), so the very common case "the
caller wants the single result of a gated block in a local variable" is impossible with
`onlyIf` — the workarounds are clunky holders (`AtomicReference`) or pushing the value into a
field. Kotlin can mutate captured `var`s in closures, Java cannot, which makes the same source
pattern work in one language and not compile in the other. A value-returning gate closes this
gap: the gated block *supplies* a value that lands in a local via a plain assignment, with an
explicit `orElse` fallback so every flavour gets a well-typed (typically non-null) result.

## What Changes

- **jitter-api**: new additive static method
  `Gate.getOnlyIf(List<String> flavours, Supplier<T> block, Supplier<T> orElse)` returning `T`.
  In the 150% model it returns `block.get()`; `orElse` is never invoked at runtime. No new
  annotations, no changes to existing members — purely additive on the compatibility-sensitive
  leaf, version-bumped in coordination with jitter-plugin.
- **jitter-plugin stripping** (`GateRemover`): recognize `getOnlyIf` call sites by name and
  descriptor `(List;Supplier;Supplier;)Object`. In a listed flavour the `block` branch survives
  and produces the value while the whole `orElse` branch (flavour-list args, `invokedynamic`,
  synthetic lambda body) is stripped; in a non-listed flavour the mirror image. The discarded
  branch's synthetic lambda is removed by the existing orphan sweep, so references to
  flavour-stripped classes that occur only in the discarded branch leave no dangling references.
- **jitter-plugin dejittering** (`Dejitterer`, `KotlinDejitterer`): replace the whole
  `getOnlyIf(...)` call expression with the selected branch's source (expression lambda
  `() -> "EU"` becomes `"EU"`, constructor method reference `EsaPolicy::new` becomes
  `new EsaPolicy()`), so no `getOnlyIf` call survives in dejittered source. The 150% variant is
  unchanged.
- **block-gates spec**: new requirement "Value-returning gates" with scenarios for
  listed/non-listed/150%, branch stripping, verification tolerance, dejittering, Kotlin.
- **eval.jitter**: replace the uncommitted, non-compiling sketch in `GatedBlocks.java`
  (`localGatedText` assigning to a captured local) with working `getOnlyIf` demos
  (`localGatedTAndSuppliedText`, `gatedPolicy`) plus per-flavour `GatedBlocksTest` cases, and a
  Kotlin twin in `KotlinGadgets.kt`.
- No change to the XML resource gates (`jitter:onlyIf`/`jitter:set`) or the XSD.

## Capabilities

### New Capabilities

- (none — this extends the existing block-gate capability)

### Modified Capabilities

- `block-gates` — gains a value-returning gate requirement: the `Gate.getOnlyIf` API, per-flavour
  stripping of the unused branch, verification tolerance for references inside the stripped
  branch, and dejittering of the selected branch, for Java and Kotlin alike.

## Impact

- `~/projects/jitter-api`: `Gate.java` (+ `java.util.function.Supplier` import). Additive,
  backward compatible; coordinate the version bump with jitter-plugin.
- `~/projects/jitter-plugin`: `GateRemover` (recognize `getOnlyIf`, branch selection, span
  rewrite), `Dejitterer` / `KotlinDejitterer` (expression replacement), plus
  `GateRemoverTest` / `DejittererTest` / Kotlin dejitterer tests.
- `~/projects/eval.jitter`: `GatedBlocks.java` demo (replacing the broken sketch),
  `GatedBlocksTest.java`, `KotlinGadgets.kt`.
- Store spec `openspec/specs/block-gates` (delta in this change; synced to the main spec on
  archive).
