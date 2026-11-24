# Tasks

## 1. jitter-api: gate API

- [x] 1.1 Add `com.github.mictaege.jitter.api.Gate` with `onlyIf(List<String> flavours, Runnable block)`
     (CLASS retention, javadoc documenting
      150%-executes / stripped-removes semantics)
- [x] 1.2 ~~jitter-api unit tests~~ dropped: no test source set/dependencies in jitter-api; the
      gate's 150% behavior is covered by the eval.jitter demo tests instead

## 2. jitter-plugin: stripping

- [x] 2.1 Detect gate call sites in ASM (invokedynamic/invokestatic targeting `Gate.onlyIf`),
      including the argument-setup instruction span for capturing and non-capturing lambdas
- [x] 2.2 Remove call site + synthetic lambda body method per non-listed flavour; recompute
      frames for affected methods
- [x] 2.3 Record removed synthetic bodies in the removal manifest
- [x] 2.4 Extend Kotlin `@kotlin.Metadata` rewrite to drop references to removed synthetic
      lambda/invoke functions

## 3. jitter-plugin: verification & dejittering

- [x] 3.1 Verification: tolerate dangling references that originate only inside removed gated
      blocks
- [x] 3.2 Dejittering: keep gated blocks in dejittered sources

## 4. eval.jitter demo + tests

- [x] 4.1 eval.jitter: demo gated block in Java and Kotlin, asserted absent/present per flavour
- [x] 4.2 jitter-plugin tests: strip ESA/NASA variants of a gated class, verify bytecode and
      verification pass

## 5. Cross-repo coordination

- [x] 5.1 Create same-named `block-level-flavour-gates` changes in `jitter-api` and
      `jitter-plugin` repo-local openspec roots; keep release notes in sync
