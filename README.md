# ansible-code-review

A Claude Code skill plugin for deep code reviews across the Ansible engineering stack: Python (Django, FastAPI, SQLAlchemy/SQLModel), React, and TypeScript.

## What it does

The `/review` skill performs a structured, multi-phase code review:

1. **Gathers context** — reads PR description, linked issues, project rules (CLAUDE.md, etc.), checks PR size
2. **Loads relevant guides** — only loads language-specific reference guides for file types in the diff
3. **Reviews against checklists** — security, correctness, performance, error handling, concurrency, framework patterns, API design, testing, maintainability
4. **Validates findings** — verifies each issue is real, introduced by this change, and not a false positive
5. **Reports with severity** — blocker, should fix, nit, suggestion, learning

## Usage

```
/review                     # Review staged/unstaged changes
/review 522                 # Review PR #522
/review src/api/auth.py     # Review a specific file
/review feature/my-branch   # Diff branch against main
```

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
├── SKILL.md                          # Core workflow (~120 lines)
└── reference/
    ├── general.md                    # Cross-language patterns (always loaded)
    ├── python.md                     # Python, Django, FastAPI, SQLAlchemy
    ├── typescript-react.md           # TypeScript, React, JavaScript
    └── api-and-data.md              # API design, migrations, data modeling
```

Only guides relevant to the changed files are loaded. A pure Python PR won't load the TypeScript guide, and vice versa.

## Key Features

- **False-positive filtering** — every finding is validated against surrounding code, pre-existing state, and project conventions before reporting
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
