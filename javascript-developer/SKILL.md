---
name: javascript-developer
description:
  Expert JavaScript and TypeScript engineer for modern Node.js and browser code.
  Use when creating or reviewing JS/TS projects, refactoring, or when the user asks about
  npm/pnpm, ESLint, TypeScript, Vitest/Jest, async patterns, bundlers, or frontend/backend
  Node conventions. Ensures current best practices with strict typing, linting, testing, and security.
disable-model-invocation: true
license: MIT
metadata:
  author: Johee Michel
  version: "1.0.0"
---

# JavaScript / TypeScript Developer

Guide for writing production-quality JavaScript and TypeScript in Node.js and modern browsers.

> **Structure:** This file contains rules, guidelines, and compact examples. For extended notes and deep dives, see [references/REFERENCE.md](references/REFERENCE.md).

## Version Support

This guide covers:

- **Node.js 22+ (LTS)**: Base runtime for tooling, servers, and CLIs
- **TypeScript 5.x**: Strict mode, modern module resolution (`moduleResolution`: `bundler` or `node16`/`nodenext`)
- **Browsers**: Target via your build tool (Vite, esbuild, etc.); align `tsconfig` `lib` and `target` with actual support matrix

### Feature Version Matrix (runtime / language)

| Feature | Node 22 | Node 24+ |
|---------|---------|----------|
| `fetch` / `Request` / `Response` (stable) | ✅ | ✅ |
| `AbortSignal` / `AbortController` | ✅ | ✅ |
| `structuredClone` | ✅ | ✅ |
| Native `import.meta` in ESM | ✅ | ✅ |
| `Promise.withResolvers` | ✅ | ✅ |
| `Array.prototype.toSorted` / `toReversed` | ✅ | ✅ |
| `Intl` improvements | ✅ | ✅ (incremental) |

Align **language features** with `tsconfig` `target` and your bundler’s transpilation — do not assume runtime support without checking the deployment environment.

## Quick Reference

**Core requirements:**

- **Node.js 22+** for new server-side and tooling projects unless the repo specifies otherwise
- **`package.json`** + lockfile (`pnpm-lock.yaml`, `package-lock.json`, or `yarn.lock`) committed for applications
- **TypeScript** with `strict: true` for new projects; JSDoc-only typing only when the project standard is JS
- **ESLint** (flat config, ESLint 9+) + **Prettier** for formatting (or ESLint stylistic rules if team forbids Prettier)
- **Vitest** for unit/integration tests in new projects; **Jest** when the codebase already standardizes on it
- **`npm audit` / `pnpm audit`** or OSV in CI; keep dependencies patched

**Test runner comparison:**

| | Vitest | Jest |
|---|--------|------|
| **Best for** | New Vite/ESM projects, fast feedback | Large legacy codebases, React CRA-era setups |
| **Config** | `vitest.config.ts`, native ESM | `jest.config.js` / `jest.config.ts` |
| **API** | Jest-compatible (`describe`, `it`, `expect`) | De facto standard for years |
| **Recommendation** | **Default for new projects** | Keep when already entrenched |

## Project Overrides

How tool settings are resolved depends on the task:

- **Reviewing existing code / PRs:** Read `package.json`, `tsconfig.json` / `tsconfig.*.json`, ESLint config (`eslint.config.*` or legacy `.eslintrc*`), Prettier config, and the test runner config. The repository is the source of truth. Do not override what the project already defines.
- **Creating new projects:** Use the defaults below when no `package.json` / `tsconfig` exists yet.

Local overrides (see **What Can Be Overridden**) apply in both modes and take precedence over both the project config and the defaults below.

### Defaults for New Projects

| Setting | Default |
|---------|---------|
| node-version | 22 (LTS) |
| language | TypeScript |
| package-manager | pnpm (or match org standard) |
| module | ESM (`"type": "module"` where appropriate) |
| test-runner | Vitest |
| linter | ESLint 9 flat config |
| formatter | Prettier |
| strictness | `strict: true`, `noUncheckedIndexedAccess: true` (recommended) |

### What Can Be Overridden

Same categories as other language skills:

- **Domain / business context**
- **Review behavior** (priority levels, max comments, exclusions)
- **Codebase state** (migrations, intentional legacy)
- **Team conventions** (React vs Svelte, monorepo tool, etc.)
- **External caveats** (CDN, edge runtime limits, old browser floor)

### How to Override

- **Cursor:** `.cursor/rules/javascript-overrides.mdc` (e.g. `globs: "**/*.{js,mjs,cjs,ts,tsx}"`)
- **Claude Code:** `CLAUDE.md` at the repository root
- **Generic / agentskills.io:** `AGENTS.md` at the repository root

**Example override:**

