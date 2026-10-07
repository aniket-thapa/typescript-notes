# Part 4: Generics

## 4.1 Generic functions 🔴 MUST KNOW

**1. What it is**
A generic function has a **type parameter** (usually `T`) that acts as a placeholder. TS fills it in from the arguments you pass, so the output type is linked to the input type.

**2. Why it exists**
Without generics, a reusable function must use `any` and lose the type, or you must write one copy per type.

**3. Code**

```ts
// ❌ any: input and output are disconnected
function wrapAny(value: any): { value: any } {
  return { value };
}

// ✅ generic: whatever goes in comes out typed
function wrap<T>(value: T): { value: T } {
  return { value };
}

const a = wrap(42); // { value: number }   (T inferred as number)
const b = wrap('hi'); // { value: string }
const c = wrap<boolean>(true); // explicit T, rarely needed

// Multiple type parameters
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}
const p = pair('age', 30); // [string, number]

// Arrow function version (.tsx needs the trailing comma)
const identity = <T>(value: T): T => value;
```

**4. ❌ Wrong / ✅ Right**

```ts
function first<T>(items: T[]): T {
  return items[0]; // ⚠️ lies when the array is empty
}
function firstOk<T>(items: T[]): T | undefined {
  return items[0]; // ✅ honest return type
}
```

**5. Real-world use**
`Array<T>`, `Promise<T>`, `Map<K, V>` and `useState<T>` are all generic types you already use. Your own helpers (`groupBy`, `pick`, `retry`) should be generic too.

**6. Common mistakes & errors**

- `TS2558: Expected 1 type arguments, but got 2.` You passed more type arguments than the function declares.
- Writing `T` and expecting it to exist at runtime. Generics are erased (Part 0). You cannot do `new T()` or `typeof T`.

**7. Interview tip**
_"What is a generic?"_ A type placeholder resolved at the call site, keeping type relationships without `any`.

---

## 4.2 Generic interfaces and type aliases 🔴 MUST KNOW

**1. What it is**
Interfaces and aliases can take type parameters too. You must supply them when you use the type (unless a default exists, see 4.5).

**3. Code**

```ts
interface Box<T> {
  value: T;
}
type Maybe<T> = T | null | undefined;
type Callback<T> = (data: T) => void;

const numberBox: Box<number> = { value: 1 };
const name: Maybe<string> = null;

const bad: Box = { value: 1 };
// ❌ TS2314: Generic type 'Box<T>' requires 1 type argument(s).
```

**5. Real-world use: a typed API response wrapper**

```ts
// Shared by every endpoint in your Express app
type ApiSuccess<T> = { success: true; data: T };
type ApiFailure = { success: false; error: string };
type ApiResponse<T> = ApiSuccess<T> | ApiFailure; // discriminated union (Part 3)

interface PaginatedData<T> {
  items: T[];
  page: number;
  total: number;
}

interface User {
  id: number;
  name: string;
}

// Backend helpers
function ok<T>(data: T): ApiSuccess<T> {
  return { success: true, data };
}
function fail(error: string): ApiFailure {
  return { success: false, error };
}

// Frontend usage
function showUsers(res: ApiResponse<PaginatedData<User>>) {
  if (res.success) {
    console.log(res.data.items[0]?.name); // narrowed: data exists
  } else {
    console.error(res.error); // narrowed: error exists
  }
}
```

One definition now types every endpoint: `ApiResponse<User>`, `ApiResponse<Order[]>`, and so on.

**6. Common mistake**
Leaving out the type argument on a generic type. If you want "any", write `Box<unknown>`, not bare `Box`.

---

## 4.3 Constraints with `extends` 🔴 MUST KNOW

**1. What it is**
A **constraint** limits which types `T` may be: `T extends Something`. Inside the function you can then safely use whatever `Something` guarantees.

**2. Why it exists**
An unconstrained `T` could be anything, so TS lets you do almost nothing with it.

**3. Code**

