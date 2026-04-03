# API Design & Data Modeling Review Guide

Covers REST API design, database schema changes, and data contract patterns.

## API Design

### Breaking Changes

```python
# ❌ Removing a field from a response — breaks existing clients
class UserResponse(BaseModel):
    id: int
    name: str
    # email field removed — any client reading .email breaks

# ✅ Deprecate first, remove in next major version
class UserResponse(BaseModel):
    id: int
    name: str
    email: str | None = Field(None, deprecated=True)
```

- Renaming fields, changing types, or removing fields without API versioning
- Changing the shape of error responses that clients may parse
- Adding required fields to request bodies (existing clients won't send them)

### Status Codes

```python
# ❌ Wrong status codes
@app.post("/users")
async def create_user(data: UserCreate):
    user = await create(data)
    return user  # returns 200, should be 201

@app.delete("/users/{id}")
async def delete_user(id: int):
    await delete(id)
    return {"message": "deleted"}  # returns 200 with body, should be 204

# ✅ Correct status codes
@app.post("/users", status_code=201)
async def create_user(data: UserCreate):
    return await create(data)

@app.delete("/users/{id}", status_code=204)
async def delete_user(id: int):
    await delete(id)
```

Common mistakes:
- 200 for resource creation (should be 201)
- 200 with body for DELETE (should be 204 No Content)
- 500 for client errors (should be 4xx)
- 200 for partial failures in batch operations (consider 207 Multi-Status)

### Naming Consistency

- Mixing `camelCase` and `snake_case` across the API boundary — pick one convention for JSON and stick to it
- Inconsistent pluralization: `/user` vs `/users` vs `/user-list`
- Inconsistent URL patterns: `/getUsers` (RPC-style) mixed with `/users` (REST-style)

### Validation

```python
# ❌ No validation — crashes or stores garbage
@app.post("/users")
async def create_user(data: dict):
    return await db.insert(data)  # raw dict, no validation

# ✅ Pydantic model with validators
class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=0, le=150)

    @field_validator("name")
    @classmethod
    def name_must_not_be_blank(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Name cannot be blank")
        return v.strip()
```

```typescript
// ❌ No validation on frontend form submission
const handleSubmit = async () => {
  await api.post("/users", formData); // sends whatever is in the form
};

// ✅ Validate with Zod before sending
const UserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});

const handleSubmit = async () => {
  const result = UserSchema.safeParse(formData);
  if (!result.success) {
    setErrors(result.error.flatten());
    return;
  }
  await api.post("/users", result.data);
};
```

### Error Response Shape

```python
# ❌ Inconsistent error shapes
# endpoint A returns: {"error": "not found"}
# endpoint B returns: {"message": "invalid", "code": 422}
# endpoint C returns: {"detail": "forbidden"}

# ✅ Consistent error contract
class ErrorResponse(BaseModel):
    detail: str
    code: str | None = None
    errors: list[FieldError] | None = None

class FieldError(BaseModel):
    field: str
    message: str
```

### Other API Patterns

- Overly chatty APIs — multiple round trips for data that should come in one request
- Missing pagination on list endpoints — unbounded result sets
- Missing rate limiting on public or expensive endpoints
- N+1 at the API level — client making one request per item instead of a batch endpoint

## Database & Migrations

### Schema Changes

```python
# ❌ Adding a non-nullable column without a default — breaks existing rows
class AddPhoneNumber(Base):
    phone: Mapped[str]  # existing rows have no value!

# ✅ Add as nullable first, backfill, then make non-nullable
# Migration 1: Add nullable column
op.add_column("users", sa.Column("phone", sa.String(), nullable=True))

# Migration 2: Backfill data
op.execute("UPDATE users SET phone = '' WHERE phone IS NULL")

# Migration 3: Make non-nullable
op.alter_column("users", "phone", nullable=False)
```

### Migration Safety

- Model changes without a corresponding Alembic/Django migration
- Destructive migrations without a downgrade path
- Migrations that lock large tables (adding indexes without `CONCURRENTLY` in PostgreSQL)
- Data migrations mixed with schema migrations — should be separate
- Missing custom SQL documentation (the `# CUSTOM:` / `# END CUSTOM` pattern)

### Indexing

```python
# ❌ Filtering/joining on unindexed columns in a large table
# This query full-scans the table every time
stmt = select(Order).where(Order.customer_email == email)

# ✅ Add an index
class Order(Base):
    customer_email: Mapped[str] = mapped_column(index=True)
```

- Composite indexes for queries that filter on multiple columns
- Partial indexes for queries that always include a WHERE clause (e.g., `WHERE deleted_at IS NULL`)
- Unused indexes that slow down writes without benefiting reads

### Relationship Cascades

```python
# ❌ cascade="all, delete" can silently delete child records
class Parent(Base):
    children: Mapped[list["Child"]] = relationship(cascade="all, delete")

# ✅ Be explicit about cascade behavior
class Parent(Base):
    children: Mapped[list["Child"]] = relationship(
        cascade="all, delete-orphan",
        passive_deletes=True,  # let the DB handle cascades
    )
```

### Other Data Patterns

- Nullable columns added without defaults or a backfill strategy
- Storing derived data that could be computed — leads to stale data bugs
- Missing foreign key constraints — relying on application code for referential integrity
- Using string columns for structured data (JSON, enums) without validation