> **JavaScript/TypeScript skill overrides for this repository:**
>
> Domain context:
> This package runs on Cloudflare Workers. Do not use Node-only APIs (`fs`, `child_process`) in worker code paths.
>
> Review behavior:
> - Only Priority 1 and 2
> - Maximum 5 comments per PR
>
> Codebase state:
> - Migrating from Jest to Vitest; both configs exist temporarily

## Project Structure

**Applications and libraries (recommended):**

```
project-name/
├── src/
│   ├── index.ts
│   └── ...
├── tests/                    # or colocated *.test.ts next to sources
├── package.json
├── tsconfig.json
├── eslint.config.js          # or eslint.config.mjs / .ts
├── prettier.config.mjs       # optional
└── vitest.config.ts          # or jest.config.ts
```

**Rules:**

- Prefer **`src/`** layout; avoid dumping application logic at repo root
- Use **consistent** test placement (single `tests/` tree or colocated) per repo
- **Barrel files** (`index.ts` re-exports): use sparingly; they can hurt tree-shaking and create circular import risk

## Configuration

### package.json (essentials)

```json
{
  "name": "your-project",
  "version": "0.1.0",
  "type": "module",
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "build": "tsc --noEmit false",
    "lint": "eslint .",
    "format": "prettier --write .",
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  }
}
```

**Rules:**

- Pin **`engines.node`** for applications and shared libraries consumed only internally
- Use **exact or caret** ranges consistently per org policy
- Prefer **`type: "module"`** for new packages; use `.cjs` only when CommonJS is required
- Avoid **silent postinstall scripts** that fetch binaries without pinning — security and supply-chain risk

### TypeScript (strict baseline)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

Adjust `module` / `moduleResolution` to `Bundler` when using Vite/esbuild bundling without NodeNext semantics.

## TypeScript and Modern JavaScript

**Rules:**

- Prefer **`interface` for object shapes** that may be extended; **`type` for unions, tuples, mapped types**
- Use **`unknown`** at boundaries, then narrow — avoid `any` unless documented escape hatch
- Prefer **`satisfies`** when you want inference plus excess property checking
- Use **nullish coalescing (`??`)** and **optional chaining (`?.`)** instead of loose `||` when `0` / `""` are valid
- Prefer **`for...of`** over `.forEach` when `async` / `await` or control flow matters
- Use **`const` by default**; `let` when reassigned; never `var`

**Forbidden:**

- **Unchecked `any`** in new code
- **Type assertions (`as`)** to silence errors without a comment or guard
- **Non-null assertion (`!`)** without proof the value exists

## Async Programming

```typescript
async function fetchWithTimeout(url: string, ms: number): Promise<Response> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), ms);
  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(timeout);
  }
}
```

**Rules:**

- **Always `await` or handle** promise-returning calls — no floating promises (ESLint: `@typescript-eslint/no-floating-promises`)
- Use **`AbortSignal`** for cancellation on network and long tasks
- Prefer **`async`/`await`** over raw `.then()` chains for readability in application code
- Use **`Promise.all`** for independent work; **`Promise.allSettled`** when partial failure is acceptable

## Error Handling

**Rules:**

- Throw **`Error`** (or subclasses) with **actionable messages**; attach **`cause`** when wrapping (`new Error('msg', { cause: err })`)
- At HTTP/API boundaries, **map errors to stable status codes and client-safe messages**; log full details server-side
- **Never swallow errors** silently (`catch {}` empty) — log, rethrow, or map explicitly
- Use **typed error classes** or **discriminated unions** for expected failure modes the caller must handle

## Testing (Vitest-oriented)

**Key patterns:**

- **Arrange / Act / Assert**; one logical behavior per test
- **`describe` / `it` or `test`** with clear names stating behavior
- **Mock I/O** at boundaries (HTTP, DB, filesystem); avoid mocking code under test
- Use **`vi.mock`** / dependency injection for modules that hit the network
- Run **`vitest run`** in CI; use **`coverage`** thresholds when the repo defines them

See [references/REFERENCE.md](references/REFERENCE.md) for extension points (snapshots, MSW, Playwright).

## Code Quality

### ESLint (flat config sketch)

Use the repo’s shareable config when present. For new projects, enable TypeScript-aware rules and disable patterns that conflict with Prettier if both run.

**Typical focus areas:**

- **`no-unused-vars`** / TS equivalent
- **`@typescript-eslint/no-floating-promises`**
- **`@typescript-eslint/no-misused-promises`** (especially Express-style handlers)
- **`eqeqeq`**, **`no-console`** (warn or error in production code per policy)
- **Import hygiene** (`import/no-duplicates`, orderly imports — often via `eslint-plugin-simple-import-sort` or similar)

### Naming Conventions

- **Variables / functions:** `camelCase`
- **Classes / types / interfaces:** `PascalCase`
- **Constants:** `SCREAMING_SNAKE_CASE` for true constants; `camelCase` for `const` object configs as team prefers
- **Files:** `kebab-case.ts` or `PascalCase.tsx` for React components — **match the existing repo**

