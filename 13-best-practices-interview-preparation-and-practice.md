# Part 13a: Best Practices, Naming & Folder Structure

Part 13 is split into four replies to stay within the word limit:

- **13a** (this reply): best practices and anti-patterns, naming conventions, folder structure.
- **13b**: 50 interview questions (Beginner, Intermediate, Advanced).
- **13c**: 10 coding/typing challenges with solutions.
- **13d**: 3 portfolio projects, the one-page cheat sheet, and the job-ready checklist.

Many items below are **team opinion**. They are marked as such. In an interview, say what you prefer and that you follow the project's convention.

---

## 13.1 Best practices 🔴 MUST KNOW

**1. What it is**
Habits that keep a TS codebase safe and readable as it grows. Each one links back to the Part where it was taught.

**2. Why it exists**
TS is only as safe as the code you write. `any`, casts and loose configs quietly turn it back into JavaScript.

**3. The rules, with examples**

**Rule 1: Always use `"strict": true`** (0.5). Add `noUncheckedIndexedAccess` for new projects. Turn flags on gradually in old ones (12.1).

**Rule 2: Prefer `unknown` over `any`** (1.4, 12.2).

```ts
// ❌
function parse(json: string): any {
  return JSON.parse(json);
}
// ✅ the caller must narrow or validate
function parseSafe(json: string): unknown {
  return JSON.parse(json);
}
```

**Rule 3: Validate data from outside; do not cast it** (8.6, 8.7).

```ts
const user = (await res.json()) as User; // ❌ a claim
const user2 = UserSchema.parse(await res.json()); // ✅ a check
```

**Rule 4: Annotate boundaries, infer the inside** (1.1). Annotate function parameters, exported return types, and empty `[]`/`{}`/`null` initial values. Let inference handle local variables.

```ts
export function total(items: OrderItem[]): number {
  // ✅ boundary annotated
  const sum = items.reduce((acc, i) => acc + i.price, 0); // ✅ inferred
  return sum;
}
```

**Rule 5: Derive types; do not copy them** (4.4, 5.x, 8.7). One source of truth means a change in one place updates everywhere.

```ts
// ❌ Three hand-written copies that drift apart
interface CreateUser {
  name: string;
  email: string;
}
interface UpdateUser {
  name?: string;
  email?: string;
}

// ✅ Derived
type CreateUser2 = z.infer<typeof CreateUserSchema>;
type UpdateUser2 = Partial<CreateUser2>;
```

**Rule 6: Prefer unions of literals over enums in new code** (1.7). Team opinion. You must still read enums.

```ts
type Role = 'admin' | 'user'; // ✅ no runtime code, works with JSON
const ROLES = ['admin', 'user'] as const; // ✅ if you also need the values at runtime
type RoleFromList = (typeof ROLES)[number];
```

**Rule 7: Model states as discriminated unions, not bags of optionals** (3.7).

```ts
// ❌ Can `data` and `error` both be set? Nobody knows.
interface State {
  loading: boolean;
  data?: User[];
  error?: string;
}
// ✅
type State2 =
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; error: string };
```

**Rule 8: Make `switch` statements exhaustive** (3.11). End with `assertNever`, so a new union member breaks the build in the right places.

**Rule 9: Narrow instead of asserting** (3.8, 3.9). Reach for `as` or `!` only when you can name the reason, and write it in a comment.

```ts
const el = document.getElementById('root')!; // ✅ acceptable: index.html guarantees it (10.1)
const name = user!.name; // ❌ "I'll just assume it's there"
if (!user) throw new NotFoundError('User', id); // ✅ prove it
```

**Rule 10: Use `import type` for types** (7.2). It is required under `verbatimModuleSyntax` and `isolatedModules`.

**Rule 11: Use `readonly` and `as const` for data that should not change** (1.5, 1.10, 3.3). Remember `readonly` is shallow and compile-time only.

**Rule 12: Keep types simple.** A type another developer cannot read in 30 seconds is a liability. Name intermediate types (5.9, 5.16), and stop at "good enough" instead of building clever generics (4.9).

**Rule 13: Handle errors honestly** (8.3, 8.4). `catch (e)` is `unknown`. Use custom error classes, never `throw "string"`, and re-throw what you cannot handle.

**Rule 14: Keep a lint setup that backs TS up** (7.11). The most valuable rules are `no-floating-promises`, `no-misused-promises`, `no-explicit-any`, `consistent-type-imports` and the `no-unsafe-*` family.

**Rule 15: Run `tsc --noEmit` in CI.** `tsx`, Vite and Vitest do not type check (0.3, 9.15, 10.1).

**4. ❌ Wrong / ✅ Right: the anti-pattern table**

| Anti-pattern                                  | Why it is bad                                       | Better                                             |
| --------------------------------------------- | --------------------------------------------------- | -------------------------------------------------- |
| `any` everywhere                              | Disables checking, spreads to everything it touches | `unknown`, real interfaces, generics               |
| `as X` on API data                            | Compiler believes a lie                             | Zod `parse` / type guard                           |
| `x!` to silence errors                        | Crashes at runtime if wrong                         | Narrow, throw a clear error                        |
| `// @ts-ignore` with no reason                | Hides future real bugs                              | `@ts-expect-error: reason` (12.3)                  |
| `Function`, `Object`, `{}` as types           | Far too wide                                        | `(a: A) => R`, `object`, `Record<string, unknown>` |
| Optional on every field                       | Callers must handle `undefined` everywhere          | Required fields; separate create/update types      |
| Return-only generics (`get<T>(url)`)          | Hidden cast                                         | Pass a schema (8.8, 10.17)                         |
| Hand-written type beside a schema             | Two sources of truth                                | `z.infer`                                          |
| `Date` in an API DTO                          | JSON turns it into a string                         | ISO string (11.3)                                  |
| `enum` + `const enum` mixed with Vite/esbuild | Tooling problems (1.7)                              | String-literal union                               |
| Deep inheritance trees                        | Rigid, hard to change                               | Composition (6.6)                                  |
| `React.FC` for generic components             | Cannot declare `T` (10.2, 10.16)                    | Function declaration                               |
| Disabling `strict` to "make errors go away"   | Hides the bugs                                      | Fix them gradually (12.1)                          |
| Giant `types.ts` for everything               | Becomes a dumping ground                            | Put types next to the code that owns them          |

**5. Real-world use**
Code reviewers look for these exact patterns. A pull request that adds `as any` will be questioned. One that adds a `// reason:` comment next to an assertion will usually pass.

**6. Common mistakes**

- Treating rules as laws. "Never use `any`" is a good default, but `(...args: any[]) => any` in a generic constraint is fine (12.2).
- Fixing symptoms. If you need five casts to make something compile, the types are probably wrong, not the compiler.
- Over-engineering types for code that three people will ever read.

**7. Interview tip**
_"What are your top TypeScript best practices?"_ Answer in this order: `strict` on; `unknown` over `any`; validate external data; derive types from one source; discriminated unions with exhaustive checks. Then give one example from your own project.

---

## 13.2 Naming conventions 🔴 MUST KNOW

**1. What it is**
Consistent names so code reads the same everywhere. These are widely used conventions, not language rules, so a team's style guide always wins.

**3. The table**

| Thing                                        | Convention                                  | Example                                 |
| -------------------------------------------- | ------------------------------------------- | --------------------------------------- |
| Variables, functions, parameters, properties | `camelCase`                                 | `userId`, `getUser()`                   |
| Types, interfaces, classes, enums            | `PascalCase`                                | `User`, `ApiResponse<T>`, `UserService` |
| React components                             | `PascalCase`                                | `UserCard`                              |
| Custom hooks                                 | `camelCase` starting with `use`             | `useDebounce`                           |
| Constants (true constants)                   | `UPPER_SNAKE_CASE` or `camelCase`           | `MAX_RETRIES`, `defaultSettings`        |
| Generic type parameters                      | `T`, `K`, `V`, or descriptive `TItem`       | `Repository<TEntity>`                   |
| Union literal values                         | Match the data (often lowercase or `UPPER`) | `"admin"`, `"NOT_FOUND"`                |
| Enum members                                 | `PascalCase`                                | `Role.Admin`                            |
| Booleans                                     | Start with `is`, `has`, `can`, `should`     | `isActive`, `hasAccess`                 |
| Event handler props / handlers               | `onX` for props, `handleX` for the function | `onSelect`, `handleSelect`              |
| Zod schemas                                  | `PascalCase` + `Schema`                     | `CreateUserSchema`                      |
| Inferred types from schemas                  | Plain name or `Dto`/`Input`                 | `CreateUserInput`, `UserDto`            |
| Props types                                  | `ComponentNameProps`                        | `UserCardProps`                         |

**Files**

- Frontend components: `UserCard.tsx` (PascalCase) is the common convention.
- Everything else: `user.service.ts` or `user-service.ts` (kebab-case, with a role suffix) is common.
- Tests: `user.service.test.ts` next to the file, or in a `__tests__` folder.
- Declaration files: `express.d.ts`, `env.d.ts`.
- Pick one file naming style per project. Mixed styles also cause trouble on case-insensitive file systems: `forceConsistentCasingInFileNames` (0.6) catches it.

