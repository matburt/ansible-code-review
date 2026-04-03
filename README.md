# ansible-code-review

A Claude Code skill plugin for deep code reviews across the Ansible engineering stack: Python (Django, FastAPI, SQLAlchemy/SQLModel), React, and TypeScript.

## What it does

The `/review` skill performs a structured code review covering:

- **Security** — injection, auth gaps, secrets exposure, OWASP patterns
- **Correctness** — race conditions, null handling, logic errors, resource leaks
- **Performance** — N+1 queries, unbounded fetches, unnecessary re-renders, bundle size
- **Framework patterns** — Django ORM, FastAPI dependencies, React hooks, TypeScript types
- **API design** — contract consistency, status codes, validation, migrations
- **Testing** — coverage gaps, flaky patterns, mock overuse
- **Maintainability** — duplication, naming, dead code, codebase consistency

Findings are grouped by severity (blocker / should fix / suggestion) with file references and concrete fixes.

## Usage

```
/review                     # Review staged/unstaged changes
/review 522                 # Review PR #522
/review src/api/auth.py     # Review a specific file
/review feature/my-branch   # Diff branch against main
```

## Installation

```bash
claude plugin install /path/to/ansible-code-review
```

Or from GitHub:

```bash
git clone https://github.com/matburt/ansible-code-review.git
claude plugin install ./ansible-code-review
```

## Stack Coverage

| Layer | Frameworks |
|-------|-----------|
| Backend | Python, Django, FastAPI, SQLAlchemy, SQLModel, Alembic |
| Frontend | React, TypeScript, Vite |
| Testing | pytest, vitest, React Testing Library |

## Review Output

The review produces:

1. **Classification** — change type, scope, and risk level
2. **Summary** — what the change does and overall assessment
3. **Findings** — grouped by severity with file/line references and suggested fixes
4. **Verdict** — approve, approve with suggestions, or request changes
