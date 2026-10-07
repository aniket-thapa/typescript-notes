# Part 3: Objects, Interfaces & Type Narrowing

## 3.1 Object types 🔴 MUST KNOW

**1. What it is**
An object type lists the property names an object must have and the type of each one.

**2. Why it exists**
Most data in Node and React is an object (a user, an order, a request body). Object types make a typo like `user.nmae` a compile error.

**3. Code**

```ts
// Inline object type
function printUser(user: { id: number; name: string }): void {
  console.log(user.id, user.name);
}

// Named with a type alias
type Product = {
  id: number;
  title: string;
  price: number;
};

const keyboard: Product = { id: 1, title: 'Keyboard', price: 999 };

// Nested objects
type Order = {
  id: string;
  customer: { id: number; name: string };
  items: Product[];
};
```

**Careful with `object`, `{}` and `Object`:**

| Type                      | Meaning                                                       | Use?                   |
| ------------------------- | ------------------------------------------------------------- | ---------------------- |
| `object`                  | Any non-primitive (object, array, function)                   | Rarely                 |
| `{}`                      | Any value except `null`/`undefined` (including `5` and `"a"`) | Avoid                  |
| `Object`                  | Same problem as `{}`                                          | Avoid                  |
| `Record<string, unknown>` | An object with string keys and unknown values                 | Good for "some object" |

**4. ❌ Wrong / ✅ Right**

```ts
const p: Product = { id: 1, title: 'Mouse' };
// ❌ TS2741: Property 'price' is missing in type '{ id: number; title: string; }' but required in type 'Product'.
const p2: Product = { id: 1, title: 'Mouse', price: 499 }; // ✅
```

**5. Real-world use**
Every API response, database document and React props object is typed this way.

**6. Common mistakes**
Typing a value as `{}` thinking it means "empty object." It accepts almost anything. Use `Record<string, never>` for a truly empty object, or just `object`.

---

## 3.2 `interface` vs `type` 🔴 MUST KNOW

**1. What it is**
Both name an object shape. `interface` only describes object-like shapes. `type` can name _any_ type (unions, primitives, tuples, function types).

**2. Why it matters**
You will see both in every codebase. Interviewers ask about it constantly.

**3. Code**

```ts
interface UserI {
  id: number;
  name: string;
}

type UserT = {
  id: number;
  name: string;
};

// Only `type` can do these:
type Id = string | number; // union
type Pair = [number, number]; // tuple
type Handler = (e: Event) => void; // function alias (interface can via call signature)
type Keys = keyof UserT; // computed from other types
```

| Feature                                              | `interface`                   | `type`                       |
| ---------------------------------------------------- | ----------------------------- | ---------------------------- |
| Object shapes                                        | ✅                            | ✅                           |
| Unions, tuples, primitives, mapped/conditional types | ❌                            | ✅                           |
| Extending                                            | `extends`                     | `&` (intersection)           |
| Declaration merging (same name twice combines)       | ✅                            | ❌ (error)                   |
| Error messages for conflicts                         | Clearer at the `extends` line | Can be harder to read        |
| Can be implemented by a class                        | ✅                            | ✅ (if it is an object type) |
| Performance with deep inheritance                    | Slightly better (cached)      | Fine in practice             |

