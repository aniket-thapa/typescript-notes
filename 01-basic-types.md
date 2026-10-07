# Part 1: Basic Types

## 1.1 Type annotations vs type inference 🔴 MUST KNOW

**1. What it is**
An **annotation** is a type you write yourself (`: string`). **Inference** is TS working out the type from the value. Both give the same safety.

**2. Why it exists**
Writing every type by hand is noisy. TS infers what it can, and you annotate only where it cannot know.

**3. Code**

```ts
const city = 'Delhi'; // inferred: "Delhi" (const keeps the exact literal)
let age = 25; // inferred: number (let can change, so it widens)
let isActive = true; // inferred: boolean

let score: number; // annotation needed: no value yet
score = 90;

// Rule of thumb:
// - Variables with an initial value: let inference work.
// - Function parameters: always annotate.
// - Function return types: annotate on exported/public functions, optional on tiny local ones.
function double(n: number) {
  // return type inferred as number
  return n * 2;
}
```

**4. ❌ Wrong / ✅ Right**

```ts
let count: number = 0; // ❌ redundant, adds noise
let total = 0; // ✅ inference already says number

const items = []; // ⚠️ evolving type in strict mode, confusing
const names: string[] = []; // ✅ annotate empty arrays/objects
```

**5. Real-world use**
In Express you annotate function parameters and API boundaries (what goes in and out) and let inference handle local variables inside.

**6. Common mistakes & errors**
`let age = 25; age = "25";` gives `TS2322: Type 'string' is not assignable to type 'number'.` The type was inferred as `number` from the first value. Fix the value, or annotate a union (`number | string`) if both are truly valid.

**7. Interview tip**
_"When should you annotate explicitly?"_ Parameters, uninitialized variables, empty arrays/objects, and public function return types.

---

## 1.2 Primitive types 🔴 MUST KNOW

**1. What it is**
The basic building blocks: `string`, `number`, `boolean`, `bigint`, `symbol`. Always use **lowercase**.

**2. Why it exists**
They mirror JavaScript's primitive values. `Number`, `String` (capitalized) are wrapper objects and almost never what you want.

**3. Code**

```ts
const username: string = 'asha';
const price: number = 499.99; // integers and decimals are both number
const hex: number = 0xff;
const isPaid: boolean = false;
const bigId: bigint = 9007199254740993n; // needs target ES2020+
const key: symbol = Symbol('id'); // unique identifier, rare in app code
```

**4. ❌ Wrong / ✅ Right**

```ts
let a: String = 'hi'; // ❌ wrapper object type
let b: string = 'hi'; // ✅
```

**5. Real-world use**
Money in `number` can cause float errors (`0.1 + 0.2`). Many teams store prices in the smallest unit (paise/cents) as integers. `bigint` is for very large IDs, but note `JSON.stringify` cannot serialize it by default.

**6. Common mistakes & errors**
`const x: bigint = 10n` with a low target gives `TS2737: BigInt literals are not available when targeting lower than ES2020.` Fix: set `target` to `ES2020` or higher.

---

## 1.3 `null` and `undefined` 🔴 MUST KNOW

**1. What it is**
`undefined` means "no value assigned." `null` means "intentionally empty." With `strictNullChecks` (part of `strict`), they are **not** assignable to other types.

**2. Why it exists**
It removes the most common JS crash: using a value that is not there.

**3. Code**

```ts
let name: string = 'Asha';
name = null; // ❌ TS2322: Type 'null' is not assignable to type 'string'.

let nickname: string | null = null; // explicit "may be empty"
let middleName: string | undefined; // may be missing

// Handling them
nickname?.toUpperCase(); // optional chaining: undefined if null
const display = nickname ?? 'Anonymous'; // nullish coalescing: fallback only for null/undefined
if (nickname !== null) {
  nickname.toUpperCase(); // narrowed to string here
}
```

**4. ❌ Wrong / ✅ Right**

```ts
const label = '';
const a = label || 'Default'; // ❌ "" is falsy, so you get "Default" (often a bug)
const b = label ?? 'Default'; // ✅ keeps "", only replaces null/undefined
```

**5. Real-world use**
`User.findById()` returns `User | null`. TS forces you to handle "not found" (return 404) before using the user.

**6. Common mistakes & errors**
`TS18047: 'user' is possibly 'null'.` / `TS18048: 'x' is possibly 'undefined'.` Fix with a check, `?.`, or `??`, not with `!` unless you are truly certain.

**7. Interview tip**
_"Difference between `??` and `||`?"_ `||` replaces any falsy value (`0`, `""`, `false`); `??` only `null`/`undefined`.