```ts
function getLength<T>(item: T): number {
  return item.length;
  // ❌ TS2339: Property 'length' does not exist on type 'T'.
}

function getLengthOk<T extends { length: number }>(item: T): T {
  console.log(item.length); // ✅ guaranteed by the constraint
  return item; // returns the ORIGINAL type, not just { length: number }
}
getLengthOk('hello'); // ✅ string has length
getLengthOk([1, 2, 3]); // ✅ array has length
getLengthOk(42);
// ❌ TS2345: Argument of type 'number' is not assignable to parameter of type '{ length: number; }'.

// Constraining to an interface
interface HasId {
  id: number;
}
function findById<T extends HasId>(items: T[], id: number): T | undefined {
  return items.find((item) => item.id === id);
}
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Loses the specific type: returns only { id: number }
function pickFirst(items: HasId[]): HasId | undefined {
  return items[0];
}
// ✅ Keeps the full type, e.g. User
function pickFirstOk<T extends HasId>(items: T[]): T | undefined {
  return items[0];
}
```

**5. Real-world use**
`extends` makes generic repositories possible (4.8), since they need `id` to exist.

**6. Common mistakes & errors**

- `TS2322: Type 'string' is not assignable to type 'T'. 'string' is assignable to the constraint of type 'T', but 'T' could be instantiated with a different subtype of constraint...` Example: `function f<T extends string>(): T { return "a"; }`. The caller might choose `T = "b"`. Return a value of type `T` that came from the input, or drop the generic.
- Constraint is a **minimum**, not an exact type. `T extends HasId` accepts objects with extra fields.

**7. Interview tip**
_"What does `T extends X` mean?"_ "T must be assignable to X", a requirement, not inheritance.

---

## 4.4 `keyof`, `typeof` and indexed access types 🔴 MUST KNOW

**1. What it is**
Three operators that build types from existing types:

| Operator                            | Result                                           |
| ----------------------------------- | ------------------------------------------------ |
| `keyof T`                           | Union of `T`'s property names                    |
| `typeof value` (in a type position) | The type of a runtime value                      |
| `T[K]`                              | The type of property `K` on `T` (indexed access) |

**2. Why it exists**
So you avoid copying types by hand. When the source changes, derived types update automatically.

**3. Code**

```ts
interface User {
  id: number;
  name: string;
  roles: string[];
}

type UserKeys = keyof User; // "id" | "name" | "roles"
type NameType = User['name']; // string
type Role = User['roles'][number]; // string (element type of the array)
type IdOrName = User['id' | 'name']; // string | number

// typeof: derive a type from a value
const defaultSettings = { theme: 'dark', pageSize: 20 };
type Settings = typeof defaultSettings; // { theme: string; pageSize: number }

// Combine with `as const` (Part 1)
const STATUSES = ['pending', 'paid', 'shipped'] as const;
type Status = (typeof STATUSES)[number]; // "pending" | "paid" | "shipped"

// The classic safe property getter
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const user: User = { id: 1, name: 'Asha', roles: ['admin'] };
const n = getProp(user, 'name'); // string
const r = getProp(user, 'roles'); // string[]
getProp(user, 'email');
// ❌ TS2345: Argument of type '"email"' is not assignable to parameter of type 'keyof User'.
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Type duplicated by hand; drifts out of sync
type Theme = 'light' | 'dark';
const themes = ['light', 'dark'];
// ✅ Single source of truth
const THEMES = ['light', 'dark'] as const;
type ThemeOk = (typeof THEMES)[number];
```

Note: `typeof` in a _type_ position is different from `typeof x === "string"` in a _value_ position (narrowing).

**5. Real-world use**
Sortable table columns (`sortBy: keyof User`), form field names, and typing `req.query.sort` against allowed model fields.

**6. Common mistakes & errors**

- `TS2749: 'config' refers to a value, but is being used as a type here. Did you mean 'typeof config'?` Add `typeof`.
- `keyof` on an object with a string index signature gives `string | number`, since JS coerces numeric keys.

**7. Interview tip**
_"Write `getProp` so it is type-safe."_ Use `K extends keyof T` and return `T[K]`. This is one of the most common generic interview questions.

---

## 4.5 Default type parameters 🟡 GOOD TO KNOW

**1. What it is**
A type parameter can have a fallback: `<T = string>`. It is used when the caller gives no type and TS cannot infer one.

