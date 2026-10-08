# Part 5a: Utility Types

Part 5 is split in two to stay within the word limit. **5a** (this reply) covers the built-in utility types. **5b** covers mapped types, conditional types, `infer`, template literal types, key remapping, recursive types, and how to read complex types.

**What utility types are:** built-in generic types that transform an existing type into a new one. They are written with the same generics from Part 4, and you can read them as "functions from types to types."

**Why they exist:** so you do not copy-paste near-identical interfaces (`User`, `CreateUserInput`, `UpdateUserInput`). Change `User` once and the derived types update automatically.

All examples below use this type:

```ts
interface User {
  id: number;
  name: string;
  email: string;
  phone?: string;
  role: 'admin' | 'user';
}
```

---

## 5.1 `Partial<T>` and `Required<T>` 🔴 MUST KNOW

**1. What it is**
`Partial<T>` makes every property optional. `Required<T>` does the opposite: it makes every property required (and removes `undefined` from optional ones).

**3. Code**

```ts
type UpdateUserBody = Partial<User>;
// { id?: number; name?: string; email?: string; phone?: string; role?: "admin" | "user" }

function updateUser(id: number, changes: Partial<User>): void {}
updateUser(1, { name: 'Asha K' }); // ✅ only some fields
updateUser(1, { age: 30 });
// ❌ TS2353: Object literal may only specify known properties, and 'age' does not exist in type 'Partial<User>'.

type FullUser = Required<User>; // phone is now required: string
const u: FullUser = { id: 1, name: 'A', email: 'a@x.com', role: 'user' };
// ❌ TS2741: Property 'phone' is missing in type '{ ... }' but required in type 'Required<User>'.
```

**4. ❌ Wrong / ✅ Right**
`Partial` is **shallow**. It does not touch nested objects.

```ts
interface Settings {
  theme: { color: string; size: number };
}
const s: Partial<Settings> = { theme: { color: 'red' } };
// ❌ TS2741: Property 'size' is missing in type '{ color: string; }' but required in type '{ color: string; size: number; }'.
// ✅ Use a custom DeepPartial<T> (recursive type, Part 5b) if you need it.
```

Also, `Partial<User>` lets the caller change `id`. Usually you want `Partial<Omit<User, "id">>` (see 5.3).

**5. Real-world use**
PATCH request bodies, and "initial form values" in React (`useState<Partial<User>>({})`).

**6. Common mistake**
Using `Partial<T>` for a _create_ payload. Then required fields (like `email`) are no longer enforced.

---

## 5.2 `Readonly<T>` 🟡 GOOD TO KNOW

**1. What it is**
Makes every property `readonly`. Like `readonly` in Part 3, it is **shallow** and compile-time only.

**3. Code**

```ts
const config: Readonly<User> = {
  id: 1,
  name: 'A',
  email: 'a@x.com',
  role: 'user',
};
config.name = 'B';
// ❌ TS2540: Cannot assign to 'name' because it is a read-only property.
```

**5. Real-world use**
Props and state in React (never mutate them) and frozen config objects. Use `Readonly<Props>` or `readonly T[]` for arrays.

**6. Common mistake**
Expecting runtime protection. For that you need `Object.freeze()`. `Readonly` only affects the type checker.

---

## 5.3 `Pick<T, K>` and `Omit<T, K>` 🔴 MUST KNOW

**1. What it is**

- `Pick<T, K>`: keep **only** the listed keys.
- `Omit<T, K>`: keep everything **except** the listed keys.

**3. Code**

```ts
type UserPreview = Pick<User, 'id' | 'name'>;
// { id: number; name: string }

type CreateUserInput = Omit<User, 'id'>;
// everything except id (the database generates it)

type UpdateUserInput = Partial<Omit<User, 'id'>>;
// common combination: no id, all optional

type PublicUser = Omit<User, 'email' | 'phone'>;

Pick<User, 'age'>;
// ❌ TS2344: Type '"age"' does not satisfy the constraint 'keyof User'.
```

**4. ❌ Wrong / ✅ Right**
`Omit` does **not** check that the key exists. A typo passes silently.

