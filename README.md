# ansible-code-review

A Claude Code skill plugin for deep code reviews across the Ansible engineering stack: Python (Django, FastAPI, SQLAlchemy/SQLModel), React, and TypeScript.

## How it works

```
                              /review 522
                                  │
                    ┌─────────────┴──────────────┐
                    │     CONTEXT GATHERING       │
                    │  PR description & metadata  │
                    │  Project rules (CLAUDE.md)  │
                    │  Change classification      │
                    │  PR size check              │
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────┴──────────────┐
                    │    LOAD LANGUAGE GUIDES     │
                    │  ┌───────┐ ┌────────────┐  │
                    │  │ *.py  │ │ *.ts *.tsx  │  │
                    │  │guides │ │  guides    │  │
                    │  └───┬───┘ └─────┬──────┘  │
                    │      └─────┬─────┘         │
                    │       general.md            │
                    │       (always loaded)       │
                    └─────────────┬──────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
    ┌─────────┴────────┐ ┌───────┴────────┐ ┌────────┴─────────┐
    │   AGENT 1        │ │   AGENT 2      │ │   AGENT 3        │
    │   Bug & Logic    │ │   Security &   │ │   Design &       │
    │                  │ │   Performance  │ │   Maintainability │
    │ • Correctness    │ │ • Injection    │ │ • API contracts   │
    │ • Null handling  │ │ • Auth gaps    │ │ • Migrations      │
    │ • Race conditions│ │ • N+1 queries  │ │ • Test coverage   │
    │ • Type errors    │ │ • Sync-in-async│ │ • Framework       │
    │ • Language       │ │ • XSS / CSRF   │ │   anti-patterns   │
    │   pitfalls       │ │ • Bundle size  │ │ • Project rules   │
    └─────────┬────────┘ └───────┬────────┘ └────────┬─────────┘
              │                   │                   │
              │      (run in parallel)                │
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  │
                    ┌─────────────┴──────────────┐
                    │   VALIDATE & DEDUPLICATE    │
                    │                            │
                    │  • Merge duplicate findings │
                    │  • Verify against context   │
                    │  • Drop pre-existing issues │
                    │  • Check project rules      │
                    │  • Confirm with surrounding │
                    │    code before reporting    │
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────┴──────────────┐
                    │       FINAL REPORT          │
                    │                            │
                    │  Summary                   │
                    │  ├── Blockers              │
                    │  ├── Should fix            │
                    │  ├── Nits                  │
                    │  ├── Suggestions           │
                    │  └── Learning              │
                    │  Verdict: approve/request  │
                    └────────────────────────────┘
```

## Usage

```
/review                     # Review staged/unstaged changes
/review 522                 # Review PR #522
/review src/api/auth.py     # Review a specific file
/review feature/my-branch   # Diff branch against main
```

## Parallel Subagent Architecture

The review launches **3 independent agents in parallel**, each focused on a different class of issues:

| Agent | Focus | What it catches |
|-------|-------|-----------------|
| Bug & Logic | Correctness | Null handling, race conditions, off-by-one, type errors, mutable defaults, closure bugs, equality pitfalls |
| Security & Performance | Risk & speed | Injection, auth gaps, secrets, XSS, N+1 queries, sync-in-async, unbounded fetches, bundle size, re-renders |
| Design & Maintainability | Structure & standards | Breaking API changes, missing migrations, test gaps, framework anti-patterns, project rule violations, duplication |

Independent agents catch issues that a single-pass review would miss — each brings a different lens to the same code. Findings are then **deduplicated** (multiple agents may spot the same issue) and **validated** against surrounding context before reporting.

## Stack Coverage

| Layer | Frameworks | Guide |
|-------|-----------|-------|
| General | All languages | `reference/general.md` |
| Backend | Python, Django, FastAPI, SQLAlchemy, SQLModel, Alembic | `reference/python.md` |
| Frontend | React, TypeScript, JavaScript | `reference/typescript-react.md` |
| API & Data | REST design, Pydantic, Zod, database migrations | `reference/api-and-data.md` |

## Architecture

The skill uses **progressive disclosure** to minimize context window usage:

```
skills/review/
├── SKILL.md                          # Core workflow + agent orchestration
└── reference/
    ├── general.md                    # Cross-language patterns (always loaded)
    ├── python.md                     # Python, Django, FastAPI, SQLAlchemy
    ├── typescript-react.md           # TypeScript, React, JavaScript
    └── api-and-data.md              # API design, migrations, data modeling
```

Only guides relevant to the changed files are loaded. A pure Python PR won't load the TypeScript guide, and vice versa.

## Key Features

- **Parallel subagent review** — 3 agents analyze the diff independently from different perspectives, then findings are merged and validated
- **False-positive filtering** — every finding is verified against surrounding code, pre-existing state, and project conventions before reporting
- **Progressive loading** — language guides load on demand, keeping the core skill lean
- **Severity labels** — blocker, should fix, nit, suggestion, learning — so authors know what's blocking vs. informational
- **Concrete fixes** — findings include code examples, not just descriptions of problems
- **Project rules awareness** — checks CLAUDE.md, AGENTS.md, and similar files for project-specific conventions
- **Constructive framing** — questions over commands, praise for notably good work

## Installation

```bash
claude plugin install /path/to/ansible-code-review
```

Or from GitHub:

```bash
git clone https://github.com/matburt/ansible-code-review.git
claude plugin install ./ansible-code-review
```

## Review Output

The review produces:

1. **Context** — PR metadata, project rules, change classification (type, scope, risk)
2. **Summary** — what the change does and overall assessment
3. **Findings** — grouped by severity with file:line references and suggested fixes
4. **Verdict** — approve, approve with suggestions, or request changes

## Prerequisites

- `gh` CLI (for PR reviews)
- Git