**3. Code**

```ts
interface ApiResult<T = unknown> {
  status: number;
  data: T;
}

const raw: ApiResult = { status: 200, data: 'anything' }; // data: unknown
const typed: ApiResult<User[]> = { status: 200, data: [] }; // data: User[]

// Constraint and default together: constraint first, then `=`
interface Page<T extends object = Record<string, unknown>> {
  items: T[];
}
```

Rule: parameters with defaults must come **after** those without (`<A, B = string>`, not `<A = string, B>`). Otherwise: `TS2706: Required type parameters may not follow optional type parameters.`

**5. Real-world use**
React types use this heavily. `useState<S>`, `ComponentProps` and Redux Toolkit generics all have defaults, which is why you often see signatures with many type parameters that you never fill in.

---

## 4.6 Generic classes 🔴 MUST KNOW

**1. What it is**
A class takes type parameters after its name; every instance property and method can use them.

**3. Code**

```ts
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }
  pop(): T | undefined {
    return this.items.pop();
  }
  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }
}

const numbers = new Stack<number>();
numbers.push(1);
numbers.push('2');
// ❌ TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

const inferred = new Stack<string>(); // constructors can't infer T from nothing, so say it
```

**4. ❌ Wrong / ✅ Right**

```ts
class Registry<T> {
  static default: T;
  // ❌ TS2302: Static members cannot reference class type parameters.
  // Statics belong to the class, not to a specific instance type.
}
```

**5. Real-world use**
Service and repository base classes (4.8), event emitters, in-memory caches (`class Cache<K, V>`). Classes get full coverage in Part 6.

---

## 4.7 Generic components (preview) 🔴 MUST KNOW

**1. What it is**
A React component that is itself a generic function, so its props adapt to the data it receives.

**3. Code**

```tsx
import type { ReactNode } from 'react';

interface Column<T> {
  header: string;
  render: (row: T) => ReactNode;
}

interface TableProps<T> {
  rows: T[];
  columns: Column<T>[];
  getKey: (row: T) => string | number;
}

function Table<T>({ rows, columns, getKey }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((c) => (
            <th key={c.header}>{c.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {rows.map((row) => (
          <tr key={getKey(row)}>
            {columns.map((c) => (
              <td key={c.header}>{c.render(row)}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage: T is inferred as User from `rows`
interface User {
  id: number;
  name: string;
  email: string;
}
const users: User[] = [{ id: 1, name: 'Asha', email: 'asha@example.com' }];

<Table
  rows={users}
  getKey={(u) => u.id}
  columns={[
    { header: 'Name', render: (u) => u.name },
    { header: 'Email', render: (u) => u.email.toLowerCase() },
    { header: 'Age', render: (u) => u.age },
    // ❌ TS2339: Property 'age' does not exist on type 'User'.
  ]}
/>;
```

**4. ❌ Wrong / ✅ Right**
Typing the component with `React.FC<TableProps<T>>` does not work, because an arrow-function constant cannot introduce `T`. Write a **function declaration** (or generic arrow `<T,>(...) =>`).

**5. Real-world use**
Tables, dropdowns (`Select<Option>`), lists and data-fetching wrappers. Part 10 covers `forwardRef` with generics, which has extra limitations.

---

## 4.8 Generic repository / service pattern 🔴 MUST KNOW

**1. What it is**
A repository hides data access (Mongo, Prisma, memory) behind one interface. Making it generic means you write the contract **once** for `User`, `Product`, `Order`, and so on.

**2. Why it exists**
Services depend on the interface, not the database, which makes code reusable and easy to test with an in-memory version.

**3. Code**

