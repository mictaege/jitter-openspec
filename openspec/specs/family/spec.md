# Jitter Family Specification

## Purpose

The jitter family is four GitHub repositories under `github.com/mictaege` that let one source base
produce multiple application "flavours" without forking or preprocessors. This store
(`openspec store jitter`, at `~/openspec/jitter`, remote `mictaege/jitter-openspec`) holds
family-level planning; each repo keeps its own `openspec/` root for repo-local specs.

## Requirements

### Requirement: Four repos with a one-way dependency direction

The family SHALL consist of: `jitter-api` (public annotation contract, dependency-free),
`jitter-plugin` (Gradle plugin depending on `jitter-api`; ASM bytecode stripping, verification,
dejittering, resource gates), `jitter` (schema repo serving `jitter-gate.xsd` at
`https://mictaege.github.io/jitter` via GitHub Pages), and `eval.jitter` (reference demo consumer,
not published). Dependencies SHALL only point `jitter-plugin` → `jitter-api` and
`eval.jitter` → both; `jitter-api` SHALL NOT depend on the others.

#### Scenario: Demo validates plugin changes locally
- **WHEN** a change is made in `jitter-api` or `jitter-plugin`
- **THEN** `eval.jitter` picks it up via `includeBuild("../jitter-api")` /
  `includeBuild("../jitter-plugin")` in its `settings.gradle`, so `./gradlew build` there
  exercises the local code per flavour without publishing

### Requirement: The schema repo stays in sync with the plugin's shipped schema

`jitter`'s `jitter-gate.xsd` and `index.html` SHALL be identical, and both SHALL be updated
together with `jitter-plugin`'s `src/main/resources/jitter-gate.xsd` (the source of truth) on
every plugin release; the site SHALL be served by GitHub Pages from the `main` branch root.

#### Scenario: IDE fetches the gate schema
- **WHEN** an XML resource declares `xmlns:jitter="https://mictaege.github.io/jitter"`
- **THEN** IDEs resolve the namespace against the served schema and offer completion for
  `jitter:onlyIf` / `jitter:set`

### Requirement: API stability of jitter-api

The annotation API of `jitter-api` SHALL stay backward compatible: `jitter-plugin` and consumer
applications are compiled against it, so removing/renaming annotations is a breaking change that
SHALL be coordinated with `jitter-plugin` and `eval.jitter` and called out in release notes.

#### Scenario: Additive annotation change
- **WHEN** a new annotation is added to `jitter-api`
- **THEN** existing consumers compile unchanged; the plugin may adopt it optionally

### Requirement: Cross-repo changes are planned together

A feature spanning repos (e.g. a new annotation plus its stripping support) SHALL get a
same-named OpenSpec change in each affected repo (or in this store), implemented on `develop`
together, so the family stays consistent.

#### Scenario: New flavour-gate annotation
- **WHEN** a new gate annotation is added to `jitter-api` and handled by `jitter-plugin`
- **THEN** both repos carry the same-named change and the schema/docs are updated in the same
  release
