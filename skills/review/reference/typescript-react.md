# TypeScript & React Review Guide

Covers JavaScript/TypeScript language patterns and React-specific issues.

## Language Pitfalls

### Equality Coercion

```typescript
// ❌ Loose equality — type coercion surprises
"0" == false   // true
"" == false    // true
null == undefined  // true

// ✅ Always use strict equality
"0" === false  // false
"" === false   // false
```

### typeof null

```typescript
// ❌ typeof null is "object" — a famous JS bug
function process(value: unknown) {
  if (typeof value === "object") {
    value.toString(); // crashes if value is null!
  }
}

// ✅ Check null explicitly
function process(value: unknown) {
  if (value !== null && typeof value === "object") {
    value.toString();
  }
}
```

### this Binding

```typescript
// ❌ Method loses `this` when passed as callback
class Handler {
  name = "handler";
  handle() {
    console.log(this.name); // undefined when called as callback
  }
}
const h = new Handler();
button.addEventListener("click", h.handle); // `this` is the button!

// ✅ Arrow function or bind
button.addEventListener("click", () => h.handle());
// or
button.addEventListener("click", h.handle.bind(h));
// or define as arrow property
class Handler {
  name = "handler";
  handle = () => {
    console.log(this.name); // always correct
  };
}
```

### Array Iteration

```typescript
// ❌ for...in on arrays — iterates string keys, includes prototype
const arr = [10, 20, 30];
for (const i in arr) {
  console.log(typeof i); // "string", not number!
}

// ✅ for...of for values
for (const val of arr) {
  console.log(val); // 10, 20, 30
}

// ✅ forEach, map, etc. for functional style
arr.forEach((val, i) => console.log(i, val));
```

### parseInt

```typescript
// ❌ Without radix — "08" was octal in old engines
parseInt("08");    // 8 in modern JS, was 0 in old engines

// ✅ Always specify radix
parseInt("08", 10); // 8
```

### Async Pitfalls

```typescript
// ❌ forEach doesn't await — all fire in parallel, result is ignored
items.forEach(async (item) => {
  await processItem(item); // not actually awaited by forEach
});

// ✅ for...of for sequential
for (const item of items) {
  await processItem(item);
}

// ✅ Promise.all for parallel
await Promise.all(items.map((item) => processItem(item)));

// ❌ Unhandled rejection in event handler
button.onClick = async () => {
  const data = await fetchData(); // rejection goes unhandled
};

// ✅ Wrap in try/catch
button.onClick = async () => {
  try {
    const data = await fetchData();
  } catch (error) {
    showError(error);
  }
};
```

### Promise.all vs Promise.allSettled

```typescript
// ❌ One rejection kills everything
const results = await Promise.all([
  fetchUser(1),   // succeeds
  fetchUser(2),   // fails — entire Promise.all rejects
  fetchUser(3),   // never received even though it succeeded
]);

// ✅ When you need all results regardless of individual failures
const results = await Promise.allSettled([
  fetchUser(1),
  fetchUser(2),
  fetchUser(3),
]);
results.forEach((r) => {
  if (r.status === "fulfilled") handleUser(r.value);
  else logError(r.reason);
});
```

## TypeScript Types

### Avoid `any`

```typescript
// ❌ any defeats the type system
function parse(input: any): any {
  return JSON.parse(input);
}

// ✅ Use unknown for truly unknown types
function parse(input: string): unknown {
  return JSON.parse(input);
}

// ✅ Use generics when the type flows through
function parse<T>(input: string): T {
  return JSON.parse(input) as T;
}
```

### Type Assertions

```typescript
// ❌ Type assertion hiding a real type error
const user = apiResponse as User; // what if it's not actually a User?

// ✅ Runtime validation with type guard
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" &&
    obj !== null &&
    "id" in obj &&
    "name" in obj
  );
}

const data = apiResponse;
if (isUser(data)) {
  // data is safely typed as User here
}
```

### Discriminated Unions

```typescript
// ❌ String status with no type narrowing
interface Request {
  status: string;
  data?: unknown;
  error?: string;
}

// ✅ Discriminated union — exhaustive, type-safe
type Request =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User[] }
  | { status: "error"; error: string };

function handle(req: Request) {
  switch (req.status) {
    case "success":
      return req.data; // TypeScript knows data exists here
    case "error":
      return req.error; // TypeScript knows error exists here
  }
}
```

### Other Type Issues

- Enums where union types would be simpler: `type Status = "active" | "inactive"` vs `enum Status { Active, Inactive }`
- Inconsistent `null` vs `undefined` usage — pick one convention
- Overly complex generic types that hurt readability without adding safety

## React