---

## 1.4 `any` vs `unknown` vs `never` vs `void` 🔴 MUST KNOW

**1. What it is**

| Type      | Meaning                                                                     | Safe?  |
| --------- | --------------------------------------------------------------------------- | ------ |
| `any`     | Turns type checking **off** for that value                                  | ❌ No  |
| `unknown` | "Could be anything, you must check before use"                              | ✅ Yes |
| `never`   | A value that can never exist (function that never returns, impossible case) | ✅     |
| `void`    | A function that returns nothing useful                                      | ✅     |

**2. Why it exists**
Sometimes you truly do not know a type (JSON from outside, `catch` errors). `unknown` is the safe way. `any` is an escape hatch that spreads bugs.

**3. Code**

```ts
// any: no checks at all
let risky: any = 'hello';
risky.foo.bar(); // compiles, crashes at runtime

// unknown: must narrow first
let input: unknown = JSON.parse('{"id": 1}');
input.id; // ❌ TS18046: 'input' is of type 'unknown'.
if (typeof input === 'object' && input !== null && 'id' in input) {
  console.log(input.id); // ✅ narrowed
}

// void: function with no meaningful return
function logMessage(msg: string): void {
  console.log(msg);
}

// never: function that never finishes normally
function fail(message: string): never {
  throw new Error(message);
}
```

**4. ❌ Wrong / ✅ Right**

```ts
function parse(data: any) {
  return data.name;
} // ❌ any hides bugs
function parseSafe(data: unknown) {
  // ✅ forces a check
  if (typeof data === 'object' && data !== null && 'name' in data) {
    return data.name;
  }
  return undefined;
}
```

**5. Real-world use**
`catch (e)` is `unknown` in strict mode. `req.body` from outside should be treated as `unknown` until validated (Part 8). `never` powers exhaustive `switch` checks (Part 3).

**6. Common mistakes & errors**

- Using `any` to silence an error. It disables checking for everything that value touches.
- `const x: never = "a"` gives `TS2322: Type 'string' is not assignable to type 'never'.`

**7. Interview tip**
_"`any` vs `unknown`?"_ Both accept any value. `any` lets you do anything with it; `unknown` forces you to narrow first. Prefer `unknown`.

---

## 1.5 Arrays 🔴 MUST KNOW

**1. What it is**
A list of values of the same type. Two equivalent syntaxes: `string[]` and `Array<string>`.

**2. Why it exists**
So `.map`, `.filter` and indexing know the element type.

**3. Code**

```ts
const tags: string[] = ['node', 'react'];
const prices: Array<number> = [10, 20]; // same thing; you'll see it in older code
const matrix: number[][] = [
  [1, 2],
  [3, 4],
];
const mixed: (string | number)[] = ['a', 1]; // parentheses are required for unions

const readOnlyIds: readonly number[] = [1, 2, 3];
readOnlyIds.push(4); // ❌ TS2339: Property 'push' does not exist on type 'readonly number[]'.

const upper = tags.map((t) => t.toUpperCase()); // inferred string[]
```

**4. ❌ Wrong / ✅ Right**

```ts
const wrong: string | number[] = ['a']; // ❌ means: a string, OR an array of numbers
const right: (string | number)[] = ['a']; // ✅
```

**5. Real-world use**
`readonly` arrays protect props and state in React, since mutating state directly is a classic bug.

**6. Common mistakes**
`const list = []` then `list.push("a")` can confuse inference. Annotate: `const list: string[] = []`. With `noUncheckedIndexedAccess`, `tags[0]` is `string | undefined`.

---

## 1.6 Tuples 🟡 GOOD TO KNOW

**1. What it is**
An array with a **fixed length** and a **known type at each position**.

**2. Why it exists**
For small, ordered groups of mixed types, such as `useState` returning `[value, setter]`.

**3. Code**

```ts
const point: [number, number] = [10, 20];
const entry: [string, number] = ['age', 30];

// Named elements (documentation only, clearer in hovers)
type Range = [start: number, end: number];

// Optional element
type Pair = [string, number?];

// Readonly tuple
const coords: readonly [number, number] = [1, 2];

const bad: [string, number] = ['a', 1, 2];
// ❌ TS2322: Type '[string, number, number]' is not assignable to type '[string, number]'.
//    Source has 3 element(s) but target allows only 2.
point[2];
// ❌ TS2493: Tuple type '[number, number]' of length '2' has no element at index '2'.

const [x, y] = point; // destructuring works with full types
```

