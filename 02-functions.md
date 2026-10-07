# Part 2: Functions

## 2.1 Typing parameters and return values 🔴 MUST KNOW

**1. What it is**
You write a type after each parameter and after the closing `)` for the return value.

**2. Why it exists**
Parameters are the entry points of your code. TS cannot guess what callers will pass, so with `noImplicitAny` it forces you to say.

**3. Code**

```ts
function calculateTotal(price: number, quantity: number): number {
  return price * quantity;
}

calculateTotal(100, 2); // ✅
calculateTotal(100); // ❌ TS2554: Expected 2 arguments, but got 1.
calculateTotal(100, '2'); // ❌ TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

// Return type is inferred if you leave it out:
function getLabel(active: boolean) {
  return active ? 'On' : 'Off'; // inferred: "On" | "Off"
}
```

**4. ❌ Wrong / ✅ Right**

```ts
function format(value) {
  return value;
} // ❌ TS7006: Parameter 'value' implicitly has an 'any' type.
function formatOk(value: string): string {
  return value;
} // ✅
```

**5. Real-world use**
Annotate return types on exported and public functions (services, utilities). It acts as a contract. If you accidentally return the wrong shape, the error shows inside the function, not far away in the caller.

**6. Common mistakes & errors**

- `TS2355: A function whose declared type is neither 'undefined', 'void', nor 'any' must return a value.` One code path forgot a `return`.
- `TS7030: Not all code paths return a value.` appears only if `noImplicitReturns` is on.

---

## 2.2 Optional, default and rest parameters 🔴 MUST KNOW

**1. What it is**

- **Optional** (`name?: string`): the caller may skip it. Inside, the type is `string | undefined`.
- **Default** (`name = "Guest"`): used when the argument is missing or `undefined`.
- **Rest** (`...nums: number[]`): collects extra arguments into an array.

**3. Code**

```ts
function greet(name: string, greeting?: string): string {
  return `${greeting ?? 'Hello'}, ${name}`;
}

function paginate(page = 1, limit = 10) {
  // types inferred as number
  return { page, limit };
}
paginate(); // { page: 1, limit: 10 }
paginate(undefined, 5); // page falls back to 1

function sum(...nums: number[]): number {
  return nums.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3);

// Rest with a tuple: fixed leading args then the rest
function log(level: 'info' | 'error', ...parts: string[]) {}
```

**4. ❌ Wrong / ✅ Right**

```ts
function f(a?: number, b: number) {}
// ❌ TS1016: A required parameter cannot follow an optional parameter.
function g(b: number, a?: number) {} // ✅ optional params go last

function h(...nums: number) {}
// ❌ TS2370: A rest parameter must be of an array type.
```

**5. Real-world use**
Pagination helpers, logger functions (`logger.info(msg, ...meta)`), and config builders with defaults.

**6. Common mistakes**
Using `?` when you actually want a default. `limit?: number` forces you to handle `undefined` yourself; `limit = 10` gives you a plain `number` inside the function.

---

## 2.3 Function type expressions and call signatures 🔴 MUST KNOW

**1. What it is**
A **function type expression** describes the _shape of a function_ so you can reuse it: `(a: number) => string`. A **call signature** does the same inside an object type, useful when a function also has properties.

**2. Why it exists**
You need to type variables, parameters and properties that hold functions.

**3. Code**

```ts
// Function type expression
type Comparator = (a: number, b: number) => number;
const ascending: Comparator = (a, b) => a - b; // parameter types come from Comparator

// Call signature: function with extra properties (🟡 less common)
type Counter = {
  (): number; // callable
  reset: () => void; // property
};

// Constructor signature (🟢 rare): something you call with `new`
type UserFactory = new (name: string) => { name: string };
```

| Syntax          | Use when                                                   |
| --------------- | ---------------------------------------------------------- |
| `(a: A) => R`   | Almost always                                              |
| `{ (a: A): R }` | The function also has properties, or inside an `interface` |

**5. Real-world use**
Typing a handler map: `Record<string, (req: Request) => Response>`, or an event callback prop: `onSave: (user: User) => void`.

