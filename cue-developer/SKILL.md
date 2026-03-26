---
name: cue-developer
description:
  You are an Expert CUE Software Engineer with deep knowledge of CUE (cue-lang) best practices.
  Use when creating or reviewing CUE modules, validating YAML/JSON, designing schemas and policies,
  or when the user asks about unification, definitions, cue vet/export/eval, integrations (JSON, YAML,
  OpenAPI, Protobuf, Go), or separating configuration from computation.
  Ensures code follows current best practices: order-independent constraints, clear layering of
  schema vs policy, and tooling-friendly structure.
disable-model-invocation: true
license: MIT
metadata:
  author: Johee Michel
  version: "1.1.0"
---

# CUE Developer

Guide for writing production-quality CUE following current best practices. Official documentation hub: [cuelang.org/docs](https://cuelang.org/docs/).

> **Structure:** This skill summarizes rules and patterns. For language details, follow the [Tour](https://cuelang.org/docs/tour/) and [Reference](https://cuelang.org/docs/reference/) sections on the official site.

## What CUE Is

CUE is an open-source **data validation language** and inference engine rooted in logic programming. It is **not** a general-purpose programming language. Typical uses: **data validation**, **configuration**, **querying** (via validation/subsumption), **code generation** and extraction, and **scripting** (tooling layer). See the [docs landing page](https://cuelang.org/docs/) and [Introduction](https://cuelang.org/docs/introduction/).

**Central idea — types are values:** CUE merges types and values into one ordered hierarchy (a lattice). There is no separate schema language; constraints, enums, and null handling collapse onto the same constructs. See [Introduction — Philosophy](https://cuelang.org/docs/introduction/).

**Unification** is the core operation: constraints on the same fields are combined and must all hold. Combining CUE from different sources is **associative, commutative, and idempotent** (order of composition does not change the result). See [Tour — Unification](https://cuelang.org/docs/tour/basics/unification/).

## Version Support

- Pin the **CUE language/tooling version** your project expects (check `cue version` in CI and local dev). Release notes: [github.com/cue-lang/cue/releases](https://github.com/cue-lang/cue/releases).
- The official docs currently highlight **CUE v0.16+**; prefer recent releases for modules, registry, and tooling fixes.

### Feature / Topic Map

| Topic | Where to read |
| --- | --- |
| Validation, vet, layering schema + policy | [How CUE enables data validation](https://cuelang.org/docs/concept/how-cue-enables-data-validation/) |
| Configuration, defaults, boilerplate, workflows | [How CUE enables configuration](https://cuelang.org/docs/concept/how-cue-enables-configuration/) |
| Codegen, extraction, OpenAPI/Go | [Code generation and extraction](https://cuelang.org/docs/concept/code-generation-and-extraction-use-case/) |
| Querying / subsumption | [Querying use case](https://cuelang.org/docs/concept/querying-use-case/) |
| JSON / YAML / OpenAPI / Protobuf / Go | [Integrations](https://cuelang.org/docs/integration/) |

## Quick Reference

**Core practices:**

- Prefer **definitions** (`#Name`) for reusable schemas; use **closed structs** when unknown fields must be rejected.
- Layer **schema** (shape and types) and **policy** (org rules) as separate files or packages that **unify** in the same package — no inheritance overrides; conflicting concrete values are errors.
- Use **`cue vet`** for validation, **`cue export`** / **`cue eval`** for emitting or inspecting evaluated config.
- Run **`cue fmt`** on CUE sources in CI; keep packages and module layout consistent (`cue help modules`).
- **Separate configuration from heavy computation:** keep declarative CUE for constraints and data; generate or inject computed data via pipelines or tooling when needed. See [Introduction — Separate configuration from computation](https://cuelang.org/docs/introduction/).

**CLI overview** (`cue help`): `vet`, `export`, `eval`, `fmt`, `import`, `trim`, `fix`, `def`, `mod`, `get`, `cmd`, plus topics `filetypes`, `inputs`, `commands`, `embed`, `environment`. See [cue help](https://cuelang.org/docs/reference/command/cue-help/).

## Project Overrides

How settings resolve:

- **Reviewing existing CUE / data in a repo:** Read **`cue.mod`**, **`module.cue`**, package layout, and any CI invocations (`cue vet`, `cue eval -e`, etc.). The repository is the source of truth for module path, constraints, and which definitions (`-d`) are validated.
- **Greenfield CUE:** Use the defaults below until the project publishes its own conventions.

Local overrides (domain context, review scope, frozen dirs, team patterns) take precedence over defaults.

### Defaults for New CUE Layout

| Setting | Default |
| --- | --- |
| Package model | One CUE package per directory; explicit `package` name |
| Schema style | Definitions with `#PascalCase`; concrete roots in separate files if large |
| Validation entry | `cue vet -c <cue-dir-or-files> <data...> [-d '#Def']` |
| Format | `cue fmt` (no manual layout arguments in CI — use defaults) |
| Closed structs | Use `#Foo: { ... }` with `...` only when extension is intentional |

### What Can Be Overridden

Same categories as other language skills: domain context, review behavior, migrations, team conventions, tool/CI specifics.

### How to Override

- **Cursor:** `.cursor/rules/cue-overrides.mdc` with `globs: "**/*.cue"` (and optionally `"**/*.{yml,yaml,json}"` if validating data).
- **Claude Code:** `CLAUDE.md`
- **Generic:** `AGENTS.md`

**Example:**

> **CUE skill overrides for this repository:**  
> Domain: schemas under `schemas/` describe GitOps promotion config; do not suggest removing `-d` flags used in CI.  
> Review: max 5 comments; skip style-only `cue fmt` nits if CI formats.  
> Codebase: `legacy/*.cue` frozen until Q4.

## Language Essentials

### Packages and modules

- **`package`** names must match across files in a directory. Use **modules** (`cue.mod`, `cue help modules`) for dependencies and reproducible builds.
- **`import`** only what you need; prefer standard library packages (`strings`, `struct`, etc.) from [standard library](https://cuelang.org/docs/tour/packages/).

### Definitions, fields, openness

- **`#Schema: { ... }`** — reusable schema. Top-level structs are **open** unless you close them; use **definitions** and **`...`** / field patterns to control extension. See [closed structs](https://cuelang.org/docs/tour/types/closed/) and [How CUE enables data validation](https://cuelang.org/docs/concept/how-cue-enables-data-validation/) (unknown field errors with `-d`).

- **Required / optional:** `field!: type` (required), `field?: type` (optional). See validation guide examples.

- **Defaults:** `field: type | *defaultValue` — understand unification with defaults so you do not “override” in the inheritance sense; conflicting concrete values fail.

### Unification and constraints

- Combine constraints with **`&`**, alternate allowed shapes with **`|`** (disjunctions / “sum types”).
- Bounds: comparisons on numbers; for strings/bytes, lexical or regex **`=~`** / **`!~`** where appropriate.
- **No contradictory concrete values:** if one place fixes `replicas: 3` and another `replicas: 4`, that is an error.

### Templates and boilerplate

- Use **struct comprehensions** and **field patterns** (e.g. `[string]:`) to apply shared constraints across many keys. See [configuration guide — Reducing boilerplate](https://cuelang.org/docs/concept/how-cue-enables-configuration/#reducing-boilerplate).
- **`cue trim`** can remove redundant fields once constraints are understood — use deliberately in pipelines.

### Tooling layer

- **`cue cmd`** and workflow commands integrate external tools (APIs, scripts, VCS). See `cue help commands` and [configuration — Tooling and automation](https://cuelang.org/docs/concept/how-cue-enables-configuration/#tooling-and-automation).

## Validation Workflow

1. **Write constraints** in CUE (schema + optional policy packages).
2. **Run `cue vet`** so each data file unifies with the constraints. Use **`-c`** / package flags as documented for your layout (`cue help vet`, `cue help inputs`).
3. For a **specific definition root**, use **`-d '#Definition'`** or an expression appropriate to your files.
4. **Silence means success** for `cue vet`.

Multi-format: CUE can unify constraints from **CUE + JSON Schema + Protobuf + YAML + JSON** in one vet — see [How CUE enables data validation](https://cuelang.org/docs/concept/how-cue-enables-data-validation/) and `cue help filetypes`.

## Integrations

- **Go:** extract CUE from Go, generate Go, use Go API — [Integration — Go](https://cuelang.org/docs/integration/go/).
- **Protobuf / OpenAPI:** constraints and exports — see [Code generation and extraction](https://cuelang.org/docs/concept/code-generation-and-extraction-use-case/) and integration docs.
- **YAML / JSON:** native interoperability; mind typing (e.g. numbers vs strings in YAML/JSON).

## Code Quality

### Naming

- **Definitions:** `#PascalCase` for schema-like definitions.
- **Fields:** `camelCase` or `lower_snake` — **match the data** you validate (YAML/JSON keys are often `snake_case`).
- **Packages:** short, lowercase, single word when possible.

### Project hygiene

- **`cue fmt`** in CI for all committed `.cue` files.
- **`cue vet`** on representative data fixtures on every PR that touches schemas or sample data.
- Document the **exact vet/eval commands** in README or CI config so reviewers can reproduce.

## Security and Safety

- Treat validated config as **security-sensitive**: vet before deploy; do not skip vet in pipelines.
- **Secrets:** do not embed secrets in CUE committed to git; inject via environment, secrets manager, or generated files excluded from VCS.
- **Understanding “open” structs:** permissive top-level validation may **not** catch extra fields — use closed definitions where rejection of unknown fields matters.

## Anti-Patterns (Avoid)

- ❌ Using CUE as a full application runtime — keep business logic outside; CUE validates and shapes config.
- ❌ Duplicating conflicting concrete values across files expecting “last wins” — unification **forbids** that.
- ❌ Giant single file with no definitions — hard to test, vet, and reuse.
- ❌ Omitting **`cue vet`** from CI for hand-edited YAML/JSON that must match schema.
- ❌ Ignoring **`cue.mod`/module** boundaries — leads to ambiguous imports and flaky CI.
- ❌ Relying on open structs when the requirement is **reject unknown fields** — close definitions or use `...` explicitly only where extension is intended.

## Checklist

### For CUE Authors

- [ ] Package names and `cue.mod` module path are correct; imports resolve.
- [ ] Schema (`#...`) vs concrete data vs policy files are layered clearly.
- [ ] Required fields (`!:`) and optionals (`?:`) match real data producers.
- [ ] `cue vet` (and `-d` if used) passes on fixtures; CI runs the same command.
- [ ] `cue fmt` applied; no spurious manual formatting drift.
- [ ] Closed/open behavior matches product needs (unknown fields allowed or not).
- [ ] No secrets in repo; sensitive config injected out-of-band.

### For Reviewers

**Quality rules (before commenting):**

- Only flag issues you are confident about; prefer fewer, high-signal comments.
- Respect repo overrides (frozen dirs, migration exceptions).
- Default scope: **critical** (wrong constraints, vet misses, contradictory values) and **important** (fragile openness, missing `-d`, CI/CLI mismatch).

**Priority 1 — Critical**

1. Constraints that **always fail** or contradict deployed data.
2. Missing or wrong **vet/eval** in CI for changed schemas or data.
3. Security: secrets in CUE, or validation gaps on authz-critical fields.

**Priority 2 — Important**

1. **Open vs closed** structs wrong for API guarantees.
2. Duplicated policy that will diverge across packages.
3. Module/import layout that will break consumers.

**Priority 3 — Nice to Have**

1. Naming consistency, file organization.
2. Opportunities for shared `#` definitions or trim — only if behavior stays identical.

## PR Review Workflow

When asked to review a PR (number, URL, or branch):

1. **Gather context:** `gh pr view` / `gh pr diff`; read `cue.mod`, vet/eval CI steps, and repo overrides.
2. **Review:** Check unification story, `-d` targets, fixtures, and CI parity with local commands.
3. **Present findings:** Group by priority; include file paths and line numbers; do **not** post to GitHub until the user approves.
4. **Post (only when approved):** Inline comments + one review; neutral `COMMENT` unless the user asks otherwise.

**Follow-up:** “re-review”, “approve”, “request changes” — same meaning as in Go/Python skills.

**Autonomous mode (CI/bots):** P1/P2 only, cap comment count, post as `COMMENT`; if no P1, silence is OK per team policy.

## Resources

| Resource | URL |
| --- | --- |
| Documentation home | [cuelang.org/docs](https://cuelang.org/docs/) |
| Introduction | [Introduction](https://cuelang.org/docs/introduction/) |
| Tour | [Tour](https://cuelang.org/docs/tour/) |
| Integrations | [Integrations](https://cuelang.org/docs/integration/) |
| Tutorials | [Tutorials](https://cuelang.org/docs/tutorial/) |
| How-to guides | [How-to](https://cuelang.org/docs/howto/) |
| Concept guides | [Concepts](https://cuelang.org/docs/concept/) |
| Reference (spec, CLI) | [Reference](https://cuelang.org/docs/reference/) |
| Installation | [Installation](https://cuelang.org/docs/introduction/installation/) |
| Playground | Linked from docs (“CUE playground”) |

**Concept guides cited above:** [Data validation](https://cuelang.org/docs/concept/how-cue-enables-data-validation/), [Configuration](https://cuelang.org/docs/concept/how-cue-enables-configuration/), [Code generation](https://cuelang.org/docs/concept/code-generation-and-extraction-use-case/), [Querying](https://cuelang.org/docs/concept/querying-use-case/).