**Which to use (team opinion, so follow your team's rule):**

- Common convention A: `interface` for object shapes and public APIs, `type` for everything else.
- Common convention B: `type` everywhere, `interface` only when you need declaration merging.
- Pick one and stay consistent. In an interview, say you know both and follow the project's convention.

**4. ❌ Wrong / ✅ Right**

```ts
type User = { id: number };
type User = { name: string };
// ❌ TS2300: Duplicate identifier 'User'.

interface Account {
  id: number;
}
interface Account {
  name: string;
} // ✅ merged: Account has both id and name
```

**5. Real-world use**
React props: `interface ButtonProps { ... }` or `type ButtonProps = { ... }`. Both are normal. Libraries use `interface` so that you can extend their types.

**7. Interview tip**
_"Interface vs type?"_ Both describe objects. `type` is more flexible (unions, mapped types). `interface` supports declaration merging and `extends`. Then state which your team prefers.

---

## 3.3 Optional and readonly properties 🔴 MUST KNOW

**1. What it is**

- `prop?: T`: may be missing. Its type becomes `T | undefined`.
- `readonly prop: T`: can be set at creation, never reassigned.

**3. Code**

```ts
interface User {
  readonly id: number; // never changes after creation
  name: string;
  phone?: string; // optional
}

const user: User = { id: 1, name: 'Asha' };
user.id = 2;
// ❌ TS2540: Cannot assign to 'id' because it is a read-only property.
user.name = 'Asha K'; // ✅

const length = user.phone.length;
// ❌ TS18048: 'user.phone' is possibly 'undefined'.
const safe = user.phone?.length ?? 0; // ✅
```

**4. ❌ Wrong / ✅ Right**
`readonly` is **shallow** and **compile-time only**.

```ts
interface Config {
  readonly tags: string[];
}
const cfg: Config = { tags: ['a'] };
cfg.tags = []; // ❌ blocked
cfg.tags.push('b'); // ⚠️ allowed! the array itself is not readonly
// ✅ Use readonly string[] (or ReadonlyArray<string>) for a truly locked array
```

**5. Real-world use**
`readonly id` and `createdAt` on database models. Optional fields for PATCH bodies (Part 5 shows `Partial<T>`).

**6. Common mistakes**

- Optional (`phone?: string`) is not the same as `phone: string | undefined`. The first lets you omit the key; the second requires the key to be present. (With `exactOptionalPropertyTypes`, the difference gets stricter. It is a rarely used flag.)

---

## 3.4 Index signatures 🟡 GOOD TO KNOW

**1. What it is**
A way to type an object whose **key names are not known in advance**, only the key type and value type.

**3. Code**

```ts
interface StringMap {
  [key: string]: string;
}
const headers: StringMap = { 'content-type': 'application/json' };

// Mixing known keys with an index signature: known keys must match the index type
interface Scores {
  [player: string]: number;
  total: number; // ✅ number matches
  // label: string;   // ❌ TS2411: Property 'label' of type 'string' is not assignable to 'string' index type 'number'.
}

// Usually clearer: Record
type Cache = Record<string, number>;

// With noUncheckedIndexedAccess, lookups include undefined
const cache: Cache = {};
const hit = cache['a']; // number | undefined
```

**4. ❌ Wrong / ✅ Right**

```ts
const data = { a: 1, b: 2 };
function read(key: string) {
  return data[key];
  // ❌ TS7053: Element implicitly has an 'any' type because expression of type 'string' can't be used to index type '{ a: number; b: number; }'.
}
function readOk(key: keyof typeof data) {
  return data[key];
} // ✅ key limited to "a" | "b"
```

**5. Real-world use**
Query objects, HTTP headers, translation dictionaries, in-memory caches.

**6. Common mistake**
Using an index signature when the keys are actually a fixed set. Use a union of keys (`Record<"en" | "hi", string>`) so missing keys are caught.

---

## 3.5 Extending and merging interfaces 🔴 MUST KNOW

**1. What it is**

- **Extending:** build a new interface from existing ones.
- **Declaration merging:** declaring the same `interface` name twice combines them into one.

**3. Code**

```ts
interface BaseEntity {
  id: number;
  createdAt: Date;
}
interface User extends BaseEntity {
  name: string;
}
interface Admin extends User {
  permissions: string[];
}
interface Auditable {
  updatedBy: string;
}
interface Post extends BaseEntity, Auditable {
  title: string;
} // multiple parents

// Type alias equivalent
type UserT = BaseEntity & { name: string };

// Declaration merging
interface Window {
  appVersion: string; // adds to the built-in browser Window type
}
```

**4. ❌ Wrong / ✅ Right**

```ts
interface Animal {
  legs: number;
}
interface Snake extends Animal {
  legs: string;
}
// ❌ TS2430: Interface 'Snake' incorrectly extends interface 'Animal'. Types of property 'legs' are incompatible.
```

An extended interface may only make a property **narrower**, never incompatible. With `&` on type aliases, conflicting properties silently become `never` instead of an error at the declaration, which is harder to debug.

**5. Real-world use**
Declaration merging is how you add `user` to Express's `Request` (full setup in Part 7):

```ts
declare global {
  namespace Express {
    interface Request {
      user?: { id: string; role: string };
    }
  }
}
```

**7. Interview tip**
_"What is declaration merging?"_ Two interfaces with the same name in the same scope combine into one. Type aliases cannot do this.

---

## 3.6 Structural typing and excess property checks 🔴 MUST KNOW

**1. What it is**
TS compares **shape**, not names (also called duck typing). If a value has the required properties, it fits, even if it has more. The exception is a **fresh object literal** assigned directly: extra properties are flagged as probable typos.

**2. Why it exists**
Structural typing lets unrelated code interoperate without shared base classes. The excess check catches misspelled keys.

**3. Code**

```ts
interface Point {
  x: number;
  y: number;
}

const p3 = { x: 1, y: 2, z: 3 };
const a: Point = p3; // ✅ extra `z` is fine: it's a variable, not a fresh literal

const b: Point = { x: 1, y: 2, z: 3 };
// ❌ TS2353: Object literal may only specify known properties, and 'z' does not exist in type 'Point'.

function draw(p: Point) {}
draw({ x: 1, y: 2, color: 'red' }); // ❌ same excess property error
draw(p3); // ✅

// Names don't matter, shapes do
class Cat {
  name = 'Tom';
}
interface Named {
  name: string;
}
const n: Named = new Cat(); // ✅
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Silencing the check with a cast hides real typos
draw({ x: 1, y: 2, colour: 'red' } as Point);
// ✅ Fix the type if the property is legitimate
interface ColoredPoint extends Point {
  color?: string;
}
```

**5. Real-world use**
A function taking `{ id: number }` accepts any full `User`. But watch out: a `req.body` object with extra fields passes type checks silently. Runtime validation (Zod, Part 8) is what strips unknown fields.

**6. Common mistake**
Assuming TS rejects extra properties everywhere. It only does so for fresh literals. Extra data from the network always gets through.

**7. Interview tip**
_"Is TS nominal or structural?"_ Structural.

---

## 3.7 Union types and discriminated unions 🔴 MUST KNOW

**1. What it is**
A **discriminated union** is a union of object types that share one property (the **discriminant**) with a unique literal value in each member. Checking that property tells TS exactly which member you have.

**2. Why it exists**
It models "one of several states" safely, replacing messy objects full of optional fields and booleans.

**3. Code**

```ts
// ❌ Messy: which fields are valid together?
type BadState = { loading: boolean; data?: string[]; error?: string };

// ✅ Discriminated union: each state carries only what it can have
type ApiState =
  | { status: 'loading' }
  | { status: 'success'; data: string[] }
  | { status: 'error'; message: string };

function render(state: ApiState): string {
  switch (state.status) {
    case 'loading':
      return 'Loading...';
    case 'success':
      return state.data.join(', '); // data exists only here
    case 'error':
      return `Failed: ${state.message}`;
  }
}

state.data;
// ❌ TS2339: Property 'data' does not exist on type 'ApiState'.
//    Property 'data' does not exist on type '{ status: "loading"; }'.
```

**5. Real-world use**

- React: data-fetching state, form wizard steps, reducer actions (`{ type: "ADD"; payload: Todo } | { type: "CLEAR" }`).
- Express: API response `{ success: true; data: T } | { success: false; error: string }`.

**6. Common mistakes**

- The discriminant must be a **literal type** (`"success"`, not `string`).
- Destructuring before checking can lose narrowing in older TS. Narrow first, then destructure.

**7. Interview tip**
_"What is a discriminated union?"_ A union where each member has a common literal property used to narrow the type. It is one of the most commonly asked TS topics.

---

## 3.8 Type narrowing 🔴 MUST KNOW

**1. What it is**
**Narrowing** means TS shrinks a broad type (like `string | number | null`) to a specific one inside a code block, based on checks it understands. TS reads your `if`, `switch` and `return` statements to do this (control flow analysis).

**3. Code: the built-in narrowing tools**

```ts
// typeof: for primitives
function pad(v: string | number): string {
  if (typeof v === 'number') return v.toFixed(2); // number here
  return v.trim(); // string here
}

// instanceof: for class instances (not interfaces)
function describe(e: Error | string): string {
  return e instanceof Error ? e.message : e;
}

// in: check whether a property exists
type Cat = { meow: () => void };
type Dog = { bark: () => void };
function speak(pet: Cat | Dog) {
  if ('meow' in pet) pet.meow();
  else pet.bark();
}

// Truthiness: removes null, undefined, 0, "", false
function len(s?: string): number {
  if (!s) return 0; // ⚠️ also catches ""
  return s.length; // string here
}

// Equality: === narrows both sides
function same(a: string | number, b: string | boolean) {
  if (a === b) {
    a;
    b;
  } // both are string here
}

// Array.isArray
function toArray(x: string | string[]): string[] {
  return Array.isArray(x) ? x : [x];
}
```

**4. ❌ Wrong / ✅ Right**

```ts
function total(n?: number) {
  if (!n) return 'none'; // ❌ treats 0 as "none"
  return n;
}
function totalOk(n?: number) {
  if (n === undefined) return 'none'; // ✅ 0 is valid
  return n;
}
```

**5. Real-world use**
Handling `User | null` from a database lookup, checking `typeof req.query.page === "string"` (query values can be `string | string[] | ParsedQs | ...`), and narrowing caught errors with `e instanceof Error`.

**6. Common mistakes & errors**

- `typeof x === "object"` is also true for `null`. Check `x !== null` too.
- `TS2693: 'User' only refers to a type, but is being used as a value here.` You used `instanceof` with an interface. Types are erased (Part 0), so use a type guard or `in`.
- Narrowing is lost inside callbacks for `let` variables or mutated values. Copy to a `const` first.

**7. Interview tip**
_"Name ways to narrow a type."_ `typeof`, `instanceof`, `in`, truthiness, equality, discriminant checks, and custom type guards.

---

## 3.9 Custom type guards (`is`) 🔴 MUST KNOW

**1. What it is**
A function whose return type is a **type predicate** (`value is T`). If it returns `true`, TS treats `value` as `T` afterward.

**2. Why it exists**
When the built-in checks are not enough, such as validating `unknown` data or telling interface-typed objects apart.

**3. Code**

```ts
interface User {
  id: number;
  name: string;
}

function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof value.id === 'number' &&
    'name' in value &&
    typeof value.name === 'string'
  );
}

const data: unknown = JSON.parse('{"id":1,"name":"Asha"}');
if (isUser(data)) {
  console.log(data.name); // User here
}

// Filtering nulls out of an array
const maybeUsers: (User | null)[] = [{ id: 1, name: 'A' }, null];
const users = maybeUsers.filter((u): u is User => u !== null); // User[]
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ The compiler trusts your predicate completely. A wrong body is a silent lie:
function isString(x: unknown): x is string {
  return true; // compiles, but wrong
}
```

Keep guards small and correct. For large shapes, use Zod (Part 8) instead of hand-written guards.

**Version note:** TS 5.5 can infer simple type predicates, so `.filter((u) => u !== null)` now narrows to `User[]` in many cases. Older code and older TS need the explicit `u is User`.

**5. Real-world use**
Checking that a caught error is an Axios error, or that a JSON message has the expected `type` field.

**7. Interview tip**
_"What does `value is T` do?"_ It tells TS to narrow `value` to `T` when the function returns true. TS does not verify the body.

---

## 3.10 Assertion functions (`asserts`) 🟡 GOOD TO KNOW

**1. What it is**
A function that **throws if a condition fails**. If it returns normally, TS treats the condition as true afterward.

**3. Code**

```ts
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== 'string') throw new Error('Expected string');
}

function assert(condition: unknown, message: string): asserts condition {
  if (!condition) throw new Error(message);
}

function process(input: unknown, user: { name: string } | null) {
  assertIsString(input);
  input.toUpperCase(); // string from here on

  assert(user !== null, 'User required');
  console.log(user.name); // no longer null
}
```

**6. Common mistakes & errors**

- `TS2775: Assertions require every name in the call target to be declared with an explicit type annotation.` Assertion functions declared as `const assert = (...) => {}` must have an explicit type. Use a `function` declaration instead.
- Node's built-in `assert` from `node:assert` already has assertion typings.

**5. Real-world use**
Config loading: `assertDefined(process.env.JWT_SECRET, "JWT_SECRET missing")`, so later code gets a plain `string`.

---

## 3.11 Exhaustive checking with `never` 🔴 MUST KNOW

**1. What it is**
After you handle every member of a union, the leftover type is `never`. Assigning it to a `never` variable proves you covered everything. If you add a new member later, the compiler flags every place you forgot.

**3. Code**

```ts
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; size: number };

function assertNever(value: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(value)}`);
}

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'square':
      return shape.size ** 2;
    default:
      return assertNever(shape); // ✅ compiles: shape is never here
  }
}