**6. Common mistake**
In a type alias the arrow uses `=>`. In a call signature (inside `{}`) it uses `:`. Mixing them gives syntax errors like `TS1005: ';' expected.`

---

## 2.4 Callbacks 🔴 MUST KNOW

**1. What it is**
A callback is a function you pass to another function. TS gives callback parameters their types automatically (**contextual typing**) when the receiving function declares the callback's type.

**3. Code**

```ts
function fetchData(
  onSuccess: (data: string) => void,
  onError: (err: Error) => void,
) {
  onSuccess('ok');
}

fetchData(
  (data) => console.log(data.toUpperCase()), // `data` is string, no annotation needed
  (err) => console.error(err.message),
);

// Built-ins are typed the same way
const ids = [1, 2, 3].map((n) => n.toString()); // string[]

// A callback may accept FEWER parameters than offered, never more
[1, 2].forEach((n) => console.log(n)); // ✅ ignores index and array
fetchData(
  (a, b) => {},
  () => {},
);
// ❌ TS2345: ... Target signature provides too few arguments. Expected 2 or more, but got 1.
```

**4. ❌ Wrong / ✅ Right**

```ts
const handler = (e) => {}; // ❌ TS7006: no context, so 'e' is implicit any
const handler2 = (e: MouseEvent) => {}; // ✅ annotate when defined outside the call site
```

Contextual typing only works when the function is written _where its type is known_. Extract a callback into its own variable and you must annotate it.

**5. Real-world use**
`array.filter((u) => u.isActive)`, Express route handlers, `setTimeout`, and React `onClick` handlers.

**7. Interview tip**
_"What is contextual typing?"_ TS infers a function's parameter types from where it is used.

---

## 2.5 Arrow functions 🔴 MUST KNOW

**1. What it is**
Same typing rules as normal functions. Arrow functions also keep the `this` of the surrounding code (no own `this`).

**3. Code**

```ts
const multiply = (a: number, b: number): number => a * b;

// Returning an object literal needs parentheses
const makeUser = (name: string): { name: string } => ({ name });

// A typed variable can carry the function type instead
const divide: (a: number, b: number) => number = (a, b) => a / b;

// Generic arrow in a .tsx file: a trailing comma stops JSX confusion
const identity = <T>(value: T): T => value;
```

**6. Common mistakes & errors**
In `.tsx` files, `<T>(x: T) => x` is parsed as a JSX tag. Use `<T,>` or `<T extends unknown>`. If you forget, you get `TS1005` or `TS17008: JSX element 'T' has no corresponding closing tag.`

---

## 2.6 `void` vs `undefined` 🟡 GOOD TO KNOW

**1. What it is**
`void` means "the caller should not use the return value." `undefined` means "returns exactly `undefined`."

**3. Code**

```ts
function save(): void {
  console.log('saved');
}
function bad(): void {
  return 5; // ❌ TS2322: Type 'number' is not assignable to type 'void'.
}

// Special rule: a *function type* returning void accepts functions that return something
const results: number[] = [];
[1, 2, 3].forEach((n) => results.push(n)); // push returns number, but forEach's callback is typed void ✅

function run(cb: () => void) {
  cb();
}
run(() => 42); // ✅ allowed, return value is ignored
```

**4. Why this rule exists**
It lets you pass `results.push` style functions to callbacks without wrapping them. The value is ignored, not forbidden.

**Version note:** since TS 5.1, a function declared to return `undefined` no longer needs an explicit `return`. Older versions raised `TS2355`.

**7. Interview tip**
_"Difference between `void` and `undefined`?"_ `void` says "ignore the return"; a callback typed to return `void` may still return a value, but a function _declared_ `: void` cannot return one.

---

## 2.7 Higher-order functions 🔴 MUST KNOW

**1. What it is**
A function that **takes** a function, **returns** a function, or both.

**3. Code**