```ts
import { randomUUID } from 'node:crypto';

interface Entity {
  id: string;
}

// Omit and Partial are built-in utility types: full coverage in Part 5.
interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, 'id'>): Promise<T>;
  update(id: string, data: Partial<Omit<T, 'id'>>): Promise<T | null>;
  delete(id: string): Promise<boolean>;
}

class InMemoryRepository<T extends Entity> implements Repository<T> {
  private items = new Map<string, T>();

  async findById(id: string): Promise<T | null> {
    return this.items.get(id) ?? null;
  }
  async findAll(): Promise<T[]> {
    return [...this.items.values()];
  }
  async create(data: Omit<T, 'id'>): Promise<T> {
    // Cast is needed: TS can't prove "Omit<T,'id'> + id" equals T for an unknown T.
    // This is a normal, accepted use of `as` in generic base classes.
    const entity = { ...data, id: randomUUID() } as T;
    this.items.set(entity.id, entity);
    return entity;
  }
  async update(id: string, data: Partial<Omit<T, 'id'>>): Promise<T | null> {
    const existing = this.items.get(id);
    if (!existing) return null;
    const updated = { ...existing, ...data };
    this.items.set(id, updated);
    return updated;
  }
  async delete(id: string): Promise<boolean> {
    return this.items.delete(id);
  }
}

// Concrete usage
interface User extends Entity {
  name: string;
  email: string;
}

const userRepo: Repository<User> = new InMemoryRepository<User>();

class UserService {
  constructor(private readonly repo: Repository<User>) {}

  async register(name: string, email: string): Promise<User> {
    return this.repo.create({ name, email }); // `id` not allowed here: it's Omit<User,"id">
  }
}
```

**5. Real-world use**
A `MongoRepository<T>` or `PrismaRepository<T>` implements the same interface. Controllers call `UserService`, and tests inject the in-memory one. Part 9 builds on this.

**6. Common mistakes**

- Putting business logic in the generic base class. Keep it to plain data access; put rules in specific services.
- Mongo documents use `_id`, not `id`. Your `Entity` constraint must match your real model, or you need a mapping layer.

**7. Interview tip**
_"How would you type a reusable data layer?"_ Describe `Repository<T extends Entity>` with `Omit<T, "id">` for creation.

---

## 4.9 Common mistakes: over-using generics 🔴 MUST KNOW

**Rule of thumb:** a type parameter should appear **at least twice** (relating two things, such as input and output). If it appears once, it is probably unnecessary.

```ts
// ❌ T used once: adds nothing
function logValue<T>(value: T): void {
  console.log(value);
}
function logValueOk(value: unknown): void {
  console.log(value);
} // ✅

// ❌ Generic with no connection to the result
function getUsers<T>(): Promise<T[]> {
  /* returns whatever the caller claims */
}
// The caller writes getUsers<Order>() and TS believes it. This is a hidden `as`.
// ✅ Return the real type
function getUsersOk(): Promise<User[]> {
  /* ... */
}

// ❌ Constraint missing or too wide
function merge<T, U>(a: T, b: U) {
  return { ...a, ...b };
}
// ❌ TS2698: Spread types may only be created from object types.
function mergeOk<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}
```

**Return-only generics are a trap.** `fetchJson<User>(url)` looks type-safe, but TS never checks that the server really sends a `User`. It is the same lie as `as User`. Use runtime validation (Part 8) for data from outside.

**Other pitfalls**

- Single-letter names get confusing past two or three parameters. Use descriptive names: `TItem`, `TResponse`, `TKey`. Both styles are common; follow your team.
- Deeply nested generics produce unreadable errors. If a signature needs five type parameters, step back and simplify.
- Prefer inference. Write `wrap(42)`, not `wrap<number>(42)`.

**7. Interview tip**
_"When should you avoid generics?"_ When there is no relationship between types to preserve. Use a concrete type or `unknown`.

---

### 📌 Key Takeaways

- A generic links types together (input to output) without `any`; TS infers `T` from your arguments.
- Use `T extends X` to require capabilities; the constraint is a minimum, and the caller's full type is preserved.
- `keyof T`, `T[K]` and `typeof value` derive types from existing ones, so there is one source of truth. `<T, K extends keyof T>` with `T[K]` is the classic pattern.
- Generic wrappers like `ApiResponse<T>` and `Repository<T extends Entity>` give your whole app consistent contracts.
- Generic React components must be function declarations (or `<T,>` arrows); `T` is inferred from props.
- If `T` appears only once, or only in the return type, skip the generic: it is either useless or a hidden unchecked assertion.

✅ Part 4 complete. Say "next" for Part 5