```ts
type Bad = Omit<User, 'emial'>; // ❌ compiles with no error, and email is NOT removed!

// ✅ A strict version that catches typos
type StrictOmit<T, K extends keyof T> = Omit<T, K>;
type Good = StrictOmit<User, 'emial'>;
// ❌ TS2344: Type '"emial"' does not satisfy the constraint 'keyof User'.
```

This is deliberate in the standard library (for flexibility with unions), but it surprises many developers.

Another trap: `Omit` and `Pick` do **not distribute over unions**. On a union they collapse it to the common keys only. For unions, map over each member yourself with a distributive conditional type (Part 5b).

**5. Real-world use**
The standard "derive input types from the model" pattern:

```ts
// Service layer
async function createUser(input: Omit<User, 'id' | 'role'>): Promise<User> {
  /* ... */
}

// Never send the password hash to the client
type SafeUser = Omit<UserDocument, 'passwordHash'>;
```

**7. Interview tip**
_"Difference between `Pick` and `Omit`?"_ `Pick` is an allow-list, `Omit` is a deny-list. Mention that `Omit` doesn't validate keys.

---

## 5.4 `Record<K, T>` 🔴 MUST KNOW

**1. What it is**
An object type whose keys are `K` and every value is `T`.

**2. Why it exists**
It is the clean replacement for index signatures when you know the key set, and it forces you to cover **every** key.

**3. Code**

```ts
type Role = 'admin' | 'user' | 'guest';

const permissions: Record<Role, string[]> = {
  admin: ['read', 'write', 'delete'],
  user: ['read', 'write'],
  guest: ['read'],
};

const incomplete: Record<Role, string[]> = { admin: [], user: [] };
// ❌ TS2741: Property 'guest' is missing in type '{ admin: never[]; user: never[]; }' but required in type 'Record<Role, string[]>'.

// Open-ended keys
const cache: Record<string, number> = {}; // same as { [key: string]: number }
```

**4. ❌ Wrong / ✅ Right**

```ts
const lookup: Record<string, User> = {};
const u = lookup['x']; // type: User, but may be undefined at runtime ⚠️
// ✅ With noUncheckedIndexedAccess on, it is User | undefined. Or use Map<string, User>.
```

If a key may be absent, use `Partial<Record<Role, string>>`.

**5. Real-world use**
Status-to-label maps (`Record<OrderStatus, string>`), HTTP status to message maps, and route handler tables. Add a new union member and TS demands you update every `Record`.

---

## 5.5 `Exclude`, `Extract`, `NonNullable` 🟡 GOOD TO KNOW

**1. What it is**
These work on **unions**:

- `Exclude<T, U>`: remove members of `T` that are assignable to `U`.
- `Extract<T, U>`: keep only members of `T` that are assignable to `U`.
- `NonNullable<T>`: remove `null` and `undefined`.

**3. Code**

```ts
type Status = 'pending' | 'paid' | 'shipped' | 'cancelled';

type ActiveStatus = Exclude<Status, 'cancelled'>; // "pending" | "paid" | "shipped"
type FinalStatus = Extract<Status, 'shipped' | 'cancelled'>; // "shipped" | "cancelled"

type MaybeName = string | null | undefined;
type Name = NonNullable<MaybeName>; // string

// Extract with discriminated unions: pick one action by its `type`
type Action =
  | { type: 'add'; payload: User }
  | { type: 'remove'; id: number }
  | { type: 'clear' };

type AddAction = Extract<Action, { type: 'add' }>; // { type: "add"; payload: User }
```

**4. ❌ Wrong / ✅ Right**
Do not confuse `Exclude` with `Omit`. `Exclude` removes **members of a union**; `Omit` removes **keys of an object**.

```ts
type A = Exclude<User, 'email'>; // ❌ User isn't a union: nothing happens
type B = Omit<User, 'email'>; // ✅
```

**5. Real-world use**
`NonNullable` cleans up after a `.filter` or when a lookup is known to succeed. `Extract` gives you the exact action type for a reducer helper.

---