**4. ❌ Wrong / ✅ Right**

```ts
interface IUser {
  id: number;
} // ⚠️ `I` prefix: carried over from C#/Java
interface User {
  id: number;
} // ✅ most modern TS style guides

type UserType = { id: number }; // ❌ `Type` suffix adds nothing
type User2 = { id: number }; // ✅

function getUserData(a: string, b: number) {} // ❌ meaningless names
function getUser(email: string, retries: number) {} // ✅
```

About `IUser`: Part 9.9 used `IUser` for the Mongoose document shape, and you will meet that style in many Mongoose projects. Both are accepted in the wild; new code usually drops the `I`. It is team opinion.

**Names that explain the layer**

```ts
type User = ...;              // domain type
type UserDto = ...;           // what goes over HTTP (11.3)
type UserRow = ...;           // database row (11.6)
type CreateUserInput = ...;   // what a service accepts
```

Suffixes like these save you from mixing up the three kinds of `User` (11.3).

**5. Real-world use**
Linters can enforce naming (`@typescript-eslint/naming-convention`). Prettier does not rename anything; it only formats.

**6. Common mistakes**

- Names that describe the type instead of the meaning (`stringValue`, `dataObject`).
- Abbreviations nobody else knows (`usrMgr`).
- Boolean names that read backwards (`notActive`, which turns into `!notActive`).
- A generic named `T` in a signature with four type parameters. Use descriptive names there (4.9).

**7. Interview tip**
_"Do you prefix interfaces with `I`?"_ Mostly no in modern TS, but follow the project's convention. A good answer shows you know both exist.

---

## 13.3 Folder structure 🔴 MUST KNOW

**1. What it is**
Where files live. Two common approaches exist, and both work.

| Approach       | Groups files by                              | Good for                            |
| -------------- | -------------------------------------------- | ----------------------------------- |
| **By layer**   | Technical role (`controllers/`, `services/`) | Small and medium backends, learning |
| **By feature** | Business area (`users/`, `orders/`)          | Larger apps and teams               |

**3. Backend: by layer** (what Part 9.2 used)

```
src/
  app.ts   server.ts
  config/env.ts
  routes/  controllers/  services/  repositories/
  middleware/  schemas/  errors/  types/
```

**Backend: by feature** (common as the app grows)

```
src/
  app.ts   server.ts
  config/
  shared/            # errors, middleware, utils used by many features
  features/
    users/
      user.routes.ts
      user.controller.ts
      user.service.ts
      user.repository.ts
      user.schema.ts
      user.test.ts
    todos/
      ...
```

Everything about "users" lives together, so deleting or changing a feature touches one folder.

**Frontend (React + Vite): by feature**

```
src/
  main.tsx   App.tsx
  api/               # typed API client (11.4)
  components/        # shared UI: Button, Modal, Table<T>
  features/
    auth/
      LoginForm.tsx
      useAuth.ts
      AuthProvider.tsx
    todos/
      TodoPage.tsx
      todoQueries.ts
  hooks/             # shared hooks: useDebounce, useLocalStorage
  lib/               # small helpers
  store/             # Redux Toolkit, if used (10.20)
  types/             # global .d.ts and shared frontend types
  vite-env.d.ts
```

**Monorepo** (Part 11.2)

```
packages/
  shared/    # schemas and DTO types only
  api/
  web/
```

**Rules that matter in any layout**

1. **Dependencies point one way:** routes → controllers → services → repositories. A repository never imports a controller.
2. **Keep types next to their owner.** Put `User` beside the user feature, not in a global `types.ts`. Share a type only when two features really need it.
3. **Keep frameworks at the edges.** `Request`/`Response` stay in controllers and middleware; React stays in `web` (9.2).
4. **Use barrel files (`index.ts` that re-exports) carefully.** They shorten imports, but large barrels can create circular imports and slow bundlers and test runs. Many teams use them only at package boundaries, such as `@shop/shared`.
5. **Keep the composition root in one place** (9.8): the only file that knows which concrete classes are used.
6. **Do not nest folders deeper than you need.** Three levels is usually enough.

**4. ❌ Wrong / ✅ Right**

```
// ❌ One 2,000-line types.ts, one 1,500-line utils.ts
// ✅ user.schema.ts, todo.schema.ts, date.ts, money.ts: names say what is inside

// ❌ components/ holds a feature page that only one route uses
// ✅ features/todos/TodoPage.tsx
```

**5. Real-world use**
When you join a team, follow the existing structure even if you prefer another. Propose changes separately from feature work.

**6. Common mistakes**

- Restructuring a project instead of building the feature.
- Circular imports (`A` imports `B` imports `A`). They cause `undefined` at runtime that TS may not flag. Tools such as `madge` or the ESLint rule `import/no-cycle` find them.
- Putting everything in `shared/` "just in case". It becomes a second dumping ground.

**7. Interview tip**
_"How do you structure a TS project?"_ Describe layers or features, say which you chose and why, and mention one-way dependencies and keeping Express/React types at the edges.

---

### 📌 Key Takeaways

- Core habits: `strict` on, `unknown` over `any`, validate outside data, derive types from one source, discriminated unions with exhaustive checks.
- Every `as`, `!`, `@ts-expect-error` and `any` should have a reason you can state in one sentence.
- Run `tsc --noEmit` and ESLint in CI, because `tsx`, Vite and Vitest do not type check.
- Naming: `PascalCase` for types and components, `camelCase` for values, `is/has` for booleans, and suffixes (`Dto`, `Row`, `Input`, `Schema`) that show which layer a type belongs to.
- Structure by layer when small, by feature as you grow; dependencies point one way, and types live next to their owner.
- Many of these are team opinions (enums, `I` prefix, barrels, file names). Know both sides and follow the project.

✅ Part 13a complete.

---

# Part 13b: 50 TypeScript Interview Questions

**How to use this:** cover the answer, say yours out loud, then compare. Answers are short on purpose. In a real interview, add **one example from your own project** to any answer.

Each question links to the Part that taught it. Opinion-based topics are flagged, because interviewers often want to hear that you know both sides.

---

## Beginner (Q1-Q20) 🔴 MUST KNOW

**Q1. What is TypeScript?**
JavaScript plus a static type system. The compiler checks types before the code runs, then removes them and emits plain JS. (0.1)

**Q2. TypeScript vs JavaScript?**
JS checks types at runtime; TS checks them at compile time. All valid JS syntax is valid TS syntax (TS is a superset). Browsers and Node never run TS directly. (0.1)

**Q3. What is type erasure?**
The compiler deletes all type syntax when emitting JS, so types do not exist at runtime. That is why you cannot `instanceof` an interface and why API data needs runtime validation. (0.2, 8.6)

**Q4. How do you run a TS file?**
Compile with `tsc` then run the JS with `node`, or run directly with `tsx` (or `ts-node` in older projects). `tsx` and Vite do **not** type check, so run `tsc --noEmit` too. (0.3)

**Q5. `any` vs `unknown`?**
Both accept any value. `any` turns checking off for it. `unknown` forces you to narrow or validate before use. Prefer `unknown`. (1.4)

**Q6. `void` vs `never`?**
`void`: the function returns nothing useful. `never`: the function never returns normally (it throws or loops forever), or a case that cannot happen. (1.4, 3.11)

**Q7. Type inference vs annotation: when do you annotate?**
Let TS infer local variables with initial values. Annotate function parameters, exported return types, and empty `[]`, `{}` or `null` initial values. (1.1)

**Q8. `interface` vs `type`?**
Both describe objects. `type` can also name unions, tuples and mapped/conditional types. `interface` supports `extends` and declaration merging. Follow the team's convention. (3.2)

**Q9. Union vs intersection?**
`A | B` is either; you can use only members common to both until you narrow. `A & B` is both; it combines all properties. (1.9)

**Q10. What is a literal type?**
An exact value as a type: `"GET"`, `42`, `true`. Combined into unions, they restrict values to a small set. `as const` keeps literals from widening. (1.8, 1.10)

**Q11. Why do many teams prefer unions over enums?**
Unions emit no runtime code, accept plain strings, and match JSON directly. Enums emit objects and have tooling friction (isolated compilers, `erasableSyntaxOnly`). Still, you must read enums. (1.7)

**Q12. What is a tuple?**
An array with fixed length and a known type at each position: `[string, number]`. `useState` returns one. (1.6)

**Q13. `phone?: string` vs `phone: string | undefined`?**
The first lets you omit the key; the second requires the key to be present. (3.3)

