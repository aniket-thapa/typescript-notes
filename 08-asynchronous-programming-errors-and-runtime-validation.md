# Part 8: Async, Errors & Runtime Validation

## 8.1 `Promise<T>` and async/await typing 🔴 MUST KNOW

**1. What it is**
A `Promise<T>` is a value that will arrive later. `T` is the type you get **after** awaiting. `await` unwraps `Promise<T>` into `T`. Part 2 introduced this; here are the deeper rules.

**2. Why it exists**
So TS can tell the difference between "the user" (`User`) and "a future user" (`Promise<User>`), and stop you from mixing them up.

**3. Code**

```ts
interface User {
  id: number;
  name: string;
}

function getUser(id: number): Promise<User> {
  return new Promise<User>((resolve, reject) => {
    if (id <= 0) reject(new Error('Invalid id'));
    else resolve({ id, name: 'Asha' });
  });
}

// Chaining: each .then changes T
getUser(1)
  .then((u) => u.name) // Promise<string>
  .then((n) => n.length); // Promise<number>

async function main() {
  const user = await getUser(1); // User
  console.log(user.name);
}

// Returning a Promise inside an async function does NOT nest it
async function load(): Promise<User> {
  return getUser(1); // ✅ not Promise<Promise<User>>
}
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ new Promise without a type argument: resolve accepts `unknown`
const p = new Promise((resolve) => resolve(5)); // Promise<unknown>
// ✅ Give T explicitly
const p2 = new Promise<number>((resolve) => resolve(5));

// ❌ Sequential awaits in a loop when requests are independent (slow)
for (const id of ids) {
  users.push(await getUser(id));
}
// ✅ Run in parallel (8.2)
const users2 = await Promise.all(ids.map((id) => getUser(id)));
```

**5. Real-world use**
Wrapping callback-style APIs (`setTimeout`, old libraries):

```ts
const sleep = (ms: number): Promise<void> =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));
```

**6. Common mistakes & errors**

- `TS2339: Property 'name' does not exist on type 'Promise<User>'.` You forgot `await`.
- `TS1308: 'await' expressions are only allowed within async functions and at the top levels of modules.` Mark the function `async`. Top-level `await` needs an ES module setup (Part 7).
- `.forEach(async ...)` does **not** wait for the callbacks. Use `for...of` or `Promise.all(array.map(...))`.

**7. Interview tip**
_"What is the type of `await somePromise`?"_ The `T` inside `Promise<T>`. `Awaited<T>` (Part 5a) is the type-level version.

---

## 8.2 `Promise.all`, `allSettled`, `race`, `any` 🔴 MUST KNOW

**1. What it is**
Helpers that run several promises together. TS keeps the **exact type of each position** when you pass a tuple or a literal array.

**3. Code**

```ts
interface Order {
  id: string;
  total: number;
}
declare function getUser(id: number): Promise<User>;
declare function getOrders(userId: number): Promise<Order[]>;
declare function getFlag(name: string): Promise<boolean>;

// all: fails fast, returns a tuple in the same order
const [user, orders, isBeta] = await Promise.all([
  getUser(1),
  getOrders(1),
  getFlag('beta'),
]);
// user: User, orders: Order[], isBeta: boolean

// all with an array built by map: array, not tuple
const users = await Promise.all([1, 2, 3].map((id) => getUser(id))); // User[]

// allSettled: never rejects; each result says what happened
const results = await Promise.allSettled([getUser(1), getUser(2)]);
for (const r of results) {
  if (r.status === 'fulfilled')
    console.log(r.value.name); // narrowed by discriminant (Part 3)
  else console.error(r.reason); // reason is any
}
```

| Method               | Resolves when                       | Rejects when                | Result type                 |
| -------------------- | ----------------------------------- | --------------------------- | --------------------------- |
| `Promise.all`        | All succeed                         | **Any** fails               | Tuple / array of values     |
| `Promise.allSettled` | All finish                          | Never                       | `PromiseSettledResult<T>[]` |
| `Promise.race`       | First finishes (success or failure) | First failure               | `T` of the winner           |
| `Promise.any`        | First **success**                   | All fail (`AggregateError`) | `T` of the winner           |