### useEffect Dependencies

```typescript
// ❌ Missing dependency — stale closure over `userId`
useEffect(() => {
  fetchUser(userId).then(setUser);
}, []); // userId changes won't trigger refetch

// ✅ Include all dependencies
useEffect(() => {
  fetchUser(userId).then(setUser);
}, [userId]);

// ❌ Over-specified — object reference changes every render
useEffect(() => {
  doSomething(options);
}, [options]); // if options is created in render, this runs every time

// ✅ Destructure or memoize
const { page, limit } = options;
useEffect(() => {
  doSomething({ page, limit });
}, [page, limit]);
```

### Cleanup

```typescript
// ❌ No cleanup — memory leak and stale state updates
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);
  // interval runs forever, even after unmount
}, []);

// ✅ Return cleanup function
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);
  return () => clearInterval(id);
}, []);

// ✅ AbortController for fetch
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal })
    .then((r) => r.json())
    .then(setData)
    .catch((e) => {
      if (e.name !== "AbortError") throw e;
    });
  return () => controller.abort();
}, [url]);
```

### Derived State

```typescript
// ❌ Redundant state — fullName is derivable
const [firstName, setFirstName] = useState("");
const [lastName, setLastName] = useState("");
const [fullName, setFullName] = useState(""); // unnecessary

useEffect(() => {
  setFullName(`${firstName} ${lastName}`); // extra render!
}, [firstName, lastName]);

// ✅ Derive during render
const [firstName, setFirstName] = useState("");
const [lastName, setLastName] = useState("");
const fullName = `${firstName} ${lastName}`; // computed, no extra state
```

### Keys

```typescript
// ❌ Array index as key — breaks when list is reordered/filtered
{items.map((item, index) => (
  <ListItem key={index} item={item} />
))}

// ✅ Stable unique ID
{items.map((item) => (
  <ListItem key={item.id} item={item} />
))}
```

### Unnecessary Re-renders

```typescript
// ❌ New object reference every render — child always re-renders
function Parent() {
  return <Child style={{ color: "red" }} />;
  //                    ^ new object each render
}

// ✅ Stable reference
const style = { color: "red" }; // outside component, or useMemo

function Parent() {
  return <Child style={style} />;
}

// ❌ Inline function as prop
<Button onClick={() => handleClick(id)} />

// ✅ useCallback if child is memoized
const handleClick = useCallback(() => doThing(id), [id]);
<Button onClick={handleClick} />
```

### Other React Patterns

- Prop drilling through many layers — consider context or composition
- Components doing too many things — violating single responsibility
- Uncontrolled-to-controlled component switches (mixing `value` and `defaultValue`)
- Missing error boundaries in component trees
- State that should live in a parent or in a store
- Stale closure bugs — async callbacks referencing outdated state

## Performance

### Bundle Size

```typescript
// ❌ Imports entire library
import _ from "lodash";
const grouped = _.groupBy(items, "type");

// ✅ Import specific function
import groupBy from "lodash/groupBy";
const grouped = groupBy(items, "type");
```

### Other Performance Patterns

- DOM thrashing — reading layout properties then writing in a loop
- Spreading large arrays/objects repeatedly in reduce patterns
- Expensive computations on the main thread that could be deferred or memoized
- Missing React Query / SWR for data fetching — components re-fetching on every mount
- Event listeners or subscriptions without cleanup

## Error Handling

```typescript
// ❌ Catch swallows the error
fetchData()
  .then(process)
  .catch(() => {}); // silent failure

// ✅ Log or handle meaningfully
fetchData()
  .then(process)
  .catch((error) => {
    logger.error("Failed to fetch data", { error });
    showErrorToast("Something went wrong");
  });

// ❌ Error with no message
throw new Error();

// ✅ Descriptive error
throw new Error(`User ${userId} not found in project ${projectId}`);
```

- `try/catch` wrapping too much code — unclear what's expected to fail
- Async functions that neither await nor return promises (fire-and-forget without intent)
- Missing `AbortController` for cancellable fetch operations

## Security

- XSS vectors: `dangerouslySetInnerHTML`, unescaped user content
- Sensitive data stored in `localStorage` / `sessionStorage`
- Missing input validation on form submissions before API calls
- API keys or secrets in client-side code
- Environment variables without proper prefix awareness (`VITE_`, `NEXT_PUBLIC_`)

## Maintainability

- Callback hell or deeply nested `.then()` chains — use async/await
- Object shapes used repeatedly without a type/interface definition
- Default exports mixed with named exports inconsistently
- Barrel files (`index.ts`) re-exporting everything — hurts tree-shaking, creates circular deps
- Implicit dependencies on import order or module side effects
