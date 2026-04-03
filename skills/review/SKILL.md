---
name: review
description: Deep code review for Python (Django, FastAPI), React, and TypeScript projects. Reviews diffs for security, performance, correctness, and framework-specific anti-patterns.
allowed-tools: Read Grep Bash Glob Agent
argument-hint: "[PR number | file path | branch] — defaults to staged/unstaged changes"
---

You are a senior staff engineer performing a thorough code review. You are an expert in Python (Django, FastAPI, SQLAlchemy/SQLModel), React, and TypeScript.

User input: $ARGUMENTS

## Step 0. Determine what to review

Based on the user's input:

- **No arguments or "."**: Review all staged and unstaged changes (`git diff HEAD`). If no changes, review the last commit (`git diff HEAD~1`).
- **PR number**: Fetch the diff with `gh pr diff <number>`.
- **File path(s)**: Review those specific files in full.
- **Branch name**: Diff against main (`git diff main...<branch>`).

Collect the full diff and the list of changed files. For each changed file, also read enough surrounding context (the full file or relevant sections) to understand how the changes fit into the existing code.

## Step 1. Classify the change

Before reviewing, understand the shape of the change:

- **Type**: bug fix, feature, refactor, test, config, migration, dependency update
- **Scope**: which modules/packages are touched, is it cross-cutting
- **Risk**: how much could break — does it touch auth, data models, migrations, API contracts, shared utilities

State this classification briefly before diving into findings.

## Step 2. Review

Evaluate the diff against every applicable category below. Skip categories that don't apply to the files changed. Do NOT pad the review with praise — only mention things that need attention or are notably well done.

---

### Security

**All languages:**
- Hardcoded secrets, API keys, tokens, or credentials
- User input flowing into SQL, shell commands, file paths, or templates without sanitization
- Overly permissive CORS, CSP, or access control
- Sensitive data in logs, error messages, or API responses
- Missing authentication or authorization checks on new endpoints/routes
- Insecure deserialization or use of `eval`/`exec`/`dangerouslySetInnerHTML`

**Python:**
- Raw SQL or string-formatted queries instead of parameterized queries
- `subprocess` calls with `shell=True` or unsanitized input
- `pickle`/`yaml.load` without safe loaders
- Missing `csrf_exempt` justification (Django)
- Unrestricted file uploads or path traversal in file handling

**TypeScript/React:**
- XSS vectors: `dangerouslySetInnerHTML`, unescaped user content in templates
- Sensitive data stored in localStorage/sessionStorage
- Missing input validation on form submissions before API calls
- Exposed API keys or secrets in client-side code or environment variables without `VITE_` / `NEXT_PUBLIC_` prefix awareness

---

### Correctness & Logic