**4. ❌ Wrong / ✅ Right**

```ts
const [a, b] = await Promise.all([getUser(1), getOrders(1)]);
console.log(a.name); // ✅ a is User
const first = results[0];
first.value;
// ❌ TS2339: Property 'value' does not exist on type 'PromiseSettledResult<User>'.
//    Property 'value' does not exist on type 'PromiseRejectedResult'.   → check r.status first
```

**5. Real-world use**
A dashboard endpoint loading user, orders and stats at once. With `allSettled`, one failing widget doesn't break the whole page.

**6. Common mistakes**

- `Promise.all` only reports the **first** rejection; the other promises still run.
- Do not fire `Promise.all` over thousands of items (database or API limits). Process in batches.
- `Promise.any` and `AggregateError` need `lib` ES2021 or higher.

---

## 8.3 Typing `catch (e: unknown)` 🔴 MUST KNOW

**1. What it is**
Anything can be thrown in JS (`throw "oops"`, `throw 404`), so TS cannot know what `catch` receives. With `strict` (via `useUnknownInCatchVariables`, TS 4.4+) the variable is `unknown`. You must narrow it first.

**3. Code**

```ts
try {
  await getUser(-1);
} catch (e) {
  console.log(e.message);
  // ❌ TS18046: 'e' is of type 'unknown'.

  if (e instanceof Error) {
    console.error(e.message); // ✅ narrowed to Error
  } else {
    console.error('Unknown error', e);
  }
}

// A reusable helper: turn anything into a message
function getErrorMessage(error: unknown): string {
  if (error instanceof Error) return error.message;
  if (typeof error === 'string') return error;
  return 'Unknown error';
}

// Preserve the original error with `cause` (ES2022; needs lib/target ES2022)
try {
  await getUser(1);
} catch (e) {
  throw new Error('Failed to load profile', { cause: e });
}
```

**4. ❌ Wrong / ✅ Right**

```ts
catch (e: any) { console.log(e.message); }    // ⚠️ allowed, but unsafe: e might not be an Error
catch (e) { if (e instanceof Error) { /* ... */ } }   // ✅
```

`catch (e: unknown)` and bare `catch (e)` are the same under `strict`. You may annotate only `any` or `unknown`.

**5. Real-world use**
Every `try/catch` in a controller or service. Library errors need their own guards, for example Axios: `import { isAxiosError } from "axios"` then `if (isAxiosError(e)) e.response?.status`. Same for Mongoose (`e instanceof mongoose.Error.ValidationError`) and Prisma.

**6. Common mistakes**

- `instanceof Error` can fail for errors from another realm (iframes, some test setups, errors crossing module boundaries after a bundler duplicates a class). If it matters, check `"message" in e`.
- A `.catch((e) => ...)` callback on a promise gives `e: any` (a lib typing quirk), so annotate it as `unknown`.

**7. Interview tip**
_"Why is `e` unknown in `catch`?"_ Because JS allows throwing any value. `unknown` forces you to narrow before use.

---

## 8.4 Custom Error classes 🔴 MUST KNOW

**1. What it is**
Subclasses of `Error` that carry extra data, such as an HTTP status or a machine-readable `code`. You met a small version in Part 6.

**2. Why it exists**
A central error handler can use `instanceof` to decide the response: a `404` for "not found", a `400` for "bad input", a `500` for everything unexpected.

**3. Code**

```ts
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number = 500,
    public readonly code: string = 'INTERNAL_ERROR',
    options?: { cause?: unknown },
  ) {
    super(message, options); // passes `cause` along (ES2022)
    this.name = new.target.name; // "NotFoundError", not just "Error"
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string | number) {
    super(`${resource} ${id} not found`, 404, 'NOT_FOUND');
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields: Record<string, string[]>,
  ) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

// Usage in a service
async function findUserOrThrow(id: number): Promise<User> {
  const user = await getUser(id).catch(() => null);
  if (!user) throw new NotFoundError('User', id);
  return user;
}

// Handling
try {
  await findUserOrThrow(99);
} catch (e) {
  if (e instanceof NotFoundError)
    console.log(e.statusCode, e.code); // 404 NOT_FOUND
  else if (e instanceof AppError) console.log(e.statusCode);
  else throw e; // unknown error: don't swallow it
}
```