```ts
// Takes a function
function applyTwice(fn: (n: number) => number, value: number): number {
  return fn(fn(value));
}

// Returns a function (a "factory")
function createMultiplier(factor: number): (n: number) => number {
  return (n) => n * factor;
}
const triple = createMultiplier(3);
triple(5); // 15

// Wraps another function (generic, so it keeps the original types)
function withLogging<Args extends unknown[], R>(
  fn: (...args: Args) => R,
): (...args: Args) => R {
  return (...args) => {
    console.log('calling with', args);
    return fn(...args);
  };
}
const loggedAdd = withLogging((a: number, b: number) => a + b);
loggedAdd(1, 2); // ✅ typed (a: number, b: number) => number
loggedAdd(1, '2'); // ❌ TS2345
```

**5. Real-world use**
Express middleware factories (`requireRole("admin")` returns a middleware), the `asyncHandler` wrapper (2.9), React higher-order components, and `useCallback`.

**6. Common mistake**
Typing the wrapper as `(...args: any[]) => any`. It compiles but erases all type information. Use a generic as above.

---

## 2.8 Generics preview 🔴 MUST KNOW

**1. What it is**
A **generic** is a type placeholder (`T`) filled in when the function is called. Full coverage is in Part 4. Here you only need to recognize and use simple ones.

**2. Why it exists**
Without it, a reusable function must use `any` and lose type information.

**3. Code**

```ts
// ❌ any loses the type
function firstAny(items: any[]): any {
  return items[0];
}

// ✅ generic keeps it
function first<T>(items: T[]): T | undefined {
  return items[0];
}

const n = first([1, 2, 3]); // number | undefined (T inferred as number)
const s = first(['a', 'b']); // string | undefined
const u = first<string>([]); // you can pass T explicitly, rarely needed
```

**5. Real-world use**
`Array<T>`, `Promise<T>`, `useState<T>`, `ApiResponse<T>`. You already use generics every time you write `Promise<User>`.

**7. Interview tip**
_"Why use generics instead of `any`?"_ Same flexibility, but the relationship between input and output types is preserved.

---

## 2.9 Typing async functions 🔴 MUST KNOW

**1. What it is**
An `async` function always returns a `Promise`. You annotate the return as `Promise<T>`, where `T` is the type you `return`.

**3. Code**

```ts
interface User {
  id: number;
  name: string;
}

async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`https://api.example.com/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return (await res.json()) as User; // ⚠️ unchecked assertion, see Part 8
}

// Async arrow
const loadUsers = async (): Promise<User[]> => [];

// Void async
async function sendEmail(to: string): Promise<void> {
  await Promise.resolve(to);
}
```

**4. ❌ Wrong / ✅ Right**

```ts
const user: User = fetchUser(1);
// ❌ TS2740: Type 'Promise<User>' is missing the following properties from type 'User': id, name
const user2: User = await fetchUser(1); // ✅ (top-level await needs module ESNext/NodeNext + ES2022 target)

async function bad(): User {
  /* ... */
}
// ❌ TS1064: The return type of an async function or method must be the global Promise<T> type.
```

**5. Real-world use: typed `asyncHandler` for Express**

```ts
import type { Request, Response, NextFunction, RequestHandler } from 'express';

const asyncHandler =
  (
    fn: (req: Request, res: Response, next: NextFunction) => Promise<void>,
  ): RequestHandler =>
  (req, res, next) => {
    fn(req, res, next).catch(next); // forwards rejected promises to the error handler
  };
```

_Version note:_ Express 5 handles rejected promises in handlers itself, so this wrapper is mainly needed in Express 4 code. You will see it often in older codebases.

**6. Common mistakes**

- Forgetting `await`. The value is a `Promise<User>`, so `user.name` fails with `TS2339: Property 'name' does not exist on type 'Promise<User>'.`
- Forgetting that `res.json()` is not validated. Depending on your typings it is `any` or `unknown`; either way TS cannot know the real shape.
- A "floating" promise (calling an async function without `await` or `.catch`) is **not** a TS error. The ESLint rule `@typescript-eslint/no-floating-promises` catches it (Part 7).

**7. Interview tip**
_"What does an `async` function return?"_ Always a `Promise`. `return 5` in an async function gives `Promise<number>`.

---

## 2.10 Function overloads 🟡 GOOD TO KNOW

**1. What it is**
Several **call signatures** for one function, so the return type can depend on which arguments were passed. You write the signatures, then **one implementation** that handles all of them.

**2. Why it exists**
For functions that behave differently depending on input shape, and where a union return type would force callers to narrow.

**3. Code**

```ts
function toDate(timestamp: number): Date; // overload 1
function toDate(year: number, month: number, day: number): Date; // overload 2
function toDate(a: number, b?: number, c?: number): Date {
  // implementation (hidden from callers)
  if (b !== undefined && c !== undefined) return new Date(a, b - 1, c);
  return new Date(a);
}

