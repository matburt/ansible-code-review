# General Review Guide

Cross-language patterns to check in every review.

## Security

- Hardcoded secrets, API keys, tokens, or credentials
- User input flowing into SQL, shell commands, file paths, or templates without sanitization
- Overly permissive CORS, CSP, or access control
- Sensitive data in logs, error messages, or API responses
- Missing authentication or authorization checks on new endpoints/routes
- Insecure deserialization or use of `eval`/`exec`

## Correctness & Logic

### Boundary & Boolean Errors

```
# Off-by-one
❌ for i in range(len(items) - 1)   # skips last item
✅ for i in range(len(items))

# De Morgan's law violations
❌ not (a and b)  written as  (not a and not b)
✅ not (a and b)  ==  (not a or not b)
```

- Integer overflow, floating-point comparison (`0.1 + 0.2 !== 0.3`), precision loss
- Assumptions about ordering in unordered collections (dicts in older Python, Sets, Maps)
- Copy vs reference bugs — shallow copies of nested objects, unintended aliasing

### Resource Management

- Unclosed connections, file handles, event listeners, subscriptions
- Missing cleanup in error paths — resources acquired before a try block but only cleaned up in the happy path
- Incorrect error propagation — catching broad exceptions and losing the original context

## Performance

- Algorithmic complexity — O(n²) or worse where O(n) or O(n log n) is possible:

```python
# ❌ O(n²) — checking membership in a list inside a loop
for item in items:
    if item in other_list:  # O(n) lookup each time
        process(item)

# ✅ O(n) — convert to set first
other_set = set(other_list)
for item in items:
    if item in other_set:  # O(1) lookup
        process(item)
```

- String concatenation in loops instead of joining/building
- Repeated computation that should be cached or memoized
- Unnecessary data copying — deep clones where shallow would suffice
- Premature optimization that hurts readability without measurable benefit

## Error Handling

- Bare/empty exception handlers that swallow errors silently
- Error messages that don't include enough context to diagnose the problem
- Inconsistent error handling strategies within the same module
- Missing cleanup in error paths (finally blocks, context managers, try/catch/finally)
- Errors returned as magic values (`-1`, `null`, `""`) instead of thrown/raised — hides failures from callers

## Concurrency & Async

- Sequential operations on independent data that should be concurrent
- Shared mutable state accessed from multiple threads/tasks without synchronization
- Fire-and-forget async operations that silently drop errors
- Missing cancellation support for long-running operations

## Maintainability

- Functions doing too many things (>~30 lines is a smell, not a rule)
- Duplicated logic that should be extracted
- Naming that doesn't communicate intent
- Dead code, commented-out code, or TODO comments without linked issues
- Magic numbers or strings that should be named constants
- Deep nesting (>3 levels) — consider early returns, guard clauses, or extraction:

```python
# ❌ Deep nesting
def process(data):
    if data:
        if data.is_valid():
            if data.has_items():
                for item in data.items:
                    if item.is_active():
                        handle(item)

# ✅ Guard clauses
def process(data):
    if not data or not data.is_valid() or not data.has_items():
        return
    for item in data.items:
        if not item.is_active():
            continue
        handle(item)
```

- Boolean parameters that change function behavior — consider separate functions or options objects
- Implicit coupling between modules via shared global state

## Testing

- New logic paths without corresponding tests
- Tests that test implementation details instead of behavior:

```python
# ❌ Testing implementation — breaks if internals change
def test_user_creation():
    service = UserService()
    service.create("alice")
    assert service._users["alice"].created_at is not None  # private attr

# ✅ Testing behavior — survives refactoring
def test_user_creation():
    service = UserService()
    service.create("alice")
    user = service.get("alice")
    assert user is not None
    assert user.name == "alice"
```

- Missing edge case coverage for error paths, empty inputs, boundary values
- Mocking too much — tests that would pass even if the code is broken
- Flaky test patterns: time-dependent, order-dependent, or network-dependent
- Test names that don't describe the scenario and expected outcome