**4. ❌ Wrong / ✅ Right**

```ts
throw 'User not found'; // ❌ a string has no stack trace and no type
throw new NotFoundError('User', 99); // ✅

class Bad extends Error {
  constructor(public status: number) {
    /* no super call */
  }
  // ❌ TS2377: Constructors for derived classes must contain a 'super' call.
}
```

**5. Real-world use**
The Part 9 centralized error handler uses exactly this: `if (err instanceof AppError) res.status(err.statusCode).json(...)`, else a generic 500.

**6. Common mistakes**

- Compiling to an ES5 target breaks `instanceof` for `Error` subclasses (older code adds `Object.setPrototypeOf(this, new.target.prototype)`). With ES2015+ targets it just works.
- Never send `err.stack` or raw internal messages to clients in production.
- Do not catch an error just to hide it. Re-throw what you cannot handle.

**7. Interview tip**
_"Why create custom error classes?"_ So handlers can branch with `instanceof` and so errors carry typed metadata (status, code).

---

## 8.5 Result / Either pattern 🟡 GOOD TO KNOW

**1. What it is**
Instead of throwing, a function **returns** either a success or a failure. It is a discriminated union (Part 3), and the type signature itself shows that the function can fail.

**2. Why it exists**
`throw` is invisible in TS. Nothing in `function parseAge(s: string): number` tells the caller it can throw. A `Result` makes failure part of the contract and forces callers to handle it.

**3. Code**

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

type AgeError = 'NOT_A_NUMBER' | 'OUT_OF_RANGE';

function parseAge(input: string): Result<number, AgeError> {
  const n = Number(input);
  if (Number.isNaN(n)) return err('NOT_A_NUMBER');
  if (n < 0 || n > 150) return err('OUT_OF_RANGE');
  return ok(n);
}

const result = parseAge('42');
if (result.ok) {
  console.log(result.value + 1); // number
} else {
  console.error(result.error); // "NOT_A_NUMBER" | "OUT_OF_RANGE"
}
result.value;
// ❌ TS2339: Property 'value' does not exist on type 'Result<number, AgeError>'.
//    Property 'value' does not exist on type '{ ok: false; error: AgeError; }'.
```

**4. ❌ Wrong / ✅ Right**
Use `Result` for **expected** failures (validation, "not found", business rules). Use `throw` for **unexpected** ones (bugs, database down). Mixing both everywhere, or wrapping every function in `Result`, is a common over-engineering mistake. Libraries such as `neverthrow` and Effect provide richer versions.

**5. Real-world use**
Service methods returning `Result<User, "EMAIL_TAKEN" | "WEAK_PASSWORD">` so the controller maps each case to a status code. Zod's `safeParse` (8.8) returns this same shape.

**7. Interview tip**
_"Exceptions vs Result?"_ Exceptions are implicit and untyped; `Result` makes errors visible in the type. Mention that it is a team-style decision.

---

## 8.6 Why types vanish at runtime 🔴 MUST KNOW

**1. What it is**
Part 0 showed that types are erased. The consequence: **TS cannot check data that arrives while the program is running.** That data is `req.body`, `req.query`, `res.json()`, `JSON.parse`, `localStorage`, `process.env`, database rows, queue messages and form values.

**2. Why it matters**

```ts
interface CreateUserBody {
  email: string;
  age: number;
}