toDate(1700000000000); // ✅
toDate(2025, 1, 15); // ✅
toDate(2025, 1);
// ❌ TS2575: No overload expects 2 arguments, but overloads do exist that expect either 1 or 3 arguments.

// Overloads that depend on the argument TYPE
function parse(input: string): number;
function parse(input: string[]): number[];
function parse(input: string | string[]): number | number[] {
  return typeof input === 'string' ? Number(input) : input.map(Number);
}
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Unnecessary overloads
function len(x: string): number;
function len(x: unknown[]): number;
function len(x: string | unknown[]): number {
  return x.length;
}

// ✅ A union does the same job
function lenOk(x: string | unknown[]): number {
  return x.length;
}
```

Rule: **prefer a union or optional parameter unless the return type changes with the input.** Order matters: TS picks the first matching overload, so put the most specific first.

**6. Common mistakes & errors**

- `TS2394: This overload signature is not compatible with its implementation signature.` The implementation must accept everything every overload accepts.
- `TS2769: No overload matches this call.` The error lists each overload's reason. Read the last one first.
- The implementation signature is **not** callable from outside. Only the overloads are.

**7. Interview tip**
_"When would you use overloads?"_ When different argument shapes produce different return types. Mention you prefer unions or generics when they are enough.

---

## 2.11 The `this` parameter 🟡 GOOD TO KNOW

**1. What it is**
A fake first parameter named `this` that declares what `this` must be inside a function. It exists **only for the compiler** and is erased in the output.

**2. Why it exists**
In JS, `this` depends on _how_ a function is called, which is a classic source of bugs. The `this` parameter lets TS check the call site.

**3. Code**

```ts
interface Button {
  label: string;
}

function handleClick(this: Button, event: string): void {
  console.log(`${this.label} received ${event}`);
}

const btn = { label: 'Save', handleClick };
btn.handleClick('click'); // ✅ called as a method on a Button-shaped object
handleClick('click');
// ❌ TS2684: The 'this' context of type 'void' is not assignable to method's 'this' of type 'Button'.

// Arrow functions use the surrounding `this`, so they cannot declare their own
```

**4. ❌ Wrong / ✅ Right**

```ts
class Counter {
  count = 0;
  increment() {
    this.count++;
  } // ❌ loses `this` when detached: const f = c.increment; f()
  incrementSafe = () => {
    this.count++;
  }; // ✅ arrow property keeps `this`
}
```

TS does **not** warn about the detached `increment` call unless you add a `this` parameter. This is why `this` bugs still slip through.

**5. Real-world use**
Mostly in older class-based code and callback-based APIs (`addEventListener`, jQuery-style libraries). In modern React with function components and hooks, you rarely write `this`.

**6. Common mistake**
`TS2683: 'this' implicitly has type 'any' because it does not have a type annotation.` This comes from `noImplicitThis` (part of `strict`). Add a `this` parameter or use an arrow function.

**7. Interview tip**
_"Is `this` in a function parameter list a real parameter?"_ No. It is erased at compile time.

---

### 📌 Key Takeaways

- Always annotate parameters; annotate return types on exported functions.
- Optional (`?`) and rest parameters come last; use defaults when you want a plain type inside the function.
- Describe functions with `(a: A) => R`; contextual typing fills in callback parameters where the type is known.
- A callback typed `() => void` may still return a value; a function declared `: void` may not.
- Use generics, not `any`, for reusable functions; use overloads only when the return type depends on the arguments.
- `async` functions return `Promise<T>`; always `await`, and remember `res.json()` is not validated.

✅ Part 2 complete.