## 5.6 `ReturnType`, `Parameters`, `ConstructorParameters`, `InstanceType` 🔴 MUST KNOW (first two) / 🟡 (last two)

**1. What it is**
They extract types **from a function or class** so you never have to rewrite them.

| Utility                    | Gives you                                      |
| -------------------------- | ---------------------------------------------- |
| `ReturnType<F>`            | What function `F` returns                      |
| `Parameters<F>`            | Tuple of `F`'s parameter types                 |
| `ConstructorParameters<C>` | Tuple of a class constructor's parameter types |
| `InstanceType<C>`          | The type of objects made by `new C()`          |

**3. Code**

```ts
function createToken(userId: number, expiresInMin: number) {
  return {
    token: 'abc',
    userId,
    expiresAt: Date.now() + expiresInMin * 60_000,
  };
}

type Token = ReturnType<typeof createToken>;
// { token: string; userId: number; expiresAt: number }
type TokenArgs = Parameters<typeof createToken>; // [userId: number, expiresInMin: number]
type FirstArg = Parameters<typeof createToken>[0]; // number

class Mailer {
  constructor(
    public host: string,
    public port: number,
  ) {}
}
type MailerArgs = ConstructorParameters<typeof Mailer>; // [host: string, port: number]
type MailerInstance = InstanceType<typeof Mailer>; // Mailer
```

**4. ❌ Wrong / ✅ Right**
You must write `typeof` for a value (a function or class variable):

```ts
type Bad = ReturnType<createToken>;
// ❌ TS2749: 'createToken' refers to a value, but is being used as a type here. Did you mean 'typeof createToken'?
type Good = ReturnType<typeof createToken>; // ✅

type NotFn = ReturnType<string>;
// ❌ TS2344: Type 'string' does not satisfy the constraint '(...args: any) => any'.
```

**5. Real-world use**

- Typing a wrapper around a library function you don't control: `Parameters<typeof jwt.sign>`.
- Redux Toolkit: `type RootState = ReturnType<typeof store.getState>` (Part 10).
- Typing a function's result without writing an interface: `type Config = ReturnType<typeof loadConfig>`.

**7. Interview tip**
_"How do you get a function's return type without redeclaring it?"_ `ReturnType<typeof fn>`. Remember the `typeof`.

---

## 5.7 `Awaited<T>` 🔴 MUST KNOW

**1. What it is**
Unwraps a `Promise` (recursively) to get the value type it resolves to.

**2. Why it exists**
`ReturnType` of an `async` function is `Promise<X>`, not `X`. `Awaited` gets you `X`. It also models what `await` and `Promise.all` do.

**3. Code**

```ts
async function fetchUser(id: number): Promise<User> {
  /* ... */ throw new Error('todo');
}

type R1 = ReturnType<typeof fetchUser>; // Promise<User>
type R2 = Awaited<ReturnType<typeof fetchUser>>; // User   ✅ the combination you will use

type A = Awaited<Promise<Promise<string>>>; // string (nested promises flattened)
type B = Awaited<string>; // string (non-promises pass through)
```

**5. Real-world use**
Deriving the shape of a Prisma query result without writing it by hand:

```ts
async function getOrders() {
  return prisma.order.findMany({ include: { items: true } });
}
type OrderWithItems = Awaited<ReturnType<typeof getOrders>>[number];
```

(Prisma is covered in Part 9; the pattern is the point here.)

**6. Common mistake**
Forgetting `Awaited` and then wondering why `user.name` fails on a `Promise<User>`: `TS2339: Property 'name' does not exist on type 'Promise<User>'.`

---

## 5.8 `NoInfer<T>` 🟡 GOOD TO KNOW (TypeScript 5.4+)

**1. What it is**
Tells TS: "do not use this position to _infer_ `T`; take `T` from elsewhere."

**2. Why it exists**
In `function f<T>(list: T[], fallback: T)`, TS infers `T` from **both** arguments and may quietly widen it, so a wrong fallback is accepted.

**3. Code**

