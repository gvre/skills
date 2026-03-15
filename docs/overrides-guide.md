# Project Overrides Guide

Both the `python-developer` and `go-developer` skills support per-repository overrides. This lets you customize how the skill behaves — what it reviews, what it ignores, and what domain knowledge it applies — without modifying the global skill itself.

## How It Works

Each skill resolves settings in two layers:

1. **Tool settings** (python-version, line-length, linter rules, etc.) come from the project's own config files (`pyproject.toml`, `go.mod`, `.golangci.yml`). The skill reads these automatically when reviewing existing code.
2. **Overrides** (domain context, review behavior, codebase state, team conventions) come from a local rule file you create in the repository. These apply on top of both the project config and the skill defaults.

You only need to specify what differs from the defaults or what the AI cannot infer from reading the code.

## Where to Place Overrides

Use whichever file your agent platform reads:

| Platform | File | Notes |
|----------|------|-------|
| **Cursor** | `.cursor/rules/<skill>-overrides.mdc` | Use `globs: "**/*.py"` or `"**/*.go"` in frontmatter |
| **Claude Code** | `CLAUDE.md` | At the repository root |
| **Generic / agentskills.io** | `AGENTS.md` | At the repository root |

## What Can Be Overridden

Overrides are not limited to tool settings. You can provide any context that affects how the skill operates:

- **Domain / business context** — what the service does, external integrations, critical invariants
- **Review behavior** — priority levels to comment on, max comments, categories to skip
- **Codebase state** — ongoing migrations, intentional tech debt, frozen directories
- **Team conventions** — patterns specific to this repo that differ from generic best practices
- **External caveats** — downstream consumers, deployment constraints, third-party quirks
- **Tool overrides** — only needed when the project config files don't define them

## Complete Examples

### Python — Cursor (`.cursor/rules/python-overrides.mdc`)

```markdown
---
description: Python skill overrides for this repository
globs: "**/*.py"
alwaysApply: false
---

## Python Skill Overrides

Domain context:
This is the billing service. It processes subscription charges via Stripe
and issues refunds through Adyen. Idempotency, retry logic, and proper
error propagation in payment flows are critical.

Review behavior:
- Only flag Priority 1 (Critical) and Priority 2 (Important) issues
- Do not comment on docstring style, naming preferences, or modern
  Python adoption (os.path vs pathlib)
- Maximum 5 review comments per PR
- Do not flag missing type hints on private helper functions

Codebase state:
- Mid-migration from SQLAlchemy 1.x to 2.0; mixed session patterns in
  `src/billing/db/` are intentional
- The `src/billing/legacy/` directory is frozen and scheduled for
  removal in Q4; do not review files under it
- `src/billing/adapters/stripe_v2.py` is experimental; relaxed review

Team conventions:
- All database operations must go through the repository pattern in
  `src/billing/repositories/`
- Use `Decimal` for all monetary values, never `float`
- Custom exceptions must inherit from `src/billing/exceptions.py:BillingError`

External caveats:
- Stripe webhook signatures use a shared secret rotated monthly;
  the rotation logic in `src/billing/webhooks.py` is intentionally complex
- Adyen API has a known issue with idempotency keys longer than 64 chars

Tool overrides (only when pyproject.toml does not define them):
- docstring-style: NumPy
- test-directory: tests/unit/, tests/integration/
```

### Go — Cursor (`.cursor/rules/go-overrides.mdc`)

```markdown
---
description: Go skill overrides for this repository
globs: "**/*.go"
alwaysApply: false
---

## Go Skill Overrides

Domain context:
This is an event-driven order processing service consuming from Kafka
and writing to PostgreSQL. Exactly-once semantics matter — always flag
missing transaction boundaries, ack-before-process patterns, or missing
idempotency keys on mutations.

Review behavior:
- Only flag Priority 1 (Critical) and Priority 2 (Important) issues
- Do not comment on naming conventions or package organization
- Maximum 5 review comments per PR

Codebase state:
- Migrating from go-kit to plain stdlib HTTP; mixed patterns in
  `internal/transport/` are intentional
- `pkg/legacy/` will be removed after v3.0; do not review

Team conventions:
- All SQL queries go through `internal/store/` — no direct `database/sql`
  usage in handlers or services
- Use `shopspring/decimal` for monetary values
- Errors from external APIs must be wrapped with the caller's context
  AND the upstream status code

External caveats:
- Kafka consumer group rebalancing can cause duplicate deliveries;
  handlers must be idempotent
- The PostgreSQL connection pool is capped at 25; flag any code that
  holds connections across goroutine boundaries

Tool overrides (only when go.mod / .golangci.yml do not define them):
- test-assertions: none (use stdlib only)
```

### Python or Go — Claude Code / AGENTS.md

Same content as above, without the YAML frontmatter:

```markdown
# Project AI Rules

## Python Skill Overrides

Domain context:
This is the billing service. It processes subscription charges via Stripe
and issues refunds through Adyen. Idempotency, retry logic, and proper
error propagation in payment flows are critical.

Review behavior:
- Only flag Priority 1 (Critical) and Priority 2 (Important) issues
- Maximum 5 review comments per PR

Codebase state:
- Mid-migration from SQLAlchemy 1.x to 2.0; mixed session patterns are intentional
- The `src/billing/legacy/` directory is frozen; do not review files under it

Team conventions:
- Use `Decimal` for all monetary values, never `float`
- Custom exceptions must inherit from `BillingError`
```

## Tips

- **Only include what the AI can't figure out from the code itself.** Tool settings already come from `pyproject.toml` / `go.mod` — the override file is for business context, review behavior, and caveats.
- **Be specific.** "This service talks to Stripe" is more useful than "be careful with external APIs."
- **Name files and directories** when excluding or calling out special cases. The AI needs exact paths, not vague references.
- **Keep it short.** A 20-line override is better than a 200-line one. The AI reads this on every invocation.
- **Use the same category names** (Domain context, Review behavior, Codebase state, Team conventions, External caveats, Tool overrides) so the AI can parse them reliably across repositories.