**Q14. What does `readonly` do?**
Blocks reassignment at compile time. It is shallow (`readonly string[]` is needed to lock an array's contents) and has no runtime effect. (3.3)

**Q15. What does `strict: true` enable?**
A group of checks. The two that matter most: `noImplicitAny` and `strictNullChecks`. Also `strictFunctionTypes`, `strictPropertyInitialization`, `useUnknownInCatchVariables` and others. (0.5)

**Q16. `??` vs `||`?**
`||` replaces any falsy value (`0`, `""`, `false`). `??` replaces only `null` and `undefined`. (1.3)

**Q17. What is a type assertion, and what is the risk?**
`value as T` tells the compiler to trust you. There is no runtime check, so a wrong assertion becomes a runtime crash. (1.10)

**Q18. What does `!` (non-null assertion) do?**
Tells TS a value is not `null`/`undefined`. Same risk as `as`. Prefer a check or throwing a clear error. (1.10)

**Q19. What is `tsconfig.json`?**
Configuration for the compiler: which files to include, strictness, output target, module system and more. It also drives VS Code's error checking. (0.4)

**Q20. Why is `e` in `catch (e)` of type `unknown`?**
JS lets you throw anything. With `strict`, TS makes the variable `unknown`, so you narrow (`e instanceof Error`) before using it. (8.3)

---

## Intermediate (Q21-Q38) 🔴 MUST KNOW

**Q21. What are generics and why use them?**
Type parameters filled in at the call site, so input and output types stay linked without `any`: `function first<T>(items: T[]): T | undefined`. (4.1)

**Q22. What does `T extends X` mean?**
`T` must be assignable to `X`. It is a minimum requirement, not inheritance, and the caller's full type is preserved. (4.3)

**Q23. Write a type-safe `getProp`.**

```ts
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

(4.4)

**Q24. What is a discriminated union?**
A union of object types sharing a literal property (like `status`). Checking that property narrows to one member. Used for state, actions and API results. (3.7)

**Q25. What is a custom type guard?**
A function returning `value is T`. When it returns true, TS narrows the value. TS does not verify the body, so keep guards correct or use Zod. (3.9)

**Q26. Name ways to narrow a type.**
`typeof`, `instanceof`, `in`, truthiness, equality, discriminant checks, `Array.isArray`, custom guards and assertion functions. (3.8-3.10)

**Q27. How do you make a `switch` exhaustive?**
Add a `default` branch that passes the value to a function taking `never`. Adding a new union member then fails the build. (3.11)

**Q28. What do `Partial`, `Pick`, `Omit` and `Record` do?**
`Partial`: all optional. `Pick`: keep listed keys. `Omit`: drop listed keys. `Record<K, V>`: object keyed by `K`. (5.1-5.4)

**Q29. What is the gotcha with `Omit`?**
It does not check that the key exists, so a typo passes silently. It also collapses unions to their common keys. Use a strict or distributive version. (5.3, 5.11)

**Q30. How do you get a function's resolved return type?**
`Awaited<ReturnType<typeof fn>>`. Remember `typeof` for values. (5.6, 5.7)

**Q31. Is TS structural or nominal? What are excess property checks?**
Structural: shape matters, not name. The one exception is fresh object literals, where extra properties are flagged as likely typos. Data from variables or the network is not checked this way. (3.6)

**Q32. `as` vs annotation vs `satisfies`?**
Annotation changes the variable's type. `as` overrides it unsafely. `satisfies` validates against a type while keeping the precise inferred type. (1.10)

**Q33. When do you use function overloads?**
When the return type depends on the arguments. Otherwise a union, optional parameter or generic is simpler. The implementation signature is not callable from outside. (2.10)

**Q34. How do you type `req.body` in Express, and what is the catch?**
Third generic of `Request`: `Request<Params, ResBody, ReqBody>`. It is an unchecked claim, so validate with Zod and use the parsed result. (9.4, 9.12)

**Q35. How do you add `req.user` to Express's `Request`?**
A `.d.ts` file inside tsconfig's `include`, with `declare global { namespace Express { interface Request { user?: ... } } }` and an `export {}` or an import so it is a module. (7.7)

**Q36. TS has types; why validate API input?**
Types vanish at runtime, so TS cannot know what a client or server really sends. A Zod schema validates at runtime and `z.infer` gives the type from the same source. (8.6, 8.7)

**Q37. What does `import type` do?**
Imports only for type checking and is erased entirely. It is required under `verbatimModuleSyntax` and helps tools that compile one file at a time. (7.2)

**Q38. What are `@types/*` packages?**
Community-written declarations from DefinitelyTyped for JS libraries without bundled types. Install as dev dependencies, and not for libraries that ship their own types. (7.5)

---

## Advanced (Q39-Q50) 🟡 GOOD TO KNOW

**Q39. What is a mapped type? Implement `Partial<T>`.**
A type that loops over keys to build a new object type.

```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
```

`-?` and `-readonly` remove modifiers. (5.10)

**Q40. What is a distributive conditional type?**
A conditional on a naked type parameter applies to each union member separately: `ToArray<string | number>` gives `string[] | number[]`. Wrap in `[T]` to stop it. (5.11)

**Q41. What does `infer` do?**
Inside a conditional's `extends` clause it names a part of the matched type: `T extends Promise<infer V> ? V : T`. It is pattern matching for types. (5.12)

**Q42. What are template literal types?**
String types built like template strings, such as `` `on${Capitalize<Event>}` ``. Unions multiply into every combination. Used for typed routes and event names. (5.13)

**Q43. What is `strictFunctionTypes`, and how does it affect callbacks?**
It makes function-type parameters contravariant (checked strictly). Method syntax (`method(x: T): void`) stays bivariant (looser), a historical compromise. (0.5)

**Q44. What is declaration merging, and where is it used?**
Two `interface` declarations with the same name combine into one. It powers module and global augmentation (extending `Express.Request`, `Window`). `type` aliases cannot merge. (3.5, 7.7)

**Q45. `private` vs `#private`?**
`private` is checked only by TS and erased at runtime. `#private` is real JS privacy enforced at runtime. Neither is a security feature. (6.2)

**Q46. Legacy vs standard decorators?**
Legacy needs `experimentalDecorators` (used by NestJS, TypeORM) and supports parameter decorators and metadata emit. Standard decorators (TS 5.0+) need no flag. They are not compatible. (6.10)

**Q47. Why do imports need `.js` in a TS file under Node ESM?**
TS does not rewrite import paths, and Node ESM needs the extension of the file that will exist at runtime. `module`, `moduleResolution` and `package.json` `"type"` must agree. (7.1)

**Q48. `@ts-ignore` vs `@ts-expect-error`?**
Both silence the next line. `@ts-expect-error` itself errors when the line no longer has a problem, so stale suppressions are caught. Always add a reason. (12.3)

**Q49. How do you get end-to-end type safety in a full stack TS app?**
A shared package holds Zod schemas and inferred DTO types. The server validates input and maps database rows to DTOs. The client validates responses in one typed API module. Alternatives: OpenAPI codegen or tRPC. Types still do not cover authorization or deployment skew. (11.1-11.6)

**Q50. Is TypeScript's type system sound?**
No, by design. Holes include `any`, `as`, `!`, array covariance (`string[]` assignable to `(string | number)[]`), method bivariance, and unchecked index access (unless `noUncheckedIndexedAccess`). Types describe what you expect, not what is guaranteed. (1.10, 10.17)

---

## Answering well: a short checklist

| Do                                                     | Avoid                                                  |
| ------------------------------------------------------ | ------------------------------------------------------ |
| State the definition in one sentence first             | Starting with history or theory                        |
| Give the "why" (the problem it solves)                 | Reciting syntax only                                   |
| Mention the common mistake or trade-off                | Pretending there is one right answer on opinion topics |
| Use a real example (typing an API response, a reducer) | `foo`/`bar` examples                                   |
| Say "I'm not sure, but I'd check the docs" when unsure | Guessing                                               |

**Interview tip:** the most-asked topics are Q5 (`any` vs `unknown`), Q8 (`interface` vs `type`), Q21-Q23 (generics, `keyof`), Q24 (discriminated unions), Q29 (utility types) and Q36 (runtime validation). Practice those until you can answer each in 30 seconds.

---

### 📌 Key Takeaways

- Know the definition, the problem it solves and one trade-off for each core topic.
- The most asked: `any` vs `unknown`, `interface` vs `type`, generics with `keyof`, discriminated unions, utility types and runtime validation.
- Be able to write from memory: `getProp`, `Partial<T>`, an exhaustive `switch`, a type guard and an `ApiResponse<T>`.
- On opinion topics (enums, `I` prefix, interface vs type), show you know both sides and follow the project's convention.
- Be honest about limits: types are erased, assertions are unchecked, and TS is not sound.
- Back every answer with one example from code you actually wrote.

✅ Part 13b complete.

---

# Part 13b: 50 TypeScript Interview Questions

**How to use this:** cover the answer, say yours out loud, then compare. Answers are short on purpose. In a real interview, add **one example from your own project** to any answer.

Each question links to the Part that taught it. Opinion-based topics are flagged, because interviewers often want to hear that you know both sides.

---

## Beginner (Q1-Q20) 🔴 MUST KNOW

**Q1. What is TypeScript?**
JavaScript plus a static type system. The compiler checks types before the code runs, then removes them and emits plain JS. (0.1)

**Q2. TypeScript vs JavaScript?**
JS checks types at runtime; TS checks them at compile time. All valid JS syntax is valid TS syntax (TS is a superset). Browsers and Node never run TS directly. (0.1)

**Q3. What is type erasure?**
The compiler deletes all type syntax when emitting JS, so types do not exist at runtime. That is why you cannot `instanceof` an interface and why API data needs runtime validation. (0.2, 8.6)

**Q4. How do you run a TS file?**
Compile with `tsc` then run the JS with `node`, or run directly with `tsx` (or `ts-node` in older projects). `tsx` and Vite do **not** type check, so run `tsc --noEmit` too. (0.3)

**Q5. `any` vs `unknown`?**
Both accept any value. `any` turns checking off for it. `unknown` forces you to narrow or validate before use. Prefer `unknown`. (1.4)

**Q6. `void` vs `never`?**
`void`: the function returns nothing useful. `never`: the function never returns normally (it throws or loops forever), or a case that cannot happen. (1.4, 3.11)

**Q7. Type inference vs annotation: when do you annotate?**
Let TS infer local variables with initial values. Annotate function parameters, exported return types, and empty `[]`, `{}` or `null` initial values. (1.1)

**Q8. `interface` vs `type`?**
Both describe objects. `type` can also name unions, tuples and mapped/conditional types. `interface` supports `extends` and declaration merging. Follow the team's convention. (3.2)

**Q9. Union vs intersection?**
`A | B` is either; you can use only members common to both until you narrow. `A & B` is both; it combines all properties. (1.9)

**Q10. What is a literal type?**
An exact value as a type: `"GET"`, `42`, `true`. Combined into unions, they restrict values to a small set. `as const` keeps literals from widening. (1.8, 1.10)

**Q11. Why do many teams prefer unions over enums?**
Unions emit no runtime code, accept plain strings, and match JSON directly. Enums emit objects and have tooling friction (isolated compilers, `erasableSyntaxOnly`). Still, you must read enums. (1.7)

**Q12. What is a tuple?**
An array with fixed length and a known type at each position: `[string, number]`. `useState` returns one. (1.6)

**Q13. `phone?: string` vs `phone: string | undefined`?**
The first lets you omit the key; the second requires the key to be present. (3.3)

**Q14. What does `readonly` do?**
Blocks reassignment at compile time. It is shallow (`readonly string[]` is needed to lock an array's contents) and has no runtime effect. (3.3)

**Q15. What does `strict: true` enable?**
A group of checks. The two that matter most: `noImplicitAny` and `strictNullChecks`. Also `strictFunctionTypes`, `strictPropertyInitialization`, `useUnknownInCatchVariables` and others. (0.5)

**Q16. `??` vs `||`?**
`||` replaces any falsy value (`0`, `""`, `false`). `??` replaces only `null` and `undefined`. (1.3)

**Q17. What is a type assertion, and what is the risk?**
`value as T` tells the compiler to trust you. There is no runtime check, so a wrong assertion becomes a runtime crash. (1.10)

**Q18. What does `!` (non-null assertion) do?**
Tells TS a value is not `null`/`undefined`. Same risk as `as`. Prefer a check or throwing a clear error. (1.10)

**Q19. What is `tsconfig.json`?**
Configuration for the compiler: which files to include, strictness, output target, module system and more. It also drives VS Code's error checking. (0.4)

**Q20. Why is `e` in `catch (e)` of type `unknown`?**
JS lets you throw anything. With `strict`, TS makes the variable `unknown`, so you narrow (`e instanceof Error`) before using it. (8.3)

---

## Intermediate (Q21-Q38) 🔴 MUST KNOW

**Q21. What are generics and why use them?**
Type parameters filled in at the call site, so input and output types stay linked without `any`: `function first<T>(items: T[]): T | undefined`. (4.1)

**Q22. What does `T extends X` mean?**
`T` must be assignable to `X`. It is a minimum requirement, not inheritance, and the caller's full type is preserved. (4.3)

**Q23. Write a type-safe `getProp`.**

```ts
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

(4.4)

**Q24. What is a discriminated union?**
A union of object types sharing a literal property (like `status`). Checking that property narrows to one member. Used for state, actions and API results. (3.7)

**Q25. What is a custom type guard?**
A function returning `value is T`. When it returns true, TS narrows the value. TS does not verify the body, so keep guards correct or use Zod. (3.9)

**Q26. Name ways to narrow a type.**
`typeof`, `instanceof`, `in`, truthiness, equality, discriminant checks, `Array.isArray`, custom guards and assertion functions. (3.8-3.10)

**Q27. How do you make a `switch` exhaustive?**
Add a `default` branch that passes the value to a function taking `never`. Adding a new union member then fails the build. (3.11)

**Q28. What do `Partial`, `Pick`, `Omit` and `Record` do?**
`Partial`: all optional. `Pick`: keep listed keys. `Omit`: drop listed keys. `Record<K, V>`: object keyed by `K`. (5.1-5.4)

**Q29. What is the gotcha with `Omit`?**
It does not check that the key exists, so a typo passes silently. It also collapses unions to their common keys. Use a strict or distributive version. (5.3, 5.11)

**Q30. How do you get a function's resolved return type?**
`Awaited<ReturnType<typeof fn>>`. Remember `typeof` for values. (5.6, 5.7)

**Q31. Is TS structural or nominal? What are excess property checks?**
Structural: shape matters, not name. The one exception is fresh object literals, where extra properties are flagged as likely typos. Data from variables or the network is not checked this way. (3.6)

**Q32. `as` vs annotation vs `satisfies`?**
Annotation changes the variable's type. `as` overrides it unsafely. `satisfies` validates against a type while keeping the precise inferred type. (1.10)

**Q33. When do you use function overloads?**
When the return type depends on the arguments. Otherwise a union, optional parameter or generic is simpler. The implementation signature is not callable from outside. (2.10)

**Q34. How do you type `req.body` in Express, and what is the catch?**
Third generic of `Request`: `Request<Params, ResBody, ReqBody>`. It is an unchecked claim, so validate with Zod and use the parsed result. (9.4, 9.12)

**Q35. How do you add `req.user` to Express's `Request`?**
A `.d.ts` file inside tsconfig's `include`, with `declare global { namespace Express { interface Request { user?: ... } } }` and an `export {}` or an import so it is a module. (7.7)

**Q36. TS has types; why validate API input?**
Types vanish at runtime, so TS cannot know what a client or server really sends. A Zod schema validates at runtime and `z.infer` gives the type from the same source. (8.6, 8.7)

**Q37. What does `import type` do?**
Imports only for type checking and is erased entirely. It is required under `verbatimModuleSyntax` and helps tools that compile one file at a time. (7.2)

**Q38. What are `@types/*` packages?**
Community-written declarations from DefinitelyTyped for JS libraries without bundled types. Install as dev dependencies, and not for libraries that ship their own types. (7.5)

---

## Advanced (Q39-Q50) 🟡 GOOD TO KNOW

**Q39. What is a mapped type? Implement `Partial<T>`.**
A type that loops over keys to build a new object type.

```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
```

`-?` and `-readonly` remove modifiers. (5.10)

**Q40. What is a distributive conditional type?**
A conditional on a naked type parameter applies to each union member separately: `ToArray<string | number>` gives `string[] | number[]`. Wrap in `[T]` to stop it. (5.11)

**Q41. What does `infer` do?**
Inside a conditional's `extends` clause it names a part of the matched type: `T extends Promise<infer V> ? V : T`. It is pattern matching for types. (5.12)

**Q42. What are template literal types?**
String types built like template strings, such as `` `on${Capitalize<Event>}` ``. Unions multiply into every combination. Used for typed routes and event names. (5.13)

**Q43. What is `strictFunctionTypes`, and how does it affect callbacks?**
It makes function-type parameters contravariant (checked strictly). Method syntax (`method(x: T): void`) stays bivariant (looser), a historical compromise. (0.5)

**Q44. What is declaration merging, and where is it used?**
Two `interface` declarations with the same name combine into one. It powers module and global augmentation (extending `Express.Request`, `Window`). `type` aliases cannot merge. (3.5, 7.7)

**Q45. `private` vs `#private`?**
`private` is checked only by TS and erased at runtime. `#private` is real JS privacy enforced at runtime. Neither is a security feature. (6.2)

**Q46. Legacy vs standard decorators?**
Legacy needs `experimentalDecorators` (used by NestJS, TypeORM) and supports parameter decorators and metadata emit. Standard decorators (TS 5.0+) need no flag. They are not compatible. (6.10)

**Q47. Why do imports need `.js` in a TS file under Node ESM?**
TS does not rewrite import paths, and Node ESM needs the extension of the file that will exist at runtime. `module`, `moduleResolution` and `package.json` `"type"` must agree. (7.1)

**Q48. `@ts-ignore` vs `@ts-expect-error`?**
Both silence the next line. `@ts-expect-error` itself errors when the line no longer has a problem, so stale suppressions are caught. Always add a reason. (12.3)

**Q49. How do you get end-to-end type safety in a full stack TS app?**
A shared package holds Zod schemas and inferred DTO types. The server validates input and maps database rows to DTOs. The client validates responses in one typed API module. Alternatives: OpenAPI codegen or tRPC. Types still do not cover authorization or deployment skew. (11.1-11.6)

**Q50. Is TypeScript's type system sound?**
No, by design. Holes include `any`, `as`, `!`, array covariance (`string[]` assignable to `(string | number)[]`), method bivariance, and unchecked index access (unless `noUncheckedIndexedAccess`). Types describe what you expect, not what is guaranteed. (1.10, 10.17)

---

## Answering well: a short checklist

| Do                                                     | Avoid                                                  |
| ------------------------------------------------------ | ------------------------------------------------------ |
| State the definition in one sentence first             | Starting with history or theory                        |
| Give the "why" (the problem it solves)                 | Reciting syntax only                                   |
| Mention the common mistake or trade-off                | Pretending there is one right answer on opinion topics |
| Use a real example (typing an API response, a reducer) | `foo`/`bar` examples                                   |
| Say "I'm not sure, but I'd check the docs" when unsure | Guessing                                               |

**Interview tip:** the most-asked topics are Q5 (`any` vs `unknown`), Q8 (`interface` vs `type`), Q21-Q23 (generics, `keyof`), Q24 (discriminated unions), Q29 (utility types) and Q36 (runtime validation). Practice those until you can answer each in 30 seconds.

---

### 📌 Key Takeaways

- Know the definition, the problem it solves and one trade-off for each core topic.
- The most asked: `any` vs `unknown`, `interface` vs `type`, generics with `keyof`, discriminated unions, utility types and runtime validation.
- Be able to write from memory: `getProp`, `Partial<T>`, an exhaustive `switch`, a type guard and an `ApiResponse<T>`.
- On opinion topics (enums, `I` prefix, interface vs type), show you know both sides and follow the project's convention.
- Be honest about limits: types are erased, assertions are unchecked, and TS is not sound.
- Back every answer with one example from code you actually wrote.

✅ Part 13b complete.

---

# Part 13c: 10 Coding & Typing Challenges

**How to use this:** copy the starter into a scratch `.ts` file (strict mode on), try it for 10-15 minutes, then compare with the solution. Don't read the solution first, because the struggle is where you learn. Each challenge names the concept it tests and the Part to revisit if you get stuck.

All solutions compile under `"strict": true` and `noUncheckedIndexedAccess` with TS 5.x.

---

## Challenge 1: Type-safe `pick` 🔴

**Tests:** generics, `keyof`, `Pick` (4.4, 5.3)

**Task:** write `pick(obj, keys)` that returns a new object with only the listed keys. The return type must contain only those keys, and a wrong key must be a compile error.

```ts
interface User {
  id: number;
  name: string;
  email: string;
}
const user: User = { id: 1, name: 'Asha', email: 'asha@example.com' };

const preview = pick(user, ['id', 'name']);
preview.name; // ✅ string
preview.email; // ❌ should be an error
pick(user, ['age']); // ❌ should be an error
```

**Solution**

```ts
function pick<T extends object, K extends keyof T>(
  obj: T,
  keys: readonly K[],
): Pick<T, K> {
  const result = {} as Pick<T, K>; // accepted cast: we fill it in right below
  for (const key of keys) {
    result[key] = obj[key];
  }
  return result;
}
```

**Why it works:** `K extends keyof T` limits keys to real ones and keeps their literal types. `Pick<T, K>` gives the exact result shape.

**Common mistake:** `keys: string[]` and `Record<string, unknown>` as the return type. It compiles but loses all safety.

---

## Challenge 2: Generic `groupBy` 🔴

**Tests:** generics, `Map`, honest return types (4.1, 4.9)

**Task:** group an array by a key function.

```ts
const orders = [
  { id: 'a', status: 'paid' as const },
  { id: 'b', status: 'pending' as const },
  { id: 'c', status: 'paid' as const },
];
const byStatus = groupBy(orders, (o) => o.status); // Map<"paid" | "pending", {...}[]>
byStatus.get('paid'); // typed array, or undefined
```

**Solution**

```ts
function groupBy<T, K>(
  items: readonly T[],
  getKey: (item: T) => K,
): Map<K, T[]> {
  const groups = new Map<K, T[]>();
  for (const item of items) {
    const key = getKey(item);
    const bucket = groups.get(key);
    if (bucket) bucket.push(item);
    else groups.set(key, [item]);
  }
  return groups;
}
```

**Why a `Map`:** a `Record<K, T[]>` return type would claim every key exists, which is false (5.4). `Map.get` returns `T[] | undefined`, which is honest. A `Record`-returning version is possible, but it needs a cast inside, because TS cannot prove that `result[key] ??= []` is valid for an unresolved generic `K`.

**Follow-up:** convert it with `Object.fromEntries(byStatus)`. Note the result type is a loose `{ [k: string]: T[] }`.

---

## Challenge 3: Typed event emitter 🟡

**Tests:** generics with mapped types, `keyof`, indexed access (4.4, 5.10)

**Task:** an emitter where each event name has its own payload type.

```ts
type AppEvents = {
  login: { userId: string };
  logout: undefined;
  error: Error;
};
const bus = new TypedEmitter<AppEvents>();
bus.on('login', (p) => console.log(p.userId)); // p is { userId: string }
bus.emit('login', { userId: 'u1' }); // ✅
bus.emit('login', { id: 1 }); // ❌
bus.emit('signup', undefined); // ❌
```

**Solution**

```ts
type Listener<T> = (payload: T) => void;

class TypedEmitter<Events extends object> {
  private listeners: { [K in keyof Events]?: Listener<Events[K]>[] } = {};

  on<K extends keyof Events>(event: K, listener: Listener<Events[K]>): void {
    const list = this.listeners[event] ?? [];
    list.push(listener);
    this.listeners[event] = list;
  }

  emit<K extends keyof Events>(event: K, payload: Events[K]): void {
    this.listeners[event]?.forEach((listener) => listener(payload));
  }
}
```

**Why it works:** `K` links the event name to its payload through `Events[K]`. The mapped type stores one listener list per event.

**Common mistakes**

- Declaring the events with an `interface` while the constraint is `Record<string, unknown>`. Interfaces have no implicit index signature, so you get `TS2344`. `extends object` avoids this.
- Typing the listeners as `Function[]`. It compiles and checks nothing.

---

## Challenge 4: Three small type-level utilities 🟡

**Tests:** mapped types, conditional types, `infer` (5.10-5.12)

**Task:** implement these without using the built-ins.

1. `ElementOf<T>`: the element type of an array.
2. `Mutable<T>`: removes `readonly` from every property.
3. `WithRequired<T, K>`: makes only the keys `K` required.

```ts
// Tiny test helpers (a common trick, copy as-is)
type Equal<A, B> =
  (<T>() => T extends A ? 1 : 2) extends <T>() => T extends B ? 1 : 2
    ? true
    : false;
type Expect<T extends true> = T;

type A = Expect<Equal<ElementOf<string[]>, string>>;
type B = Expect<Equal<Mutable<{ readonly id: number }>, { id: number }>>;
// WithRequired<{ a?: string; b?: number }, "a"> should be { a: string; b?: number }
```

**Solution**

```ts
type ElementOf<T> = T extends readonly (infer E)[] ? E : never;

type Mutable<T> = { -readonly [K in keyof T]: T[K] };

type WithRequired<T, K extends keyof T> = T & { [P in K]-?: T[P] };
```

**Why they work:** `infer E` names the matched element. `-readonly` removes a modifier. In `WithRequired`, the intersection makes `a` required, because a property that is optional in one member and required in the other is required overall. Hover over the result to see the merged type, or wrap it in `Prettify` (5.16).

**Common mistake:** writing `T extends (infer E)[]`, which fails for `readonly string[]`. Use `readonly (infer E)[]` to match both.

---

## Challenge 5: Fix the broken code 🔴

**Tests:** reading errors, removing `any`, validating input (12.2, 8.7)

**Task:** this code compiles in a loose project. Under `strict`, find every problem and fix it without using `any` or `as`.

```ts
function summarize(orders) {
  const totals = {};
  for (const o of orders) {
    totals[o.userId] = (totals[o.userId] || 0) + o.amount;
  }
  return Object.keys(totals).map((id) => ({ id, total: totals[id] }));
}

const result = summarize(JSON.parse(raw) as any);
```

**Problems the compiler (and a reviewer) would flag**

- `TS7006`: `orders` is implicitly `any`.
- `TS7053`: `totals` is `{}`, so indexing it by a string is an error.
- `|| 0` would also replace a legitimate `0`. It is harmless here, but `??` states the intent (1.3).
- `as any` hides all of it, and `JSON.parse` output is never validated.

**Solution**

```ts
import { z } from 'zod';

interface Order {
  userId: string;
  amount: number;
}
interface Summary {
  id: string;
  total: number;
}

function summarize(orders: readonly Order[]): Summary[] {
  const totals: Record<string, number> = {};
  for (const o of orders) {
    totals[o.userId] = (totals[o.userId] ?? 0) + o.amount;
  }
  return Object.entries(totals).map(([id, total]) => ({ id, total }));
}

const OrdersSchema = z.array(
  z.object({ userId: z.string(), amount: z.number() }),
);
const result = summarize(OrdersSchema.parse(JSON.parse(raw)));
```

**Lesson:** type the **boundaries** (function parameters, external data) and the inside mostly types itself.

---

## Challenge 6: Typed `retry` 🔴

**Tests:** async generics, `unknown` errors (2.9, 8.3)

**Task:** `retry(fn, attempts, delayMs)` runs an async function and retries on failure with a growing delay. It must preserve the result type of `fn`.

```ts
const user = await retry(() => fetchUser(1), 3, 200); // user: User
```

**Solution**

```ts
const sleep = (ms: number): Promise<void> =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function retry<T>(
  fn: () => Promise<T>,
  attempts = 3,
  delayMs = 200,
): Promise<T> {
  let lastError: unknown;
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (e) {
      lastError = e;
      if (i < attempts - 1) await sleep(delayMs * 2 ** i); // 200, 400, 800...
    }
  }
  throw lastError; // throwing `unknown` is allowed
}
```

**Why it works:** `T` flows from `fn`'s return type to `retry`'s return type. This is a generic that appears twice, as 4.9 requires.

**Common mistakes**

- `return fn()` without `await` inside `try`. A rejected promise then escapes the `catch` and is never retried.
- Retrying on every error, including `400 Bad Request`. In real code, retry only on network errors and `5xx` responses.
- Calling it with `attempts = 0`. The function throws `undefined`. Validate the argument in production code.

---

## Challenge 7: `mapResult` on a `Result` type 🟡

**Tests:** discriminated unions, generics, `never` (3.7, 8.5)

**Task:** given the `Result` type from 8.5, write `mapResult`, which transforms the success value and passes failures through untouched.

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });

const r = ok(21);
const doubled = mapResult(r, (n) => n * 2); // Result<number, never>
```

**Solution**

```ts
function mapResult<T, U, E>(
  result: Result<T, E>,
  fn: (value: T) => U,
): Result<U, E> {
  if (result.ok) return { ok: true, value: fn(result.value) };
  return result; // narrowed to the failure member: already a valid Result<U, E>
}
```

**Why the last line compiles:** the failure member `{ ok: false; error: E }` has no `T` in it, so it is assignable to `Result<U, E>` as is.

**Follow-up:** write `andThen`, where `fn` itself returns a `Result<U, E>`. This is how Result chains compose.

---

## Challenge 8: A typed endpoint map 🟡

**Tests:** indexed access, conditional types, tuple parameters (4.4, 5.11)

**Task:** declare `call` so that the route name decides the body and response types. GET routes take no body.

```ts
type Todo = { id: string; title: string };

type Endpoints = {
  'GET /todos': { response: Todo[] };
  'POST /todos': { body: { title: string }; response: Todo };
};

const list = await call('GET /todos'); // Todo[]
const created = await call('POST /todos', { title: 'x' }); // Todo
call('POST /todos'); // ❌ body required
call('GET /todos', { title: 'x' }); // ❌ no body allowed
```

**Solution (the signature is the challenge)**

```ts
type BodyArgs<K extends keyof Endpoints> = Endpoints[K] extends {
  body: infer B;
}
  ? [body: B]
  : [];

declare function call<K extends keyof Endpoints>(
  route: K,
  ...args: BodyArgs<K>
): Promise<Endpoints[K]['response']>;
```

**Why it works:** the rest parameter is a **tuple** computed from the route. For POST it is `[body: { title: string }]`; for GET it is `[]`, so extra arguments are rejected.

**Reality check:** this checks calls, not the server. Implementing `call` needs a runtime schema per route (11.4); otherwise the response type is only a claim. In a real project, derive `Endpoints` from shared Zod schemas.

---

## Challenge 9: Exhaustive reducer 🔴

**Tests:** discriminated unions, `never`, immutable updates (3.11, 10.10)

**Task:** write a cart reducer for these actions. Adding a new action later must break the build until it is handled.

```ts
interface CartItem {
  productId: string;
  qty: number;
}
type CartState = { items: CartItem[] };

type CartAction =
  | { type: 'add'; item: CartItem }
  | { type: 'remove'; productId: string }
  | { type: 'setQty'; productId: string; qty: number }
  | { type: 'clear' };
```

**Solution**

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled action: ${JSON.stringify(x)}`);
}

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add': {
      const existing = state.items.find(
        (i) => i.productId === action.item.productId,
      );
      if (!existing) return { items: [...state.items, action.item] };
      return {
        items: state.items.map((i) =>
          i.productId === existing.productId
            ? { ...i, qty: i.qty + action.item.qty }
            : i,
        ),
      };
    }
    case 'remove':
      return {
        items: state.items.filter((i) => i.productId !== action.productId),
      };
    case 'setQty':
      return {
        items: state.items
          .map((i) =>
            i.productId === action.productId ? { ...i, qty: action.qty } : i,
          )
          .filter((i) => i.qty > 0),
      };
    case 'clear':
      return { items: [] };
    default:
      return assertNever(action);
  }
}
```

**Test it:** add `| { type: "applyCoupon"; code: string }` to the union. The `default` line now fails with `TS2345: Argument of type '{ type: "applyCoupon"; ... }' is not assignable to parameter of type 'never'`. That error is the feature.

**Common mistake:** `state.items.push(...)`. Mutating state means React will not re-render.

---

## Challenge 10: Narrow `unknown` without `as` 🔴

**Tests:** type guards, `in` narrowing, Zod (3.9, 8.7)

**Task:** write `parsePoint(input: unknown): Point | null`. First do it by hand, then with Zod.

```ts
interface Point {
  x: number;
  y: number;
}
parsePoint({ x: 1, y: 2 }); // { x: 1, y: 2 }
parsePoint({ x: '1', y: 2 }); // null
parsePoint(null); // null
```

**Solution A: a hand-written guard**

```ts
function isPoint(value: unknown): value is Point {
  return (
    typeof value === 'object' &&
    value !== null &&
    'x' in value &&
    typeof value.x === 'number' &&
    'y' in value &&
    typeof value.y === 'number'
  );
}

function parsePoint(input: unknown): Point | null {
  return isPoint(input) ? input : null;
}
```

After `"x" in value`, TS narrows `value` so that `value.x` is `unknown`. Then `typeof` narrows it to `number`. No cast is needed.

**Solution B: Zod**

```ts
import { z } from 'zod';

const PointSchema = z.object({ x: z.number(), y: z.number() });
type Point2 = z.infer<typeof PointSchema>;

function parsePoint2(input: unknown): Point2 | null {
  const result = PointSchema.safeParse(input);
  return result.success ? result.data : null;
}
```

**Which to choose:** Solution A is fine for two fields. For anything bigger, a hand-written guard becomes long and easy to get wrong, and TS never checks its body (3.9). Use a schema.

**Common mistakes:** `typeof value === "object"` is also true for `null` (check both), and forgetting that arrays are objects, so `[]` passes the first two checks.

---

## Scoring yourself

| Challenges solved without looking | Where you are                                     |
| --------------------------------- | ------------------------------------------------- |
| 1, 2, 5, 6, 9, 10                 | Job-ready core: this is what most interviews test |
| + 3, 7                            | Comfortable with generics and unions in real code |
| + 4, 8                            | Able to read and write advanced types             |

If the type-level ones (4, 8) felt hard, that is normal for a fresher. Revisit 5.10-5.16 and attempt them again in a few days. Do not skip 1, 2, 5, 6, 9 and 10.

**Interview tip:** in a live coding round, say your plan first ("I'll make this generic so the return type follows the input"), write the signature before the body, and use hover to check your types. Interviewers care more about your reasoning than about reproducing a perfect solution from memory.

---

### 📌 Key Takeaways

- Most solutions are a **signature problem**: link the types with generics (`K extends keyof T`, `T` in and out) and the body follows.
- Return honest types: `Map.get` and `Partial<Record<...>>` admit missing keys; `Record<K, V>` does not.
- Discriminated unions plus `assertNever` turn "forgot a case" into a compile error.
- Fix legacy-style code at its boundaries: annotate parameters, validate external data with Zod, and avoid `as any`.
- Tuple-typed rest parameters (`...args: BodyArgs<K>`) let one function have route-dependent arguments.
- A type guard's body is unchecked by TS. Keep guards short, or use a schema.

✅ Part 13c complete.

---

# Part 13d: Portfolio Projects, Cheat Sheet & Job-Ready Checklist

This is the final Part. It has three sections: **3 portfolio projects**, a **one-page cheat sheet**, and the **Skills Checklist**.

---

## 13.4 Three portfolio-worthy mini projects 🔴 MUST KNOW

**Why these three:** each one proves a different skill to a hiring manager. A recruiter will open your GitHub for about two minutes, so each repo needs a clear README, a live demo link if possible, and a short "what I learned about TypeScript" section.

**General rules for all three**

- `"strict": true` and `noUncheckedIndexedAccess` on, **zero** `any`, and every `as` or `!` has a comment.
- `tsc --noEmit`, ESLint and tests run in GitHub Actions on every push (9.17).
- Validate all outside data with Zod (8.7). A `.env.example`, and no secrets committed.
- README sections: what it does, tech stack, how to run it, a type-safety diagram (see 11.6), and known limitations.
- Build it in vertical slices (one feature through every layer), not layer by layer (11.6).

---

### Project 1: Task Tracker (full stack, REST + shared types) 🔴

**What it proves:** you can build and type a complete full stack feature, which is exactly what a fresher interview asks about.

**Stack:** npm workspaces monorepo (`shared`, `api`, `web`), Express 5, Prisma + PostgreSQL (or Mongoose + MongoDB), React + Vite, TanStack Query, React Hook Form + Zod.

**Feature list**

1. Sign up and log in (hashed passwords, JWT, validated payload).
2. CRUD for tasks with title, description, status (`"todo" | "doing" | "done"`), due date, and priority.
3. Filter by status, search by title, sort, and paginate (`page`, `limit`, `total` in the response meta).
4. Users can only see and change **their own** tasks (ownership check in the repository query).
5. Optimistic update when toggling status (TanStack Query `onMutate` with rollback).
6. Centralized error handler with typed `AppError` classes and a consistent `ApiResponse<T>` envelope.
7. Login and task forms share their Zod schemas with the backend.

**TypeScript concepts used**

| Concept                                  | Where               | Part         |
| ---------------------------------------- | ------------------- | ------------ |
| Shared Zod schemas, `z.infer`            | `shared` package    | 8.7, 11.2    |
| Discriminated union for `ApiResponse<T>` | Envelope and client | 3.7, 9.14    |
| Utility types (`Partial<Omit<...>>`)     | Update DTOs         | 5.3          |
| Module augmentation                      | `req.user`          | 7.7          |
| Typed middleware and error handler       | Express             | 9.6, 9.13    |
| DB row → DTO mapper                      | Repository boundary | 11.3         |
| Typed API client                         | `web/src/api`       | 11.4         |
| Typed hooks, reducer for filter state    | React               | 10.10, 10.15 |

**Stretch goals:** refresh tokens in `httpOnly` cookies; Vitest tests for services and `supertest` for routes; Dockerfile; deploy API and web.

**Interview talking point:** "I changed `done` to `status` in the shared schema and the compiler listed every file to fix in both apps" (11.6, Step 6).

---

### Project 2: Typed Component Library + Storybook-style Demo (frontend depth) 🟡

**What it proves:** you understand React + TS beyond basic props: generics, polymorphism, refs, and accessible, reusable components.

**Stack:** Vite + React + TS, CSS Modules or Tailwind, Vitest + React Testing Library, optionally Storybook.

**Feature list**

1. `Button` extending `ComponentPropsWithoutRef<"button">` with `variant` and `size` unions and a `loading` state.
2. `Input` and `Select<TValue>` (generic) with ref forwarding (`forwardRef`, or the `ref` prop in React 19).
3. `Table<T>` with typed columns, sortable by `keyof T`, and a `getKey` prop.
4. `Modal` with `children: ReactNode`, focus handling and `Escape` to close.
5. `Tabs` or `Accordion` built with typed context (`Value | null` plus a throwing hook).
6. A `Toast` system with a `useToast` hook and a discriminated-union reducer.
7. A docs page that shows each component with its props table.
8. Tests that assert behavior, plus `@ts-expect-error` type tests for invalid prop combinations.

**TypeScript concepts used**

| Concept                                    | Where                   | Part             |
| ------------------------------------------ | ----------------------- | ---------------- |
| `ComponentProps`, `Omit` for collisions    | Wrappers                | 10.7             |
| Generic components (function declarations) | `Table<T>`, `Select<T>` | 4.7, 10.16       |
| Typed context with a safe hook             | Tabs, Toast             | 10.12            |
| Discriminated-union props                  | Link-or-button, alerts  | 10.5             |
| `useReducer` with `assertNever`            | Toasts                  | 10.10            |
| `keyof T` for sortable columns             | Table                   | 4.4              |
| Type-level tests (`Expect<Equal<...>>`)    | Test files              | 13c, Challenge 4 |

**Stretch goals:** a polymorphic `as` prop (hard to type, a good deep dive); publish to npm with `declaration: true` and an `exports` map (7.4, 11.2); accessibility audit.

**Interview talking point:** "Why `Table<T>` cannot be `React.FC`, and what `forwardRef` does to generics" (10.16).

---

### Project 3: Typed Webhook & Job Processor (backend depth) 🟡

**What it proves:** you can handle untrusted input, errors, async flows and a clean architecture on the backend, which is what separates strong candidates.

**Stack:** Node 20, Express 5 (or Fastify), Zod, Prisma or SQLite, Vitest. An in-memory queue is fine; BullMQ + Redis is a stretch.

**Feature list**

1. `POST /webhooks/:provider` accepts events from simulated providers (payments, git pushes). Each provider has its **own** Zod schema.
2. Verify an HMAC signature header before parsing the body (use the raw body; a classic gotcha).
3. A `Result<T, E>` type for expected failures (invalid signature, unknown provider) and `AppError`s for unexpected ones (8.4, 8.5).
4. A typed event bus: `TypedEmitter<Events>` from Challenge 3 dispatching `"payment.succeeded"`, `"push.received"`.
5. A job queue with retry and backoff (Challenge 6), a dead-letter list, and idempotency keys so the same event is never processed twice.
6. `GET /events` with cursor pagination and typed query validation (remember Express 5's read-only `req.query`, 9.12).
7. Repository interfaces with an in-memory and a database implementation, wired in one composition root (9.8).
8. Structured logging and env validation at startup (9.7).

**TypeScript concepts used**

| Concept                                | Where          | Part        |
| -------------------------------------- | -------------- | ----------- |
| Discriminated union of provider events | Event handling | 3.7, 3.11   |
| `unknown` input narrowed with Zod      | Webhook bodies | 8.6-8.8     |
| `Result<T, E>` and custom errors       | Services       | 8.4, 8.5    |
| Generic repository + interfaces        | Persistence    | 4.8, 6.8    |
| Typed emitter with mapped types        | Event bus      | Challenge 3 |
| Async generics (`retry<T>`)            | Job runner     | Challenge 6 |
| `Mocked<Interface>`                    | Tests          | 9.15        |
| Config and env typing                  | Startup        | 9.7         |

**Stretch goals:** a worker process in a separate file; OpenAPI generation from the Zod schemas; load test with `autocannon`.

**Interview talking point:** "Why the body type is a claim until it is validated, and how idempotency keys protect against duplicate deliveries."

**Order to build them:** Project 1 first (it is what interviewers expect), then 3 or 2 depending on whether you are targeting backend or frontend roles. Two polished projects beat three half-finished ones.

---

## 13.5 One-page cheat sheet 🔴 MUST KNOW

Print this or keep it open in a tab.

### Setup and config

```bash
npm i -D typescript tsx @types/node       # per project
npx tsc --noEmit                          # type check (tsx / Vite do NOT check)
npx tsc --showConfig                      # final resolved config
```

`strict: true` + `noUncheckedIndexedAccess` · Node: `module`/`moduleResolution: NodeNext`, imports end in `.js` · Vite: `ESNext` + `bundler`, `jsx: react-jsx`, `noEmit`

### Types at a glance

```ts
let a: string, b: number, c: boolean, d: bigint, e: symbol;
let list: string[];  let ro: readonly number[];  let t: [string, number?];
let u: string | null;  let both: A & B;
type Mode = "dark" | "light";            // literal union (prefer over enum)
const ROLES = ["admin", "user"] as const;  type Role = (typeof ROLES)[number];
value as T        // unchecked claim        value!   // unchecked non-null
cfg satisfies T   // checked, keeps precise type
x?.y  x ?? fallback   // optional chaining · nullish coalescing
```

`unknown` (safe, must narrow) · `any` (off switch) · `never` (impossible) · `void` (no return value)

### Functions

```ts
function f(a: number, b?: string, c = 1, ...rest: boolean[]): void {}
type Fn = (x: number) => string;
async function load(): Promise<User> {}
function g<T>(x: T): T {
  return x;
} // generic
function h<T extends { id: number }>(x: T): T {
  return x;
} // constraint
function pad(this: Button, x: string) {} // `this` param (erased)
```

### Objects

```ts
interface User extends Base {
  readonly id: number;
  name: string;
  phone?: string;
}
type Dict = Record<string, number>;
interface Index {
  [key: string]: string;
}
type Keys = keyof User;
type N = User['name'];
type V = typeof value;
```

`interface`: merges, `extends` · `type`: unions, tuples, mapped/conditional

### Narrowing

```ts
typeof x === "string"   x instanceof Error   "key" in obj   Array.isArray(x)
if (x === undefined) {}  switch (s.status) { case "ok": ... default: assertNever(s) }
function isUser(v: unknown): v is User { /* checks */ }
function assert(c: unknown): asserts c { if (!c) throw new Error() }
```

### Utility types

|                                                            |                                              |
| ---------------------------------------------------------- | -------------------------------------------- |
| `Partial<T>` `Required<T>` `Readonly<T>`                   | all optional / required / locked (shallow)   |
| `Pick<T,K>` `Omit<T,K>`                                    | keep / drop keys (`Omit` doesn't check keys) |
| `Record<K,V>`                                              | object keyed by a union                      |
| `Exclude<U,X>` `Extract<U,X>` `NonNullable<T>`             | filter union members                         |
| `ReturnType<typeof f>` `Parameters<typeof f>`              | from functions (needs `typeof`)              |
| `Awaited<T>` · `NoInfer<T>` (5.4+)                         | unwrap promise · block inference             |
| `InstanceType<typeof C>` `ConstructorParameters<typeof C>` | from classes                                 |

### Advanced

```ts
type MyPartial<T> = { [K in keyof T]?: T[K] }; // mapped  (-? and -readonly remove)
type Unwrap<T> = T extends Promise<infer V> ? V : T; // conditional + infer
type Handler = `on${Capitalize<'click' | 'hover'>}`; // template literal
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
}; // remap
```

Conditionals **distribute** over unions on a naked `T`; wrap `[T]` to stop it.

### Classes

```ts
class Svc extends Base implements Contract {
  constructor(
    private readonly repo: Repo,
    public name: string,
  ) {
    super();
  } // parameter properties
  #secret = 1;
  protected x = 2;
  static count = 0;
  override run(): void {}
}
abstract class A {
  abstract go(): void;
}
```

### Modules and declarations

```ts
import type { User } from './user.js';
export type { User };
declare module '*.svg' {
  const url: string;
  export default url;
}
declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
} // file needs export {}
```

`@types/x` as devDependency (only if the lib ships no types) · `paths` aliases need a runtime/build tool too

### Errors and validation

```ts
try {
} catch (e) {
  if (e instanceof Error) e.message;
} // e: unknown
class AppError extends Error {
  constructor(
    msg: string,
    public readonly statusCode = 500,
  ) {
    super(msg);
  }
}
const Schema = z.object({ email: z.string().email() });
type T = z.infer<typeof Schema>;
const r = Schema.safeParse(input);
if (r.success) r.data; // use r.data, not the original
```

### Express

```ts
Request<Params, ResBody, ReqBody, ReqQuery>; // generics are claims: validate with Zod
const mw: RequestHandler = (req, res, next) => {
  next();
};
const eh: ErrorRequestHandler = (err: unknown, req, res, _next) => {}; // 4 params, registered last
```

Route params and query values are strings · `env = EnvSchema.parse(process.env)` at startup

### React

```tsx
function Card({ title, children }: { title: string; children: ReactNode }) {}
interface BtnProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'a' | 'b';
}
const [u, setU] = useState<User | null>(null);
const ref = useRef<HTMLInputElement>(null);
(e: ChangeEvent<HTMLInputElement>) => e.target.value; // currentTarget for the element
const Ctx = createContext<Value | null>(null); // + throwing useX() hook
function Table<T>(props: TableProps<T>) {} // generic = function declaration
useEffect(() => {
  void load();
  return () => cleanup();
}, [dep]); // callback cannot be async
```

`useQuery`: `data` is `T | undefined` until `isPending`/`isError` handled · `useForm<z.infer<typeof S>>` + `zodResolver`

### Full stack rules

Shared package = Zod schemas + DTO types only · DB row ≠ DTO ≠ UI state (convert with mappers) · JSON has no `Date` · validate on both sides · types never prove authorization

### Top error codes

`TS2322` wrong value · `TS2339` missing property · `TS2345` bad argument · `TS7006` implicit any · `TS18047/18048` possibly null/undefined · `TS2307` module not found · `TS7016` no declaration file · `TS2554` wrong arg count · `TS2741` missing property · `TS2353` excess property · `TS2769` no overload · `TS2578` unused `@ts-expect-error`

### Mindset

Boundaries annotated, inside inferred · `unknown` over `any` · narrow, don't assert · derive, don't copy · reason in a comment next to every `as`, `!` and `@ts-expect-error`

---

## 13.6 TypeScript Skills Checklist: are you job-ready? 🔴 MUST KNOW

Be honest. Tick an item only if you can do it **without notes** on a blank file. Anything unticked is a pointer back to the Part in brackets.

### Foundation (Parts 0-3)

- [ ] I can explain type erasure and why types do not exist at runtime (0.2)
- [ ] I can set up a project with `tsc`, `tsx` and a correct `tsconfig` for Node and for Vite (0.3-0.6)
- [ ] I know what `strict` enables and why `tsx`/Vite do not type check (0.5)
- [ ] I read an error by position, code, then the last nested line (0.7)
- [ ] I choose between `unknown`, `any`, `never` and `void` correctly (1.4)
- [ ] I replace enums with literal unions and `as const` arrays, and can still read enums (1.7, 1.10)
- [ ] I type functions, callbacks, async functions and overloads (Part 2)
- [ ] I can explain `interface` vs `type` and when each is needed (3.2)
- [ ] I model state as discriminated unions and end `switch` statements with `assertNever` (3.7, 3.11)
- [ ] I narrow with `typeof`, `in`, `instanceof` and write a type guard (3.8, 3.9)

### Intermediate (Parts 4-8)

- [ ] I write generics with constraints, and `getProp` with `keyof` from memory (4.3, 4.4)
- [ ] I can type an `ApiResponse<T>` and a generic repository (4.2, 4.8)
- [ ] I know when **not** to use a generic (4.9)
- [ ] I combine `Partial`, `Omit`, `Pick`, `Record`, `ReturnType` and `Awaited` (Part 5a)
- [ ] I can read a mapped, conditional or `infer` type and explain it step by step (5.10-5.16)
- [ ] I use access modifiers, `readonly`, parameter properties, `implements` and abstract classes (Part 6)
- [ ] I can read decorator-based code (NestJS) and know legacy vs standard decorators (6.10)
- [ ] I use `import type`, extend Express `Request`, and write a `.d.ts` for an untyped library (7.2, 7.6, 7.7)
- [ ] I fix path-alias and ESM `.js` extension problems (7.1, 7.9)
- [ ] I set up ESLint with `typescript-eslint` and know `no-floating-promises` (7.11)
- [ ] I narrow `catch (e)`, write custom errors and explain Result vs exceptions (8.3-8.5)
- [ ] I validate every external input with Zod and derive types with `z.infer` (8.6-8.8)

### Stack skills (Parts 9-11)

- [ ] I can build a typed Express API with routes, controllers, services and repositories (9.8)
- [ ] I type env vars by validating them at startup, not by casting (9.7)
- [ ] I type Mongoose or Prisma models and map rows to DTOs (9.9, 9.10, 11.3)
- [ ] I implement JWT auth with a validated payload and typed middleware (9.11)
- [ ] I write a central error handler with four parameters, registered last (9.13)
- [ ] I write unit and route tests with Vitest or Jest and typed mocks (9.15)
- [ ] I know how to build and deploy compiled output, not `tsx`, in production (9.17)
- [ ] I type React props, children, events, refs and `ComponentProps` wrappers (Part 10a)
- [ ] I type `useState`, `useReducer`, `useRef`, context and custom hooks correctly (Part 10b)
- [ ] I write generic components as function declarations (10.16)
- [ ] I use TanStack Query and React Hook Form + Zod with correct types (10.18, 10.19)
- [ ] I can type Redux Toolkit store, slices, hooks and thunks (10.20)
- [ ] I can set up a shared Zod contract between frontend and backend (11.2, 11.3)
- [ ] I can trace one value from database to UI and say what each layer's type guarantees, and what it does not (11.6)

### Legacy and professional (Parts 12-13)

- [ ] I can migrate a JS folder to TS gradually (12.1)
- [ ] I can remove `any` using the ladder and know the few acceptable uses (12.2)
- [ ] I use `@ts-expect-error` with a reason, not `@ts-ignore` (12.3)
- [ ] I can find my way around an unfamiliar codebase with Go to Definition and Find References (12.4)
- [ ] I debug a type error methodically, and check TS versions when behavior differs (12.5)
- [ ] I recognize the top 25 errors and fix them without `any` (12.6)
- [ ] I can answer Q1-Q38 of the interview list in about 30 seconds each (13b)
- [ ] I solved challenges 1, 2, 5, 6, 9 and 10 without looking (13c)
- [ ] I have **two** deployed portfolio projects with CI running `tsc --noEmit`, lint and tests (13.4)

### How to read your score

| Ticked    | What it means                                         | Next step                                                                      |
| --------- | ----------------------------------------------------- | ------------------------------------------------------------------------------ |
| **0-20**  | You are still learning the basics, and that is normal | Redo Parts 1-4 with the challenges; type code by hand every day                |
| **21-33** | Solid base, gaps in stack skills                      | Build Project 1; fill the unticked Parts as you hit them                       |
| **34-42** | Interview-ready for fresher full stack roles          | Apply now; polish the README and practice explaining your project              |
| **43+**   | Strong for a fresher                                  | Start Project 2 or 3, read the TS release notes, and contribute to open source |

Do not wait for 100%. Real jobs are learned on the job, and nobody ticks every box. What interviewers look for is clear fundamentals (`unknown` vs `any`, narrowing, generics, validation), one project you understand end to end, and honesty about what you do not know yet.

---

### 📌 Key Takeaways

- Build **Project 1** (full stack Task Tracker with shared Zod types) first; add a frontend-depth or backend-depth project based on the role you want.
- Every portfolio repo needs strict mode, zero unexplained `any` or `as`, CI running `tsc --noEmit`, lint and tests, and a README that explains your type-safety choices.
- The cheat sheet is a memory aid for syntax; the real skill is knowing what each construct guarantees and what it does not (erased types, unchecked assertions, unvalidated input).
- The checklist is a map, not an exam. Close gaps by building and reading real code, using the Part references.
- In interviews, lead with fundamentals, show one project you can explain from database to UI, and say "I'm not sure, I'd check the docs" when you are unsure.

✅ **Part 13d complete. The TypeScript notes are finished.**