app.post('/users', (req: Request<{}, {}, CreateUserBody>, res) => {
  const { email, age } = req.body;
  console.log(email.toLowerCase()); // ✅ compiles, but...
  // A client sends { "email": 42 } → runtime crash: email.toLowerCase is not a function
  // A client sends {} → crash: Cannot read properties of undefined
});
```

The generic on `Request` is an **unchecked claim** (the same as `as`). Anyone can send anything. Beyond crashes, unvalidated input is how security bugs (injection, privilege escalation through extra fields like `"role": "admin"`) happen.

**3. The rule**
Treat all outside data as `unknown`, **validate** it, and let the validated result carry the type. Hand-written type guards (Part 3) work, but they are long and easily wrong. Use a schema library.

```ts
const raw: unknown = JSON.parse(text); // ✅ honest
const bad = JSON.parse(text) as User; // ❌ a lie the compiler believes
```

**7. Interview tip**
_"TS gives type safety, so why validate API input?"_ Types vanish at runtime, so TS cannot guarantee what a client or server actually sends.

---

## 8.7 Zod: schemas and `z.infer` 🔴 MUST KNOW

**1. What it is**
**Zod** is a library where you describe data **once** as a runtime schema. It then (a) validates real data and (b) gives you the matching TS type, so the type and the check can never drift apart.

**2. Why it exists**
Without it you write the interface, then a validator, then keep both in sync by hand.

**3. Code**

```bash
npm install zod
```

```ts
import { z } from 'zod';

const UserSchema = z.object({
  id: z.number().int().positive(),
  name: z.string().min(1),
  email: z.string().email(), // Zod 4 prefers z.email(); both exist, see version note
  age: z.number().int().min(0).max(150).optional(),
  role: z.enum(['admin', 'user']).default('user'),
  tags: z.array(z.string()),
  address: z.object({ city: z.string(), pin: z.string().regex(/^\d{6}$/) }),
});

// The TS type comes FROM the schema
type User = z.infer<typeof UserSchema>;
// { id: number; name: string; email: string; age?: number | undefined;
//   role: "admin" | "user"; tags: string[]; address: { city: string; pin: string } }

// Common building blocks
z.string().trim().toLowerCase();
z.number();
z.boolean();
z.date();
z.literal('ok');
z.union([z.string(), z.number()]);
z.nullable(z.string()); // string | null
z.string().nullish(); // string | null | undefined
z.record(z.string(), z.number()); // Record<string, number>
z.tuple([z.string(), z.number()]);

// Derive variants, like the utility types in Part 5
const CreateUserSchema = UserSchema.omit({ id: true });
const UpdateUserSchema = CreateUserSchema.partial();
type CreateUserInput = z.infer<typeof CreateUserSchema>;
```

**Input vs output types:** when a schema uses `.default()`, `.transform()` or `z.coerce`, the type going **in** differs from the type coming **out**. `z.input<typeof S>` and `z.output<typeof S>` (same as `z.infer`) give each. Forms need `input`, the rest of your app needs `output`.

**Version note:** Zod 4 (2025) moved some helpers to top level (`z.email()`, `z.url()`, `z.uuid()`), deprecated others (`.email()` on strings, `error.flatten()`), and changed error customization. Zod 3 code is still everywhere. Check which major version your project uses.

**4. ❌ Wrong / ✅ Right**

```ts
type Bad = z.infer<UserSchema>;
// ❌ TS2749: 'UserSchema' refers to a value, but is being used as a type here. Did you mean 'typeof UserSchema'?
type Good = z.infer<typeof UserSchema>; // ✅

// ❌ Writing the interface by hand next to the schema: two sources of truth
// ✅ Derive the type with z.infer
```

**5. Real-world use**
One schema file per feature, imported by the validation middleware (Part 9), React Hook Form (Part 10) and shared between both sides (Part 11).

**6. Common mistakes**

- Zod's inference needs `"strict": true`; with `strictNullChecks` off, optional fields become required.
- `z.number()` rejects `"5"`. Query strings and form fields are strings, so use `z.coerce.number()`. Caution: `z.coerce.boolean()` turns the string `"false"` into `true` (any non-empty string is truthy). Parse booleans from strings explicitly.
- By default `z.object` **strips unknown keys** from the output. Use `.strict()` to reject them, or `.passthrough()` to keep them.

**7. Interview tip**
_"What does `z.infer` do?"_ It extracts the static TS type from a runtime schema, giving one source of truth.

---

## 8.8 `parse` vs `safeParse` 🔴 MUST KNOW

**1. What it is**

- `schema.parse(data)`: returns the typed data, or **throws** a `ZodError`.
- `schema.safeParse(data)`: **never throws**; returns a `Result`-style object (8.5).

**3. Code**

```ts
const input: unknown = {
  id: 1,
  name: '',
  email: 'not-an-email',
  tags: [],
  address: { city: 'Delhi', pin: '110001' },
};