```ts
// ❌ Without NoInfer: T widens to include "blue"
function pickColor<C extends string>(colors: C[], fallback: C): C {
  return colors[0] ?? fallback;
}
pickColor(['red', 'green'], 'blue'); // compiles! C becomes "red" | "green" | "blue"

// ✅ With NoInfer: only `colors` decides C
function pickColorSafe<C extends string>(colors: C[], fallback: NoInfer<C>): C {
  return colors[0] ?? fallback;
}
pickColorSafe(['red', 'green'], 'blue');
// ❌ TS2345: Argument of type '"blue"' is not assignable to parameter of type '"red" | "green"'.
```

**5. Real-world use**
Generic components and helpers with a "default value" or "initial state" argument, such as a typed `useLocalStorage<T>(key, defaultValue)`.

**Version note:** needs TypeScript 5.4 or newer. Older code achieves the same with tricks like `T & {}`, which you may see in legacy codebases.

---

## 5.9 Quick reference and combining utilities 🔴 MUST KNOW

| Need                             | Use                         |
| -------------------------------- | --------------------------- |
| All fields optional (PATCH body) | `Partial<T>`                |
| All fields required              | `Required<T>`               |
| Lock fields                      | `Readonly<T>`               |
| Keep some keys / drop some keys  | `Pick<T, K>` / `Omit<T, K>` |
| Object keyed by a union          | `Record<K, V>`              |
| Remove / keep union members      | `Exclude` / `Extract`       |
| Strip `null` and `undefined`     | `NonNullable<T>`            |
| Function result / arguments      | `ReturnType` / `Parameters` |
| Resolved value of a promise      | `Awaited<T>`                |

Utilities combine by nesting, reading **inside out**:

```ts
// Express: input for PATCH /users/:id
type PatchUserBody = Partial<Omit<User, 'id' | 'role'>>;
// 1) Omit removes id and role  2) Partial makes the rest optional

// Result type of a service method, safely
type LoginResult = Awaited<ReturnType<typeof authService.login>>;
```

**Common mistake:** over-nesting (five utilities deep) makes hover tooltips unreadable. Name the intermediate types:

```ts
type UserWithoutId = Omit<User, 'id'>;
type PatchUserBody2 = Partial<UserWithoutId>;
```

**Interview tip:** _"How do you create a type for updating a user where `id` can't change?"_ `Partial<Omit<User, "id">>`.

---

### 📌 Key Takeaways

- Utility types derive new types from existing ones, so one source of truth feeds all your input and output types.
- `Partial<Omit<T, "id">>` for updates and `Omit<T, "id">` for creates are the patterns you will write weekly.
- `Omit` does not validate keys, so typos pass silently. `Pick` and `Record` do enforce them.
- `Partial` and `Readonly` are shallow. Nested objects need a custom deep version (Part 5b).
- `ReturnType`, `Parameters` and `Awaited` need `typeof` for values, and `Awaited<ReturnType<typeof asyncFn>>` gives the resolved result.
- `NoInfer` (TS 5.4+) stops a parameter from widening the inferred generic.

✅ Part 5a complete.

---

# Part 5b: Mapped, Conditional & Advanced Types

This continues Part 5a. These are the tools behind the utility types you just learned. You will write them less often than `Partial` or `Omit`, but you must be able to **read** them in old codebases and libraries.

---

## 5.10 Mapped types 🔴 MUST KNOW

**1. What it is**
A mapped type builds a new object type by looping over a list of keys: `{ [K in Keys]: ValueType }`. Read it as "for each key `K` in `Keys`, make a property."

**2. Why it exists**
It is how `Partial`, `Readonly`, `Pick` and `Record` are built. Knowing it lets you create your own and read theirs.

**3. Code**

```ts
interface User {
  id: number;
  name: string;
  email: string;
}

// Re-implementing built-ins
type MyPartial<T> = { [K in keyof T]?: T[K] };
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type MyPick<T, Keys extends keyof T> = { [K in Keys]: T[K] };

// Your own: every property becomes nullable
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Form errors: one optional message per field
type FormErrors<T> = { [K in keyof T]?: string };
const errors: FormErrors<User> = { email: 'Invalid email' };

// Modifiers: + adds, - removes
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
type AllRequired<T> = { [K in keyof T]-?: T[K] }; // this is how Required<T> works
```