// Later someone adds: | { kind: "triangle"; base: number; height: number }
// ❌ TS2345: Argument of type '{ kind: "triangle"; ... }' is not assignable to parameter of type 'never'.
// The error points at every switch that needs updating.
```

**4. ❌ Wrong / ✅ Right**
Without the `default` branch, TS may still catch it through the return type: `TS2366: Function lacks ending return statement and return type does not include 'undefined'.` The explicit `assertNever` is clearer, and it also throws at runtime if bad data sneaks through.

**5. Real-world use**
Redux/`useReducer` reducers (`default: return assertNever(action)`) and rendering by `status` in React.

**7. Interview tip**
_"How do you ensure a switch covers all union cases?"_ Add a `default` branch that assigns the value to `never`.

---

### 📌 Key Takeaways

- Object types describe shape; `interface` and `type` overlap, and `type` is needed for unions and computed types. Follow your team's convention.
- `readonly` is shallow and compile-time only; optional `?` adds `| undefined`.
- TS is structural. Extra properties are rejected only on fresh object literals, so data from the network is never checked by types alone.
- Model "one of several states" as a discriminated union with a literal discriminant.
- Narrow with `typeof`, `instanceof`, `in`, equality and truthiness, and write `is` guards for the rest. Prefer `=== undefined` over `!x` when `0` or `""` are valid.
- End every union `switch` with `assertNever` so adding a case breaks the build in the right places.

✅ Part 3 complete.
