# Agent Skills

A collection of reusable [agent skills](https://docs.anthropic.com/en/docs/claude-code/skills) that provide specialized knowledge and best practices for various domains. Compatible with Claude Code, Cursor, and other AI coding agents.

## Skills

| Skill | Description |
|-------|-------------|
| [pr-reviewer](pr-reviewer/SKILL.md) | Language-agnostic PR review orchestrator. Detects languages in a diff, loads each language skill's review priorities, and applies shared quality rules, risk-tiering by blast radius, and review workflow in a single pass |
| [python-developer](python-developer/SKILL.md) | Rules and best practices for modern Python 3.12+ projects, covering tooling, type hints, async patterns, testing, and security |
| [go-developer](go-developer/SKILL.md) | Comprehensive rules and best practices for writing modern Go 1.25+ projects, covering tooling, interfaces, generics, concurrency, testing, and security |

## Installation

Install skills using the [skills CLI](https://www.npmjs.com/package/skills):

```
npx skills add gvre/skills
```

This will install all skills from this repository to your Claude Code agent.

### Install Specific Skills

```
npx skills add gvre/skills --skill pr-reviewer
npx skills add gvre/skills --skill python-developer
npx skills add gvre/skills --skill go-developer
```

### Installation Options

| Option | Description |
|--------|-------------|
| `-g, --global` | Install to user directory instead of project |
| `-a, --agent <agents>` | Target specific agents (e.g., `claude-code`, `cursor`) |
| `-s, --skill <skills>` | Install specific skills by name |
| `-l, --list` | List available skills without installing |
| `-y, --yes` | Skip all confirmation prompts |

## PR Review

The `pr-reviewer` skill is the entry point for reviewing pull requests across one or more languages. It detects the languages in a diff, reads the **Review Priorities** section from each relevant language skill (`go-developer`, `python-developer`, …), and runs a unified review with shared quality rules, risk-tiering by blast radius, and a consistent workflow. As you add new `<lang>-developer` skills, it picks up their priorities automatically.

The language skills also retain a standalone review workflow, so invoking one directly (e.g. `/go-developer` then "review this PR") still works for single-language reviews.

## Project Overrides

The skills support per-repository overrides for domain context, review behavior, codebase state, and more — without modifying the global skill. See the [Overrides Guide](docs/overrides-guide.md) for complete examples you can copy into your repository.

## Usage

Skills are invoked in AI coding agents using the `/` prefix followed by the skill name:

```
/pr-reviewer
/python-developer
/go-developer
```

The AI coding agent automatically loads the skill's context, making it available for the current task.

## Structure

Each skill lives in its own directory and follows this layout:

```
skill-name/
├── SKILL.md          # Skill definition and main content
└── references/       # Optional supplementary reference files
```

The `SKILL.md` file contains a YAML front matter header with metadata:

```yaml
---
name: skill-name
description: What the skill does and when to use it
license: MIT
metadata:
  author: Your Name
  version: "1.0.0"
---
```

## License

- [MIT](LICENSE.md)
