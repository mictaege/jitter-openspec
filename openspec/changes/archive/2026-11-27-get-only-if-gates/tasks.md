# Tasks

## 1. jitter-api: value-gate API

- [x] 1.1 Add `Gate.getOnlyIf(List<String> flavours, Supplier<T> block, Supplier<T> orElse)` to
     `jitter-api` `Gate.java` returning `block.get()`; javadoc documents 150%-returns-block /
     never-invokes-orElse, the per-flavour branch stripping, and the Kotlin call shape
     (`Supplier { ... }` for the block argument, trailing lambda for orElse)
- [x] 1.2 Version bump: NOT in this change — feature commits keep the current version
     (as 104c604 did for `onlyIf`); the coordinated 2026.4 → next bump happens in the
     release commit after final testing, since functional-test fixtures resolve the
     published jitter-api from Maven Central

## 2. jitter-plugin: bytecode stripping of value gates

- [x] 2.1 `GateRemover`: recognize `getOnlyIf` call sites (owner `Gate`, name `getOnlyIf`,
     desc `(List;Supplier;Supplier;)Object`); locate the flavour-list args, the block indy and
     the orElse indy (incl. captured-argument loads)
- [x] 2.2 Per flavour, remove the unselected branch's span (list args / indy / capture loads)
     and the `getOnlyIf` call, keep the selected branch's indy and append
     `INVOKEINTERFACE Supplier.get()Object`; the discarded lambda branch is removed by the
     existing orphan sweep
- [x] 2.3 `GateRemoverTest`: listed flavour strips orElse branch (lambda and constructor-ref
     branches), non-listed strips block branch, nested `getOnlyIf` in orElse resolves per
     flavour, verification passes when the stripped branch references an `@OnlyIf`-stripped
     class

## 3. jitter-plugin: dejittering of value gates

- [x] 3.1 `Dejitterer`: replace the whole `getOnlyIf(...)` call expression with the selected
     branch's source (expression lambda body; `EsaPolicy::new` → `new EsaPolicy()`); assignment
     RHS keeps its target (`t = "EU";`); 150% unchanged
- [x] 3.2 `KotlinDejitterer`: same by call name for Kotlin sources
- [x] 3.3 `DejittererTest` / Kotlin dejitterer test: listed → block branch inline, non-listed →
     orElse branch inline, no `getOnlyIf` survives, 150% unchanged

## 4. eval.jitter: demo + flavour tests

- [x] 4.1 Replace the uncommitted non-compiling sketch in `GatedBlocks.java` with working
     `getOnlyIf` demos: `localGatedTAndSuppliedText()` (`var t = getOnlyIf(ESA, () -> "EU",
     () -> "NONE-EU")`) and `gatedPolicy()` (`var policy = getOnlyIf(ESA, EsaPolicy::new,
     OtherPolicy::new)` with the `@OnlyIf` policy classes)
- [x] 4.2 `GatedBlocksTest`: per-flavour `@OnlyIf` tests asserting the selected branch's value
     for the new demo methods
- [x] 4.3 Kotlin twin in `KotlinGadgets.kt` using `Gate.getOnlyIf(listOf(...), Supplier { ... }) { ... }`

## 5. Verification end-to-end

- [x] 5.1 `./gradlew test` in jitter-plugin (all stripper/dejitterer tests green)
- [x] 5.2 `./gradlew test` in eval.jitter for each flavour (ESA/NASA/ROSKOSMOS) — stripped model
     and dejittered sources compile and the gated demos assert the expected branch
- [x] 5.3 `openspec validate --change get-only-if-gates --store jitter` passes; archive the change
     and sync the `block-gates` delta into the main spec
