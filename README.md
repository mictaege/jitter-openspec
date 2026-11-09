# jitter-openspec — family store for the jitter family

This repository is the [OpenSpec](https://github.com/Fission-AI/OpenSpec) **store** for the jitter
family: the place for family-level planning (cross-repo changes, backlog ideas) next to the
per-repo `openspec/` roots. It is registered on each developer machine via `openspec store`.

The family:

| Repo | Role |
|---|---|
| [`jitter-api`](https://github.com/mictaege/jitter-api) | public flavour-markup annotation API (Java, dependency-free, Maven Central `io.github.mictaege:jitter-api`) |
| [`jitter-plugin`](https://github.com/mictaege/jitter-plugin) | Gradle plugin: ASM bytecode stripping, verification, dejittering, XML resource gates (Plugin Portal + Maven Central) |
| [`jitter`](https://github.com/mictaege/jitter) | schema site serving `jitter-gate.xsd` at `https://mictaege.github.io/jitter` (GitHub Pages from `main`) |
| [`eval.jitter`](https://github.com/mictaege/eval.jitter) | reference JavaFX demo consumer, flavours ESA/NASA/ROSKOSMOS (composite build via `includeBuild`; not published) |

Family architecture and conventions: `openspec/specs/family/spec.md`, `openspec/config.yaml`.

## Onboarding — from scratch

### 1. Prerequisites (install)

- Git with SSH access to `github.com/mictaege` (SSH keys configured).
- JDK 17+; always build via each repo's Gradle wrapper (`./gradlew`) — never a global Gradle.
  (eval.jitter on Linux/WSL additionally needs `libgtk-3-0` for the JavaFX UI.)
- OpenSpec CLI, standalone (no Node.js needed): `openspec install`
  (or `npm i -g @fission-ai/openspec` for the Node variant). Check with `openspec version`.
- Your agent (Vibe/Junie) picks up the OpenSpec skills automatically: they are committed in every
  project repo under `.vibe/skills/openspec-*` and `.agents/skills/openspec-*` — no per-repo
  install needed for existing repos.

### 2. Check out the store and register it

```bash
git clone git@github.com:mictaege/jitter-openspec.git ~/openspec/jitter
openspec store register ~/openspec/jitter --id jitter
openspec store list          # jitter must be listed
```

(Do the same for other families, e.g. `arete-openspec` → `~/openspec/arete`, id `arete`.)

### 3. Check out the project repos

```bash
cd ~/projects
git clone git@github.com:mictaege/jitter-api.git
git clone git@github.com:mictaege/jitter-plugin.git
git clone git@github.com:mictaege/jitter.git
git clone git@github.com:mictaege/eval.jitter.git
```

### 4. Configure / verify

- Global: `openspec config list` (default schema `spec-driven` is what we use).
- Per repo: nothing to init — `openspec/` is committed. Verify with `openspec doctor` inside the
  repo and `openspec list --specs`. If `openspec list --json` shows `"root": null`, the repo is not
  initialized.
- Optional convenience (machine-local, not in git) — recreate the workset:
  ```bash
  openspec workset create jitter-family \
    --member jitter-api=~/projects/jitter-api \
    --member jitter-plugin=~/projects/jitter-plugin \
    --member jitter-schema=~/projects/jitter \
    --member eval.jitter=~/projects/eval.jitter \
    --member jitter-store=~/openspec/jitter
  openspec workset open jitter-family
  ```
- Schema sync rule: when `jitter-plugin`'s `src/main/resources/jitter-gate.xsd` changes, update
  `jitter`'s `jitter-gate.xsd` AND `index.html` together in the same release (see the `gate-schema`
  spec in the `jitter` repo).

### 5. Everyday workflow

- Repo-local work: plain `openspec list` / `show` / `validate` inside the repo (root auto-detected).
- Family-wide work: add `--store jitter` (sticky per session), e.g. `openspec list --store jitter`,
  `openspec new change <id> --store jitter`, `openspec doctor --store jitter`.
- Change lifecycle: `/openspec-propose` → implement with `/openspec-apply-change` →
  `openspec validate <id>` → `/openspec-archive-change` (merges delta specs into `specs/`).
  - `specs/` = current behavior (never edit directly for behavior changes — go through a change).
  - `changes/` = proposals; backlog entries are proposal-only with `skip_specs: true`.
- Dashboards/health: `openspec view`, `openspec context --store jitter`, `openspec doctor --store jitter`.
- Pointers: each repo's `AGENTS.md` maps files → the spec to read before changing them; user-facing
  docs are the repos' READMEs.

### 6. Adding OpenSpec to a new repo

```bash
openspec init <path> --tools vibe,agents
```

Review the `AGENTS.md` diff it produces before committing, then seed `openspec/config.yaml`
(project context) and `openspec/specs/` from the existing docs/README/code.

### 7. Troubleshooting

- Store not found / "Invalid store declaration": `openspec store register <path>` and
  `openspec doctor --store <id>`.
- Validation errors: every `### Requirement` needs at least one `#### Scenario:` with
  `- **WHEN**` / `- **THEN**` bullets; run `openspec validate <item>`.