**4. ❌ Wrong / ✅ Right**
Tuples for records with many fields are hard to read. `[string, number, boolean, string]` says nothing. Use an object type.

**5. Real-world use**
A custom hook returning `[data, loading]`, or `Object.entries()` results (`[string, T][]`).

**6. Common mistake**
Tuple methods like `push` are allowed by the compiler on non-readonly tuples (a known TS limitation). Use `readonly` to prevent it.

---

## 1.7 Enums and why many teams prefer unions 🟡 GOOD TO KNOW

**1. What it is**
An `enum` is a named set of constants. It is one of the few TS features that creates **real JS code** (an object), not just types.

**2. Why it exists**
It predates union types. Today, a union of string literals does the same job with less machinery.

**3. Code**

```ts
// Numeric enum: auto-increments from 0, supports reverse lookup
enum Status {
  Pending,
  Active,
  Closed,
}
Status.Active; // 1
Status[1]; // "Active"

// String enum (the most common form): no reverse mapping
enum Role {
  Admin = 'ADMIN',
  User = 'USER',
}
const r: Role = Role.Admin;

// const enum: inlined at compile time, no runtime object
const enum Direction {
  Up = 'UP',
  Down = 'DOWN',
}

// The modern alternative: a union of literals
type RoleType = 'ADMIN' | 'USER';
const role: RoleType = 'ADMIN';

// Need the values at runtime too? Use an object + as const
const ROLES = { Admin: 'ADMIN', User: 'USER' } as const;
type RoleValue = (typeof ROLES)[keyof typeof ROLES]; // "ADMIN" | "USER"
```

|                                 | Enum                                         | Union of literals   |
| ------------------------------- | -------------------------------------------- | ------------------- |
| Runtime code emitted            | Yes (except `const enum`)                    | None                |
| Plain strings accepted          | String enums: **no** (`"ADMIN"` is rejected) | Yes                 |
| Works with type-stripping tools | Problematic                                  | Yes                 |
| Seen in old code                | Very common                                  | Increasingly common |

**4. ❌ Wrong / ✅ Right**

```ts
enum Role {
  Admin = 'ADMIN',
}
function setRole(r: Role) {}
setRole('ADMIN'); // ❌ TS2345: Argument of type '"ADMIN"' is not assignable to parameter of type 'Role'.
setRole(Role.Admin); // ✅
```

**5. Real-world use**
Mongoose schemas and Prisma often use string values like `"ADMIN"`. Unions match JSON from APIs directly, with no conversion.

**6. Common mistakes**