**Homomorphic mapped types:** a mapped type over `keyof T` keeps the original `readonly` and `?` modifiers unless you change them. It also maps arrays and tuples element by element. This is why `Partial<string[]>` still gives an array.

**4. ❌ Wrong / ✅ Right**

```ts
type Bad<T> = { [K in keyof T]: T[K]; extra: string };
// ❌ TS7061: A mapped type may not declare properties or methods.
type Good<T> = { [K in keyof T]: T[K] } & { extra: string }; // ✅ use an intersection
```

**5. Real-world use**
`FormErrors<T>` and `TouchedFields<T>` for form state, `Nullable<T>` for database rows, and `Record`-style lookup tables.

**7. Interview tip**
_"Implement `Partial<T>` yourself."_ `{ [K in keyof T]?: T[K] }`. Very common.

---

## 5.11 Conditional types 🔴 MUST KNOW

**1. What it is**
A type-level `if`: `T extends U ? X : Y`. If `T` is assignable to `U` you get `X`, otherwise `Y`.

**2. Why it exists**
So a type can change depending on another type. `Exclude`, `Extract`, `NonNullable` and `ReturnType` are all conditional types.

**3. Code**

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<'hi'>; // true
type B = IsString<42>; // false

// Re-implementing built-ins
type MyExclude<T, U> = T extends U ? never : T;
type MyNonNullable<T> = T extends null | undefined ? never : T;
```

**Distribution (the surprising part):** when `T` is a **bare type parameter** and you pass a union, TS applies the condition to **each member** and unions the results.

```ts
type ToArray<T> = T extends unknown ? T[] : never;
type R = ToArray<string | number>; // string[] | number[]   (not (string | number)[])

// To turn distribution off, wrap both sides in tuples
type ToArrayWhole<T> = [T] extends [unknown] ? T[] : never;
type R2 = ToArrayWhole<string | number>; // (string | number)[]
```

`never` is an empty union, so `ToArray<never>` is `never`, not `never[]`. This surprises people.

**Fixing `Omit` on unions** (promised in 5a):

```ts
type Circle = { kind: 'circle'; radius: number; id: string };
type Square = { kind: 'square'; size: number; id: string };
type Shape = Circle | Square;

type Plain = Omit<Shape, 'id'>; // ❌ collapses to { kind: "circle" | "square" }, loses radius and size
type DistributiveOmit<T, K extends PropertyKey> = T extends unknown
  ? Omit<T, K>
  : never;
type Good = DistributiveOmit<Shape, 'id'>; // ✅ { kind: "circle"; radius: number } | { kind: "square"; size: number }
```

**4. ❌ Wrong / ✅ Right**
Returning a value from a function whose return type is conditional:

```ts
function parse<T extends string | number>(
  x: T,
): T extends string ? number : string {
  return typeof x === 'string' ? Number(x) : String(x);
  // ❌ TS2322: Type 'number | string' is not assignable to type 'T extends string ? number : string'.
}
```

TS cannot resolve a conditional type while `T` is still unknown. Fix: use **overloads** (Part 2) for the public signature, or cast inside the body (`as ...`) and accept that it is an unchecked promise.

**5. Real-world use**
Type-level helpers in libraries (`ReturnType`, Redux Toolkit, React's `ComponentProps`). In app code: `type Unwrap<T> = T extends Array<infer U> ? U : T`.

**7. Interview tip**
_"What is a distributive conditional type?"_ A conditional on a naked type parameter splits over each member of a union.

---

## 5.12 `infer` 🔴 MUST KNOW (to read) / 🟡 (to write)

**1. What it is**
Inside the `extends` part of a conditional type, `infer U` means "figure out this part of the type and name it `U`." It is pattern matching for types.

**3. Code**

```ts
type MyReturnType<F> = F extends (...args: any[]) => infer R ? R : never;
type ElementOf<T> = T extends (infer E)[] ? E : never;
type Unpromise<T> = T extends Promise<infer V> ? V : T;
type FirstArg<F> = F extends (first: infer A, ...rest: any[]) => any
  ? A
  : never;