## Logging

**Rules:**

- Use a **structured logger** (`pino`, `winston` with JSON, or platform-native logging) in servers — not raw `console.log` in hot paths
- **Never log secrets, tokens, or full PII**
- Include **correlation / request IDs** at boundaries when building services

## Dependencies and Security

**Rules:**

- **Lockfiles** for apps; **minimal dependency footprint** — prefer stdlib + small focused packages
- Run **`npm audit` / `pnpm audit`** or dedicated scanners in CI
- **Pin** or use **lockfile-only** installs in CI (`npm ci`, `pnpm install --frozen-lockfile`)
- Review **supply-chain** risk for install scripts and postinstall hooks

## Anti-Patterns (Never Do)

- ❌ **`==` with mixed types** — use `===` / `Object.is` as appropriate
- ❌ **Floating promises** in async code
- ❌ **Mutating function arguments** or shared objects without clear contract
- ❌ **`eval` / `new Function` on user-controlled strings**
- ❌ **Dynamic `require` with user input** (path traversal / RCE risk)
- ❌ **Hardcoded secrets** — use env vars or secret managers
- ❌ **Disabling ESLint / `@ts-ignore` wholesale** without justification

## Documentation

- **Public APIs:** TSDoc / JSDoc on exported functions and types when behavior is non-obvious
- **Types live in signatures** — avoid duplicating type information in comments unless clarifying invariants

## Checklist

### For Code Authors

- [ ] Node / browser targets documented or implied by `engines` + build config
- [ ] TypeScript `strict` (or repo standard) satisfied
- [ ] No floating promises; `AbortSignal` used for cancellable work
- [ ] Errors handled at boundaries; no empty catches
- [ ] Tests for new behavior; mocks at I/O boundaries
- [ ] ESLint and Prettier (or equivalent) pass
- [ ] Lockfile updated intentionally; audits considered
- [ ] No secrets in source or logs

### For Code Reviewers

**Review quality rules (apply before writing any comment):**

- **Only comment when confident.** False positives waste time.
- **Signal over noise.** Fewer, higher-impact comments beat long nit lists.
- **Default scope: Priority 1 and 2 only** unless overrides say otherwise.
- **Never nit style** that ESLint/Prettier already enforces.
- **Respect migrations and documented tech debt.**
- **Use domain context** (payment flows, auth, multi-tenant) to weight severity.
- **Honor review behavior overrides** (max comments, categories to skip).

**Priority 1 — Critical (must fix):**

1. **Security** (injection, XSS, path traversal, secrets in repo, unsafe deserialization)
2. **Correctness** (wrong async handling, race conditions, broken auth checks)
3. **Resource leaks** (missing `close` on streams, undisposed listeners, unbounded caches)

**Priority 2 — Important (should fix):**

1. **Error handling** (swallowed errors, leaking stack traces to clients)
2. **Type safety** (unsafe `any`, assertions masking real bugs)
3. **Testing gaps** for changed behavior or critical paths

**Priority 3 — Nice to have:**

1. Naming / file organization (non-blocking if lint passes)
2. Micro-optimizations without evidence
3. Suggesting alternative libraries without a concrete problem

Escalate P3 to P2 when impact is concrete (hot paths, large data, SSR TTFB, memory).

## PR Review Workflow

When asked to review a PR (by number, URL, or branch name):

### Step 1: Gather context

- Fetch the PR with `gh pr view` and `gh pr diff`
- Read `package.json`, `tsconfig.json`, ESLint/Prettier/test configs
- Read local overrides (`.cursor/rules/`, `AGENTS.md`, `CLAUDE.md`)
- Map changed files to the PR’s stated intent

### Step 2: Review

- Apply review quality rules and priorities above
- Flag deviations from the PR’s purpose, not only generic style
- Skip paths excluded by overrides

### Step 3: Present findings

- Group by priority with file paths and line numbers
- Do **not** post to GitHub until the user approves
- A clean review with no material findings is valid

### Step 4: Post (only when approved)

- One review submission with inline comments on specific lines
- Submit as neutral **`COMMENT`**, not **`REQUEST_CHANGES`**

### Follow-up commands

- **"re-review"** — latest diff, new/changed hunks only
- **"approve"** — approve with a short constructive summary
- **"request changes"** — request changes with outstanding issues listed

### Autonomous mode

When no human is in the loop:

- **P1 and P2 only**; max **5** inline comments; summarize any overflow
- **Post directly** (skip preview); **`COMMENT`** only
- **If no P1 issues, post nothing**
- On new commits, re-review changed hunks only

Repository overrides can adjust scope and limits.

> **Note:** Posting reviews requires `gh` CLI authenticated for the repo.

## Resources

- **Extension index:** [references/REFERENCE.md](references/REFERENCE.md)