- `const enum` can break with `isolatedModules` and with tools like Vite/esbuild or Babel, which compile one file at a time. Avoid it in new code.
- Some tooling (Node's built-in type stripping, and the `erasableSyntaxOnly` flag in newer TS) does not support enums. This is version- and tooling-dependent, so check your setup.
- Opinion: many teams ban enums; many others use string enums happily. You must be able to **read** both.

**7. Interview tip**
_"Why prefer union types over enums?"_ No runtime output, plain strings are accepted, better compatibility with JSON and tooling.

---

## 1.8 Literal types and type aliases 🔴 MUST KNOW

**1. What it is**
A **literal type** is an exact value as a type (`"GET"`, `42`, `true`). A **type alias** (`type`) gives any type a name.

**2. Why it exists**
To restrict values to a small allowed set, and to avoid repeating long types.

**3. Code**

```ts
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type StatusCode = 200 | 400 | 404 | 500;
type ID = string | number;

function request(method: HttpMethod, url: string) {}
request('GET', '/users');
request('FETCH', '/users');
// ❌ TS2345: Argument of type '"FETCH"' is not assignable to parameter of type 'HttpMethod'.

// Widening trap
let method = 'GET'; // type: string (let widens)
const method2 = 'GET'; // type: "GET"
request(method, '/x'); // ❌ string is not assignable to HttpMethod
request(method2, '/x'); // ✅
```

**4. ❌ Wrong / ✅ Right**

```ts
type Status = string; // ❌ accepts any typo
type StatusOk = 'active' | 'inactive'; // ✅ typos are compile errors
```

**5. Real-world use**
Typing `req.method`, button `variant` props (`"primary" | "danger"`), and Redux action types.

**6. Common mistakes**
Object properties widen too: `const cfg = { method: "GET" }` gives `method: string`. Fix with `as const` (1.10) or an annotation.

---

## 1.9 Union and intersection basics 🔴 MUST KNOW

**1. What it is**

- **Union** `A | B`: the value is A **or** B.
- **Intersection** `A & B`: the value is A **and** B (all members combined).

**2. Why it exists**
Real data is often "one of several shapes" (union) or "a combination of shapes" (intersection).

**3. Code**

```ts
// Union
function format(id: string | number) {
  id.toFixed(2);
  // ❌ TS2339: Property 'toFixed' does not exist on type 'string | number'.
  //    Property 'toFixed' does not exist on type 'string'.
  if (typeof id === 'number') return id.toFixed(2); // ✅ narrowing
  return id.toUpperCase();
}

// Intersection
type Timestamps = { createdAt: Date; updatedAt: Date };
type User = { id: number; name: string };
type UserWithTimestamps = User & Timestamps;

const u: UserWithTimestamps = {
  id: 1,
  name: 'Asha',
  createdAt: new Date(),
  updatedAt: new Date(),
};

type Impossible = string & number; // never (no value is both)
```

**4. ❌ Wrong / ✅ Right**
With a union you may only use members **common to all** options. Narrow first, then use specific members.

**5. Real-world use**
`type ApiState = Loading | Success | Failure` (union) and `type AdminUser = User & { permissions: string[] }` (intersection). Discriminated unions come in Part 3.

**7. Interview tip**
_"Union vs intersection?"_ Union means "either", intersection means "both." Unions _narrow_ what you can safely access; intersections _add_ properties.

---

## 1.10 Assertions: `as`, `!`, `as const`, `satisfies` 🔴 MUST KNOW

**1. What it is**
Tools to tell the compiler something it cannot work out itself.

| Tool                          | Meaning                                                | Risk                                    |
| ----------------------------- | ------------------------------------------------------ | --------------------------------------- |
| `value as T`                  | "Trust me, this is T"                                  | No runtime check. A lie causes crashes. |
| `value!`                      | "This is not null/undefined"                           | Same: no runtime check.                 |
| `as const`                    | Make a value deeply readonly with exact literal types  | Safe                                    |
| `value satisfies T` (TS 4.9+) | Check against T **without** changing the inferred type | Safe                                    |

**2. Why it exists**
Sometimes you know more than the compiler (for example a DOM element's type). Others keep literal types or still validate shape.

**3. Code**

```ts
// as: DOM example
const input = document.getElementById('email') as HTMLInputElement;
input.value;

// as cannot convert unrelated types
const n = 'hello' as number;
// ❌ TS2352: Conversion of type 'string' to type 'number' may be a mistake...
const m = 'hello' as unknown as number; // double assertion: compiles, but a red flag

// ! non-null assertion
const el = document.querySelector('#app')!; // you promise it exists

// as const
const config = { env: 'prod', ports: [80, 443] } as const;
// type: { readonly env: "prod"; readonly ports: readonly [80, 443] }

const METHODS = ['GET', 'POST'] as const;
type Method = (typeof METHODS)[number]; // "GET" | "POST"

// satisfies
type Palette = Record<string, string | number[]>;
const colors = {
  red: [255, 0, 0],
  green: '#00ff00',
} satisfies Palette;
colors.red.map((c) => c * 2); // ✅ still known as number[]
colors.green.toUpperCase(); // ✅ still known as string
colors.blue = 'x'; // ❌ TS7053-style error: no such key (typo caught)
```

With `: Palette` instead of `satisfies`, `colors.red` would be `string | number[]` and `.map` would error.

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Lying to the compiler
const user = JSON.parse(text) as User; // no check that text really matches User

// ✅ Validate (Part 8), or at minimum treat as unknown first
const raw: unknown = JSON.parse(text);
```

**5. Real-world use**

- `as const` arrays drive dropdown options and derive union types.
- `satisfies` is excellent for config objects, route tables and theme objects.
- `req.user!` appears in controllers after auth middleware (Part 9). Know the risk.

**6. Common mistakes**
Using `as` or `!` to silence errors. Each one is a promise you must keep at runtime. If the promise is wrong, TS cannot save you.

**7. Interview tip**
_"`as` vs type annotation vs `satisfies`?"_ Annotation changes the variable's type; `as` overrides it unsafely; `satisfies` validates while keeping the precise inferred type.

---

### 📌 Key Takeaways

- Annotate parameters and boundaries; let inference handle the rest.
- Use lowercase primitives, and handle `null`/`undefined` explicitly with `?.`, `??` and checks.
- Prefer `unknown` over `any`; use `never` for impossible states and `void` for no return value.
- Tuples are for short, fixed, ordered data; objects are better for named fields.
- Prefer string literal unions (or `as const` objects) over enums in new code, but be able to read enums.
- `as` and `!` are unchecked promises; prefer `as const` and `satisfies`, which are safe.

✅ Part 1 complete.