// parse: throws
try {
  const user = UserSchema.parse(input); // User
} catch (e) {
  if (e instanceof z.ZodError) {
    console.log(e.issues);
    // [{ path: ["name"], message: "Too small: ..." }, { path: ["email"], message: "Invalid ..." }]
  }
}

// safeParse: result object (discriminated union on `success`)
const result = UserSchema.safeParse(input);
if (!result.success) {
  console.log(result.error.issues); // ZodError
  // Zod 3: result.error.flatten().fieldErrors   |   Zod 4: z.flattenError(result.error).fieldErrors
} else {
  console.log(result.data.name); // typed User
}
result.data;
// ❌ TS2339: Property 'data' does not exist on type 'SafeParseReturnType<...>'.
//    Property 'data' does not exist on type 'SafeParseError<...>'.   → check result.success first
```

|                 | `parse`                                                       | `safeParse`                                                |
| --------------- | ------------------------------------------------------------- | ---------------------------------------------------------- |
| On failure      | Throws `ZodError`                                             | Returns `{ success: false, error }`                        |
| Best for        | Trusted-ish data where failure is a bug (env vars at startup) | User input where failure is normal (request bodies, forms) |
| Needs try/catch | Yes                                                           | No                                                         |

There are also `parseAsync` and `safeParseAsync` for schemas with async `.refine()` checks (like "is this email already taken?").

**4. ❌ Wrong / ✅ Right**

```ts
const user = UserSchema.parse(req.body); // ❌ uncaught throw becomes a 500 for the client's mistake
const r = UserSchema.safeParse(req.body); // ✅ reply 400 with the issues
if (!r.success) return res.status(400).json({ errors: r.error.issues });
```

**5. Real-world use: a typed, validated fetch**

```ts
async function fetchValidated<S extends z.ZodType>(
  url: string,
  schema: S,
): Promise<z.infer<S>> {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const json: unknown = await res.json();
  return schema.parse(json); // throws if the server returned the wrong shape
}
const me = await fetchValidated('/api/me', UserSchema); // User
```

This fixes the "return-only generic" problem from 4.9: the type is now backed by a real check.

**Environment variables (preview of Part 9):**

```ts
const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
  JWT_SECRET: z.string().min(32),
});
export const env = EnvSchema.parse(process.env); // crash at startup if anything is wrong
```

**6. Common mistakes**

- Validating, then using the **original** object instead of `result.data`. Only `result.data` is guaranteed typed (and stripped, coerced and defaulted).
- Writing custom checks with `.refine((v) => ..., { message, path })` but forgetting `path`, so the error is not attached to a field.
- Showing raw Zod issues to end users. Map them to friendly messages.

**7. Interview tip**
_"`parse` vs `safeParse`?"_ `parse` throws; `safeParse` returns a success/failure result. Use `safeParse` for expected bad input. Alternatives to Zod include Valibot, Yup, io-ts and ArkType; Zod is the most common.

---

### 📌 Key Takeaways

- `await` turns `Promise<T>` into `T`; `Promise.all` preserves per-position types, and `allSettled` results must be narrowed by `status`.
- `catch (e)` is `unknown` under `strict`; narrow with `instanceof Error` before using it, and re-throw what you cannot handle.
- Custom `AppError` subclasses with a status and a code let one central handler decide the response.
- `Result<T, E>` makes expected failures visible in types; use `throw` for unexpected ones.
- Types are erased, so `req.body`, `res.json()`, `JSON.parse` and `process.env` are unverified until validated. Generic or `as` claims are not checks.
- Define a Zod schema once, derive the type with `z.infer<typeof Schema>`, and use `safeParse` for user input. Always use `result.data`, and check which Zod major version you have.

✅ Part 8 complete.
