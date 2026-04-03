# Python Review Guide

Covers Python language patterns, Django, FastAPI, and SQLAlchemy/SQLModel.

## Language Pitfalls

### Mutable Default Arguments

```python
# ❌ Shared across calls — a classic Python gotcha
def add_item(item, items=[]):
    items.append(item)
    return items

add_item(1)  # [1]
add_item(2)  # [1, 2] — not [2]!

# ✅ Use None sentinel
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

Also applies to mutable class attributes:

```python
# ❌ Shared across all instances
class User:
    permissions = []  # every User shares this list

# ✅ Initialize in __init__ or use dataclass
@dataclass
class User:
    permissions: list = field(default_factory=list)
```

### Late Binding Closures

```python
# ❌ All lambdas capture the same variable by reference
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]  # [2, 2, 2]

# ✅ Capture by value with default argument
funcs = [lambda i=i: i for i in range(3)]
[f() for f in funcs]  # [0, 1, 2]
```

### Identity vs Equality

```python
# ❌ `is` compares identity, not value
if x is 1000:  # may fail — CPython only caches -5 to 256
    pass

# ✅ Use == for value comparison
if x == 1000:
    pass

# ✅ `is` is correct for None and sentinel objects
if x is None:
    pass
```

### Generator Exhaustion

```python
# ❌ Generator can only be iterated once
gen = (x for x in range(10))
total = sum(gen)      # 45
total2 = sum(gen)     # 0 — silently empty!

# ✅ Use a list if you need multiple passes
items = list(x for x in range(10))
# Or recreate the generator
```

### Datetime Timezone Awareness

```python
# ❌ Naive datetime — ambiguous, breaks comparisons with aware datetimes
from datetime import datetime
now = datetime.now()

# ✅ Always use timezone-aware datetimes
from datetime import datetime, timezone
now = datetime.now(timezone.utc)
```

### Exception Handling

```python
# ❌ Bare except catches KeyboardInterrupt, SystemExit
try:
    risky()
except:
    pass

# ❌ Too broad, swallows everything
try:
    risky()
except Exception:
    pass

# ✅ Catch specific exceptions
try:
    risky()
except (ValueError, KeyError) as e:
    logger.error("Failed to process: %s", e)
    raise

# ❌ Loses the exception chain
try:
    external_api.call()
except APIError as e:
    raise RuntimeError("API failed")  # original traceback lost

# ✅ Preserve chain with `from`
try:
    external_api.call()
except APIError as e:
    raise RuntimeError("API failed") from e
```

### Logging Performance

```python
# ❌ f-string evaluated even if log level is disabled
logger.debug(f"Processing {len(expensive_list)} items: {expensive_list}")

# ✅ Use % formatting — deferred evaluation
logger.debug("Processing %d items: %s", len(expensive_list), expensive_list)
```

### Other Pitfalls

- Missing `__all__` or wildcard imports (`from mod import *`) polluting namespaces
- Incorrect `super()` usage in multiple inheritance (MRO issues)
- `*args`/`**kwargs` pass-through that obscures the actual interface
- Circular imports or import-time side effects
- Missing `__repr__` on classes that appear in logs or debugging
- Using `os.path` when `pathlib.Path` would be cleaner
- Not using dataclasses/NamedTuples/Pydantic for structured data — passing raw dicts/tuples

## Performance

### Collection Choice

```python
# ❌ O(n) membership check
if item in large_list:
    pass

# ✅ O(1) membership check
large_set = set(large_list)
if item in large_set:
    pass
```

### String Building

```python
# ❌ O(n²) — creates new string each iteration
result = ""
for item in large_list:
    result += str(item)

# ✅ O(n) — join at the end
result = "".join(str(item) for item in large_list)
```

### Generators vs Lists

```python
# ❌ Materializes entire sequence in memory
squares = [x**2 for x in range(10_000_000)]
total = sum(squares)

# ✅ Generator — constant memory
total = sum(x**2 for x in range(10_000_000))
```

### Other Performance Patterns

- Repeated dictionary/attribute lookups in tight loops — hoist to a local variable
- Creating regex objects inside loops instead of compiling once with `re.compile()`
- Expensive operations inside loops that could be batched (especially DB queries)

## Async / Concurrency

```python
# ❌ Blocking the event loop with sync I/O in an async function
async def get_data():
    data = requests.get(url)  # blocks!
    return data.json()

# ✅ Use async HTTP client
async def get_data():
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.json()

