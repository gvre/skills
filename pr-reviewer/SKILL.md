---
name: pr-reviewer
description:
  Expert PR reviewer that orchestrates language-specific review across any combination
  of languages in a single pass. Use for all PR reviews — single or multi-language.
  Detects languages from the diff, reads the Review Priorities from the corresponding
  language skill (go-developer, python-developer, etc.), and applies shared quality
  rules, risk-tiering, and workflow once. Prefer this over invoking a language skill
  directly when PR review is your primary goal.
disable-model-invocation: true
license: MIT
metadata:
  author: Giannis Vrentzos
  version: "1.0.0"
---

# PR Reviewer

General-purpose PR review skill. Detects languages in the diff, reads the
**Review Priorities** section from each relevant language skill, and runs a
unified review with shared quality rules, risk-tiering, and workflow.

## How It Works

1. **Detect languages** from changed file extensions in the diff.
2. **Read language priorities.** For each detected language, locate the
   corresponding skill in the same skills directory and read its
   `### Review Priorities` section. Skills live at
   `<lang>-developer/SKILL.md` alongside this file:
   - `.go` → `go-developer/SKILL.md`
   - `.py` → `python-developer/SKILL.md`
   - Other extensions → apply shared P1 security/correctness checks only
3. **Apply risk tiering** to each changed file (see below).
4. **Run review** using the combined language priorities + shared rules below.
5. **Follow the workflow** for presenting and posting findings.

When the user adds a new language skill (`<lang>-developer/SKILL.md`) that
contains a `### Review Priorities` section, this skill picks it up automatically
— no changes needed here.

## Risk Tiering by Blast Radius

Classify each changed file by consequence *before* reviewing. Review depth
scales with blast radius, not diff size.

| Tier | Examples | Review depth |
|------|----------|-------------|
| **Critical** | auth, payments, crypto, data-deletion, DB migrations, anything handling untrusted input or PII | Full rigor — apply all priority levels, escalate borderline findings up one level |
| **Standard** | business logic, APIs, service layers | Normal priority rules |
| **Low** | config, docs, generated code, test-only changes | P1 correctness/security only — do not nitpick |

## Review Quality Rules

Apply these before writing any comment:

- **Only comment when confident.** Uncertainty = silence. A false positive
  wastes more time than a missed suggestion.
- **Signal over noise.** Two critical findings beat fifteen mixed-confidence
  suggestions. Fewer, higher-quality comments.
- **Default scope: Priority 1 and 2 only.** P3 only when repository overrides
  explicitly request it.
- **Never comment on pure style preferences** the project's linter does not
  flag. If the linter passes, the style is acceptable.
- **Respect the codebase's current state.** Known migrations, intentional
  tech debt, and phased-out legacy patterns are not findings.
- **Use domain context.** A missing timeout in a payment service is critical;
  the same in a migration script may not be. Weigh findings against what the
  code actually does.
- **Honor review behavior overrides.** Max comment count, confidence threshold,
  excluded categories — follow them strictly.
- **Findings are sensor data, not verdicts.** Never imply a change is safe to
  merge because the review is clean — surface what you checked and what you
  could not. Borrowed confidence (a confident automated approval with no human
  understanding behind it) is itself a risk. You may be one of several reviewers
  on this PR; a clean result covers only the priorities in this skill, not the
  PR as a whole.
- **Stay in your lane.** Focus on the priorities defined here (language
  correctness, security, resource safety). Don't expand into generic coverage
  another reviewer is better positioned to provide — independent, specialized
  signal is more valuable than correlated overlap.

## Shared Priority Levels

These apply across all languages. Language-specific items (goroutine leaks,
`errors.Is` for Go; mutable defaults, bare `except` for Python; etc.) are
loaded from each language skill's `### Review Priorities` section.

**Priority 1 - Critical (Must Fix):**

1. **Security vulnerabilities** (injection, hardcoded secrets, path traversal,
   unsafe eval, untrusted input reaching dangerous sinks). Includes **prompt
   injection** — untrusted/user-controlled text flowing into an LLM call without
   safeguards. This risk is latent (the dangerous data arrives at runtime, not
   in the diff), so scrutinize any new code path that pipes external input into
   a model prompt.