type X1 = MyReturnType<() => string>; // string
type X2 = ElementOf<User[]>; // User
type X3 = Unpromise<Promise<number>>; // number
type X4 = FirstArg<(id: number, n: string) => void>; // number

// Constrain what is inferred (TS 4.7+)
type FirstString<T> = T extends [infer S extends string, ...unknown[]]
  ? S
  : never;
```

Read `ElementOf` aloud: "if `T` is an array of _something_, call that something `E` and give me `E`; otherwise `never`."

**4. ❌ Wrong / ✅ Right**

```ts
type Bad<T> = infer U;
// ❌ TS1338: 'infer' declarations are only permitted in the 'extends' clause of a conditional type.
type Good<T> = T extends (infer U)[] ? U : never; // ✅
```

**5. Real-world use**
Extracting props from a component, event payloads from a handler map, and the `ReturnType`/`Parameters` family. Most `infer` you meet is in library types and old utility files.

**6. Common mistake**
Using `any[]` in the function pattern is normal here, because the pattern must match every function. It is one of the few acceptable `any` uses (Part 13).

---

## 5.13 Template literal types 🟡 GOOD TO KNOW

**1. What it is**
String types built like JS template strings: `` `on${string}` ``. Unions inside multiply out into every combination.

**3. Code**

```ts
type EventName = 'click' | 'hover';
type Handler = `on${Capitalize<EventName>}`; // "onClick" | "onHover"

type Size = 'sm' | 'lg';
type Color = 'red' | 'blue';
type ClassName = `${Color}-${Size}`; // "red-sm" | "red-lg" | "blue-sm" | "blue-lg"

const h: Handler = 'onScroll';
// ❌ TS2322: Type '"onScroll"' is not assignable to type 'Handler'.

// Built-in helpers: Uppercase, Lowercase, Capitalize, Uncapitalize
type Loud = Uppercase<'get'>; // "GET"

// With infer: extract route params from a path string
type RouteParams<S extends string> =
  S extends `${string}:${infer P}/${infer Rest}`
    ? P | RouteParams<Rest>
    : S extends `${string}:${infer P}`
      ? P
      : never;

type Params = RouteParams<'/users/:id/posts/:postId'>; // "id" | "postId"
```

**4. ❌ Wrong / ✅ Right**
Big unions explode: three unions of 10 members each gives 1,000 strings. TS refuses past about 100,000: `TS2590: Expression produces a union type that is too complex to represent.` Use `` `${string}-${string}` `` patterns for open-ended text.

**5. Real-world use**
Typed route helpers (`params` inferred from `"/users/:id"`, as in some routers), CSS class props, event name maps, and `` `${Table}Id` `` keys.

---

## 5.14 Key remapping with `as` 🟡 GOOD TO KNOW

**1. What it is**
Inside a mapped type, `as` renames each key (or removes it by mapping it to `never`).

**3. Code**

```ts
interface User {
  id: number;
  name: string;
}

// Rename: generate getters
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<User>; // { getId: () => number; getName: () => string }

// Filter: drop keys whose value is a function
type DataOnly<T> = {
  [K in keyof T as T[K] extends (...args: any[]) => any ? never : K]: T[K];
};
class Account {
  id = 1;
  balance = 10;
  deposit(n: number) {}
}
type AccountData = DataOnly<Account>; // { id: number; balance: number }

// Map a union of objects into a handler table, keyed by the discriminant
type Action = { type: 'add'; item: string } | { type: 'clear' };
type Handlers = { [A in Action as A['type']]: (action: A) => void };
// { add: (action: { type: "add"; item: string }) => void; clear: (action: { type: "clear" }) => void }
```

**4. ❌ Wrong / ✅ Right**

```ts
type Bad<T> = { [K in keyof T as `get${Capitalize<K>}`]: T[K] };
// ❌ TS2344: Type 'K' does not satisfy the constraint 'string'.
type Good<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: T[K] }; // ✅
```

`keyof T` may include `number` and `symbol`, so `string & K` narrows it to the string keys.

**5. Real-world use**
Generating `onChange`-style props per field, stripping methods before sending a model to the client, and building typed event-emitter maps.

---

## 5.15 Recursive types 🟡 GOOD TO KNOW

**1. What it is**
A type that refers to itself. Needed for nested data of unknown depth: JSON, trees, nested config.

**3. Code**

```ts
// JSON value
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };

// Tree
interface TreeNode<T> {
  value: T;
  children: TreeNode<T>[];
}

// DeepPartial (the fix promised in 5a)
type DeepPartial<T> = T extends (...args: any[]) => any | Date
  ? T
  : T extends object
    ? { [K in keyof T]?: DeepPartial<T[K]> }
    : T;

interface Settings {
  theme: { color: string; size: number };
  tags: string[];
}
const s: DeepPartial<Settings> = { theme: { color: 'red' } }; // ✅ size may be missing

type DeepReadonly<T> = T extends (...args: any[]) => any
  ? T
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```

Functions and `Date` are returned unchanged, otherwise TS would map over their internal keys and produce nonsense.

**4. ❌ Wrong / ✅ Right**
Recursion that never ends: `TS2589: Type instantiation is excessively deep and possibly infinite.` Simplify, add a base case, or stop recursing at primitives. TS also caps recursion depth, so extreme types are slow.

**5. Real-world use**
`DeepPartial` for test fixtures and config overrides, `Json` for message payloads and `JSON.parse` results, trees for comment threads and menus.

---

## 5.16 How to read a complex type step by step 🔴 MUST KNOW

You will meet types like this in legacy code:

```ts
type Cleaned<T> = {
  [K in keyof T as T[K] extends (...args: any[]) => any
    ? never
    : K]: T[K] extends object ? Cleaned<T[K]> : T[K];
};
```

**Step-by-step method**

1. **Find the outer shape.** Is it a mapped type `{ [K in ...] }`, a conditional `A extends B ? C : D`, or a union? Here: a mapped type.
2. **Read the loop header.** `[K in keyof T as ...]` means "for each key of `T`, with a rename/filter."
3. **Read the `as` clause.** `T[K] extends Function ? never : K` means "if the value is a function, drop the key; else keep it."
4. **Read the value part.** After the `:`, "if the value is an object, apply `Cleaned` again (recursion); else keep it."
5. **Say it in one sentence.** "Remove all methods, at every nesting level."
6. **Test with a tiny example** (below).

**Tools**

```ts
// 1) Make hover tooltips readable (the "Prettify" trick)
type Prettify<T> = { [K in keyof T]: T[K] } & {};

// 2) Try the type on a small sample and hover over the result
interface Sample {
  id: number;
  save(): void;
  profile: { name: string; touch(): void };
}
type Result = Prettify<Cleaned<Sample>>; // { id: number; profile: { name: string } }

// 3) Break a big type into named pieces
type IsMethod<V> = V extends (...args: any[]) => any ? true : false;

// 4) Force an error to see the type: assign it to something wrong
const probe: Result = 123; // ❌ error message prints the full expanded type
```

**Other tips:** hover and use "Go to Type Definition" in VS Code; check `typeof x` in the editor; search for each unfamiliar helper's definition (Ctrl/Cmd+click); keep a scratch file where you paste a type with sample data.

**Common mistake:** trying to understand the whole expression at once. Work from the outside in, one layer at a time.

**Interview tip:** _"How would you debug a complex type?"_ Break into named aliases, test with small samples, hover, and use a scratch file.

---

### 📌 Key Takeaways

- A mapped type loops over keys (`[K in keyof T]`); `?`, `readonly`, `-?` and `-readonly` change modifiers.
- A conditional type (`T extends U ? X : Y`) distributes over unions when `T` is a naked type parameter; wrap in `[T]` to stop it.
- `infer` names a piece of a matched type, and it works only in the `extends` clause of a conditional.
- Template literal types build string unions; `as` in a mapped type renames or filters keys (map to `never` to drop).
- Recursive types model nested data (`Json`, `DeepPartial`); give them a base case and handle functions and `Date`.
- To read a complex type, go outside-in, say each layer in plain words, and test it on a small sample.

✅ Part 5b complete.