# ❌ Sequential awaits for independent operations
async def fetch_all():
    users = await get_users()      # waits
    orders = await get_orders()    # waits again
    return users, orders

# ✅ Concurrent with gather
async def fetch_all():
    users, orders = await asyncio.gather(
        get_users(),
        get_orders(),
    )
    return users, orders
```

- `asyncio.create_task` without holding a reference — task can be garbage collected
- Not using `async with` for async context managers
- Thread safety — shared mutable state without locks in threaded code
- `time.sleep()` in async code — blocks the entire event loop, use `asyncio.sleep()`
- Missing `CancelledError` handling in tasks that need cleanup

## Django

### N+1 Queries

```python
# ❌ N+1 — one query per author
for book in Book.objects.all():
    print(book.author.name)  # hits DB each iteration

# ✅ Eager load with select_related (ForeignKey/OneToOne)
for book in Book.objects.select_related("author").all():
    print(book.author.name)  # single JOIN query

# ✅ prefetch_related for ManyToMany/reverse ForeignKey
for author in Author.objects.prefetch_related("books").all():
    print(author.books.count())
```

### Other Django Patterns

- Fat views — business logic should live in services/managers, not views
- Missing `select_for_update` in write-heavy concurrent paths
- Querysets evaluated multiple times — should be cached or sliced
- Custom managers/querysets not used where they'd reduce duplication
- Missing migration for model changes
- Signals used where explicit calls would be clearer and more testable
- Missing `csrf_exempt` justification

## FastAPI

### Dependency Injection

```python
# ❌ Manual instantiation
@app.get("/users")
async def get_users():
    db = Database()  # creates new instance every time
    return await db.get_users()

# ✅ Use Depends
async def get_db() -> AsyncGenerator[Database, None]:
    db = Database()
    try:
        yield db
    finally:
        await db.close()

@app.get("/users")
async def get_users(db: Database = Depends(get_db)):
    return await db.get_users()
```

### Response Models

```python
# ❌ Leaks internal fields (password_hash, internal IDs)
@app.get("/users/{id}")
async def get_user(id: int, db: Session = Depends(get_db)):
    return db.get(User, id)  # returns entire ORM object

# ✅ Explicit response model
class UserResponse(BaseModel):
    id: int
    name: str
    email: str

@app.get("/users/{id}", response_model=UserResponse)
async def get_user(id: int, db: Session = Depends(get_db)):
    return db.get(User, id)
```

### Other FastAPI Patterns

- Pydantic models with overly permissive types (`Any`, `dict`) where a specific schema is known
- Background tasks that should use a proper task queue (Celery, Temporal) for anything non-trivial
- Missing or incorrect status codes (200 for creation instead of 201, etc.)
- Async endpoints calling sync ORM/IO without `run_in_executor`
- Missing request validation — relying on raw dict access instead of Pydantic models

## SQLAlchemy / SQLModel

### Session Management

```python
# ❌ Session leaked — not closed on error
def get_user(user_id: int):
    session = Session()
    user = session.get(User, user_id)
    return user  # session never closed!

# ✅ Context manager
def get_user(user_id: int):
    with Session() as session:
        return session.get(User, user_id)
```

### Async Pitfalls

```python
# ❌ Lazy loading in async context — raises MissingGreenlet error
async def get_user_orders(user_id: int):
    user = await session.get(User, user_id)
    return user.orders  # lazy load fails in async!

# ✅ Eager load with selectinload
from sqlalchemy.orm import selectinload

async def get_user_orders(user_id: int):
    stmt = select(User).where(User.id == user_id).options(selectinload(User.orders))
    result = await session.execute(stmt)
    user = result.scalar_one()
    return user.orders
```

### Other SQLAlchemy Patterns

- Missing `Mapped[]` type annotations on model columns
- Alembic migrations not matching model changes
- Relationship cascades that could cause unintended deletes
- Unbounded queries — missing pagination or `LIMIT`
- Missing database indexes for columns used in filters/joins

## Security

- Raw SQL or string-formatted queries instead of parameterized:

```python
# ❌ SQL injection
query = f"SELECT * FROM users WHERE name = '{user_input}'"

# ✅ Parameterized
session.execute(text("SELECT * FROM users WHERE name = :name"), {"name": user_input})
```

- `subprocess` calls with `shell=True` or unsanitized input
- `pickle` / `yaml.load()` without safe loaders (`yaml.safe_load()`)
- Unrestricted file uploads or path traversal in file handling