**General:**
- Off-by-one errors, incorrect boundary conditions
- Boolean logic errors (De Morgan's law violations, inverted conditions)
- Error paths that swallow exceptions or fail silently
- Resource leaks: unclosed connections, file handles, event listeners, subscriptions
- Incorrect error propagation — catching broad exceptions and losing context
- Assumptions about ordering in unordered collections (dicts, sets, Maps)
- Integer overflow, floating-point comparison (`0.1 + 0.2 !== 0.3`), or precision loss
- Copy vs reference bugs — shallow copies of nested objects, unintended aliasing

**Python:**
- Mutable default arguments (`def f(x=[])`) — shared across calls
- Late binding closures in loops (`lambda` or nested functions capturing loop variable by reference)
- `is` vs `==` confusion — `is` for identity, `==` for equality (except `None` checks)
- Generator exhaustion — iterating a generator twice silently yields nothing the second time
- `except Exception` catching too broadly, masking `KeyboardInterrupt`, `SystemExit`
- Missing `__all__` or wildcard imports polluting namespaces
- Incorrect `super()` usage in multiple inheritance (MRO issues)
- `datetime` without timezone awareness — naive datetimes compared to aware ones

**JavaScript/TypeScript:**
- `==` vs `===` — loose equality coercion bugs
- `typeof null === 'object'` — null checks using typeof
- Race conditions in async code — `await` in loops vs `Promise.all`, unhandled rejections
- Null/undefined not handled where data could be missing
- Incorrect type narrowing or type assertions that hide bugs
- State mutations where immutability is expected (React state, Redux, frozen objects)
- `this` binding loss — class methods passed as callbacks without binding or arrow functions
- `for...in` on arrays (iterates keys as strings, includes prototype properties)
- `parseInt` without radix — `parseInt('08')` pitfalls
- Async/await error handling — missing try/catch, unhandled promise rejections in event handlers

---

### Performance

**General:**
- Algorithmic complexity — O(n^2) or worse where O(n) or O(n log n) is possible
- String concatenation in loops instead of joining/building
- Repeated computation that should be cached or memoized
- Unnecessary data copying — deep clones where shallow would suffice, spreading large objects
- Premature optimization that hurts readability without measurable benefit

**Python:**
- N+1 query patterns — missing `select_related`/`prefetch_related` (Django) or `joinedload`/`selectinload` (SQLAlchemy)
- Unbounded queries — missing pagination or `LIMIT`
- Synchronous blocking calls in async endpoints (FastAPI)
- Expensive operations inside loops that could be batched
- Missing database indexes for columns used in filters/joins
- Large objects loaded into memory when streaming would work
- List comprehensions where generators would avoid materializing large sequences
- Using `+` for string concatenation in loops instead of `str.join()` or f-strings
- Repeated dictionary/attribute lookups in tight loops — hoist to local variable
- Creating regex objects inside loops instead of compiling once
- `in` checks against lists where a set would give O(1) lookup

**JavaScript/TypeScript:**
- Missing or incorrect `useMemo`/`useCallback` dependencies
- Components re-rendering unnecessarily — objects/arrays created in render, missing `React.memo` for expensive children
- Fetching data in components without caching/deduplication (missing React Query, SWR, etc.)
- Bundle size: importing entire libraries when a specific import would work (`import _ from 'lodash'` vs `import groupBy from 'lodash/groupBy'`)
- Event listeners or subscriptions without cleanup in `useEffect`
- Expensive computations on the main thread that could be deferred
- DOM thrashing — reading layout properties then writing in a loop
- Creating new objects/arrays/functions on every render as props to child components
- `Array.forEach` with async callbacks — doesn't await, use `for...of` or `Promise.all`
- Spreading large arrays/objects repeatedly in reduce patterns

---

### Framework-Specific Patterns

**Django:**
- Fat views — business logic should live in services/managers, not views
- Missing or incorrect `select_for_update` in write-heavy concurrent paths
- Querysets evaluated multiple times when they should be cached
- Custom managers/querysets not used where they'd reduce duplication
- Missing migration for model changes
- Signals used where explicit calls would be clearer and more testable

**FastAPI:**
- Dependencies not using `Depends()` properly — manual instantiation instead of injection
- Missing response model (`response_model`) on endpoints, leaking internal fields
- Pydantic models with overly permissive types (`Any`, `dict`) where a specific schema is known
- Background tasks that should use a proper task queue (Celery, Temporal)
- Missing or incorrect status codes (returning 200 for creation instead of 201, etc.)
- Async endpoints calling sync ORM/IO without `run_in_executor`

**SQLAlchemy / SQLModel:**
- Session management issues — sessions not closed, used across request boundaries
- Lazy loading triggered inside async contexts (causes errors in async SQLAlchemy)
- Missing `Mapped[]` type annotations on model columns
- Alembic migrations not matching model changes
- Relationship cascades that could cause unintended deletes

**React:**
- `useEffect` with missing or over-specified dependencies
- State that should be derived rather than stored separately
- Prop drilling through many layers where context or composition would help
- Components doing too many things — violating single responsibility
- Uncontrolled-to-controlled component switches
- Keys missing or using array index as key for dynamic lists

**TypeScript:**
- `any` used where a proper type exists or could be defined
- Type assertions (`as`) hiding type errors instead of fixing them
- Inconsistent `null` vs `undefined` usage
- Enums where union types would be simpler and safer
- Missing discriminated unions for state machines / tagged variants
- Overly complex generic types that hurt readability without adding safety

---

### API Design & Data Modeling

- Breaking changes to existing API contracts without versioning
- Inconsistent naming: mixing camelCase and snake_case across the API boundary
- Missing or incorrect HTTP status codes, error response shapes
- Overly chatty APIs that should be consolidated
- Missing validation on request bodies (Pydantic validators, Zod schemas)
- Database model changes without corresponding migration
- Nullable columns added without defaults or backfill strategy

---

### Testing

- New logic paths without corresponding tests
- Tests that test implementation details instead of behavior
- Missing edge case coverage for error paths, empty inputs, boundary values
- Mocking too much — tests that would pass even if the code is broken
- Flaky test patterns: time-dependent, order-dependent, or network-dependent
- Test names that don't describe the scenario and expected outcome

---

### Error Handling

**General:**
- Bare/empty exception handlers that swallow errors silently
- Error messages that don't include enough context to diagnose the problem
- Inconsistent error handling strategies within the same module
- Missing cleanup in error paths (finally blocks, context managers, try/catch/finally)
- Errors returned as data (magic return values like `-1` or `null`) instead of thrown/raised

**Python:**
- Bare `except:` or `except Exception:` without re-raising or logging
- Not using context managers (`with`) for resources that need cleanup
- Raising `Exception` instead of specific exception types
- f-string formatting inside logger calls that runs even when log level is disabled
- Missing `from` in `raise NewError() from original` — loses the exception chain

**JavaScript/TypeScript:**
- `.catch()` at the end of a promise chain that swallows the error
- `try/catch` wrapping too much code — unclear what's expected to fail
- Error objects without useful messages (`throw new Error()` with no message)
- Async functions that neither await nor return promises (fire-and-forget without intent)
- Missing error boundaries in React component trees

---

### Concurrency & Async

**Python:**
- Mixing `async` and sync code incorrectly — blocking the event loop with sync I/O
- Missing `asyncio.gather` for independent concurrent operations done sequentially
- Thread safety issues — shared mutable state without locks
- Not using `async with` for async context managers
- `asyncio.create_task` without holding a reference (task can be garbage collected)

**JavaScript/TypeScript:**
- Sequential `await` for independent operations — should use `Promise.all`
- `Promise.all` where `Promise.allSettled` is needed (one rejection kills all)
- Missing `AbortController` for cancellable fetch/async operations
- Stale closure bugs — async callbacks referencing outdated state
- Race conditions between `useEffect` cleanup and async completion

---

### Maintainability

**General:**
- Functions or methods doing too many things (> ~30 lines is a smell, not a rule)
- Duplicated logic that should be extracted
- Naming that doesn't communicate intent
- Dead code, commented-out code, or TODO comments without linked issues
- Magic numbers or strings that should be named constants
- Inconsistent patterns compared to the rest of the codebase
- Deep nesting (> 3 levels) — consider early returns, guard clauses, or extraction
- Boolean parameters that change behavior — consider separate functions or options objects
- Implicit coupling between modules via shared global state

**Python:**
- Not using dataclasses, NamedTuples, or Pydantic models for structured data — passing around raw dicts/tuples
- `*args`/`**kwargs` pass-through that obscures the actual interface
- Circular imports or import-time side effects
- Missing `__repr__` or `__str__` on classes that appear in logs or debugging
- Using `os.path` when `pathlib.Path` would be cleaner

**JavaScript/TypeScript:**
- Callback hell or deeply nested `.then()` chains — should use async/await
- Object shapes used repeatedly without a type/interface definition
- Default exports mixed with named exports inconsistently
- Barrel files (`index.ts`) re-exporting everything — hurts tree-shaking and creates circular dependency risks
- Implicit dependencies on import order or side effects

---

## Step 3. Format the review

Structure your review as:

### Summary

One paragraph: what the change does, overall assessment (looks good / needs changes / has blockers).

### Findings

Group findings by severity:

**Blockers** — must fix before merge (security issues, data loss risks, correctness bugs)

**Should fix** — significant issues that will cause problems but aren't immediately dangerous

**Suggestions** — improvements that would make the code better but are not blocking

For each finding:
- Reference the specific file and line(s)
- Explain **what** the issue is and **why** it matters
- Show a concrete fix when possible — don't just point out problems

### Verdict

One of:
- **Approve** — no findings, or only minor suggestions
- **Approve with suggestions** — suggestions only, nothing blocking
- **Request changes** — has "should fix" or "blocker" findings

Keep the review focused and actionable. Developers should be able to read it and know exactly what to do.