2. **Newly suppressed security-linter checks** — `//nolint gosec`, `# noqa S...`,
   or equivalent suppressions *added in this diff*; treat the suppressed warning
   as an active finding. Pre-existing suppressions are out of scope.
3. **Suspicious test changes** — assertions weakened or rewritten to match new
   (possibly broken) behavior, tests deleted or skipped, coverage of the changed
   code path removed. When a diff changes both code and tests, verify the tests
   still assert *correct* behavior, not just *current* behavior.
4. **Resource leaks** (unclosed handles, missing cleanup, goroutine/task leaks)

**Priority 2 - Important (Should Fix):**

1. **Error handling gaps** (silent failures, overly broad catches, missing context)
2. **Testing gaps** (missing tests for changed behavior, no mocking of external deps)

**Priority 3 - Nice to Have (Consider):**

1. **Modern language adoption** (idiomatic rewrites, better stdlib alternatives)
2. **Documentation** (missing or incomplete docstrings / godoc)
3. **Performance** (unnecessary allocations, repeated expensive calls). Escalate
   to P2 when concrete: hot-path allocations, O(n²) over large inputs, blocking
   sync I/O in async code.
4. **Reinvention** — new code that reimplements an existing stdlib or internal
   helper instead of reusing it. Escalate to P2 if the reimplemented helper
   handles security-sensitive logic.
5. **Generic safety gate weakening** — unrelated `//nolint`, `# noqa`, or
   `# type: ignore` without justification, skipped non-critical tests, relaxed
   coverage thresholds, loosened linter or type-checker config.

## PR Review Workflow

When asked to review a PR (by number, URL, or branch name):

### Step 1: Gather context
- Fetch the PR description, metadata, and diff using `gh pr view` and
  `gh pr diff`
- **Fast-fail screening:** a sprawling diff, mass test rewrites, or a
  vague/missing intent statement are themselves findings. Flag "PR too large
  to review confidently — recommend splitting" rather than rubber-stamping.
- Detect languages from changed file extensions; read the `### Review
  Priorities` section from each relevant language skill
- Read project config files (e.g. `go.mod`, `.golangci.yml`, `pyproject.toml`)
- Read any local overrides (`.cursor/rules/`, `AGENTS.md`, `CLAUDE.md`)
- Fetch existing review comments (`gh pr view --comments`) and skip findings
  already raised by another reviewer or the author — don't restate them
- Apply risk tiering to classify each changed file

### Step 2: Review
- Apply the review quality rules and priority levels above plus any
  language-specific priorities loaded in Step 1
- Assess changes against the PR's stated intention — flag deviations from
  what the PR says it does, not just generic code quality
- Ignore files excluded by local overrides (e.g. frozen directories)

### Step 3: Present findings
- Show findings grouped by priority, with file paths and line numbers
- Do NOT post to GitHub until the user explicitly approves
- If there are no findings worth flagging, say so — a clean review is a
  valid outcome

### Step 4: Post (only when approved)
- Post each finding as a separate inline review comment on the specific line
- Use a single review submission (not individual comments) so the author gets
  one notification, not N
- Submit as a neutral `COMMENT` review, not `REQUEST_CHANGES`. The user
  decides the disposition via follow-up commands below.

### Follow-up commands
These work without repeating the full context:
- **"re-review"** — fetch latest changes, review only new/modified hunks,
  and present findings
- **"approve"** — approve the PR with a constructive comment
- **"request changes"** — submit the review requesting changes with a summary
  of outstanding issues

### Autonomous mode
When running without a human in the loop (CI, GitHub Actions, bots):
- **Scope: Priority 1 and 2 only.** Do not comment on P3 issues.
- **Maximum 5 review comments per PR.** Post the 5 most critical; note the
  count of remaining issues in the review summary.
- **Post directly** — skip Step 3. There is no user to approve.
- **Submit as `COMMENT`**, never `REQUEST_CHANGES` or `APPROVE`.
- **If no P1 issues are found, do not post any review.** Silence means
  approval from the autonomous reviewer.
- **On new commits** to the same PR, re-review only the changed hunks.

Repository overrides can widen or narrow these defaults (e.g. include P2,
raise or lower the max comment count).

> **Note:** Posting to GitHub requires `gh` CLI to be installed and
> authenticated.
