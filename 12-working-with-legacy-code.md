# Part 12: Working with Existing / Legacy Code

**Assumptions:** TypeScript 5.x. Error codes and messages below are real, but wording can change slightly between TS versions. When a message looks different, search by the `TS####` code.

---

## 12.1 Migrating JavaScript to TypeScript step by step 🔴 MUST KNOW

**1. What it is**
A gradual process: TS and JS files live side by side, and you tighten the rules one step at a time. Never rewrite everything in one go.

**2. Why it exists**
A "big bang" migration freezes feature work and produces thousands of errors at once. Gradual migration keeps the app shipping.

**3. The steps**

```bash
npm install -D typescript @types/node
npx tsc --init
```

**Step 1: let TS see JS files, with loose rules**

```json
{
  "compilerOptions": {
    "allowJs": true, // .js files can be imported by .ts files, and compiled
    "checkJs": false, // do NOT report errors in .js yet
    "noEmit": true, // only check (the old build tool still builds)
    "strict": false,
    "skipLibCheck": true,
    "include": ["src"]
  }
}
```

**Step 2: opt in file by file** with a comment at the top of a `.js` file:

```js
// @ts-check
/** @param {number} price @param {number} qty @returns {number} */
function total(price, qty) {
  return price * qty;
}
total(5, '2'); // ❌ TS2345 now reported in this JS file, using JSDoc types
```

(`// @ts-nocheck` does the opposite: skip a file once `checkJs` is on.)

**Step 3: rename files to `.ts` / `.tsx`**, starting with leaves (files that import little: utils, constants, types), and moving toward the entry points. Fix errors file by file. Typical first fixes: type parameters, add `@types/*` packages (7.5), and write interfaces for the main data objects.

**Step 4: turn strictness on gradually**, one flag at a time, committing after each:

1. `noImplicitAny` (usually the biggest batch)
2. `strictNullChecks` (finds real bugs)
3. `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`
4. Finally replace the pieces with `"strict": true`, and consider `noUncheckedIndexedAccess`

**Step 5: block regressions.** Run `tsc --noEmit` in CI. For very large codebases, tools such as `typescript-strict-plugin` or per-folder tsconfigs let new code be strict while old code is excused. Tools also exist that auto-insert `@ts-expect-error` to get to zero errors fast (tell your team before you do this).

**Step 6: switch the build** from Babel/webpack-only to one that handles TS (`tsx`, Vite, `tsc`), and remove `allowJs` when no `.js` remains.

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Renaming 200 files, adding `any` everywhere to get green, and calling it migrated
// ✅ Rename a feature folder, type its boundaries (inputs/outputs) properly, leave a few
//    marked TODOs, then move on. Typed boundaries give the most value first.
```

**Common conversion errors**

- `TS7016: Could not find a declaration file for module 'x'` → install `@types/x` or declare it (7.6).
- `TS2339: Property 'foo' does not exist on type '{}'` → JS built objects incrementally (`const o = {}; o.foo = 1`). Declare the interface up front.
- `TS7053: Element implicitly has an 'any' type because expression of type 'string' can't be used to index type` → use `Record<string, T>` or `keyof` (3.4).
- `module.exports` / `require` → switch to `import`/`export` (7.1), or keep the CommonJS style only in files you have not migrated.

**5. Real-world use**
This is the most common "first task" for a TS developer at an existing company.

**7. Interview tip**
_"How would you migrate a JS codebase to TS?"_ `allowJs` + `checkJs` per file, rename leaf files first, type the boundaries, enable strict flags one at a time, and enforce with CI.

---

## 12.2 Replacing `any` safely 🔴 MUST KNOW

**1. What it is**
A repeatable way to remove `any` without breaking the code.

**3. The ladder** (try from the top)

| Situation                                      | Replace `any` with                                                                            |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Truly unknown external data                    | `unknown` + narrowing or a Zod schema                                                         |
| Object of known shape                          | An `interface` / `type` (or `z.infer`)                                                        |
| Object with arbitrary keys                     | `Record<string, T>`                                                                           |
| Function parameter that could be several types | A union, or a generic `T`                                                                     |
| Callback of any signature                      | `(...args: never[]) => unknown` for storing, or a generic `F extends (...args: any[]) => any` |
| Third-party untyped value                      | A small typed wrapper (7.6)                                                                   |
| Truly impossible to type (rare)                | A **commented** cast: `as X // reason`                                                        |

```ts
// BEFORE
function process(data: any) {
  return data.items.map((i: any) => i.price * i.qty);
}

// AFTER: step 1. Describe the shape you actually use
interface OrderItem {
  price: number;
  qty: number;
}
interface OrderData {
  items: OrderItem[];
}
function processTyped(data: OrderData): number[] {
  return data.items.map((i) => i.price * i.qty);
}

// AFTER: step 2. If `data` comes from outside, validate at the boundary
const OrderDataSchema = z.object({
  items: z.array(z.object({ price: z.number(), qty: z.number() })),
});
function processExternal(raw: unknown): number[] {
  return processTyped(OrderDataSchema.parse(raw));
}
```

**How to find the real shape:** hover at call sites, log a sample at runtime, read the API docs, or paste JSON into a "JSON to TypeScript" tool and then tidy it by hand.

**Tooling:** the ESLint rules `@typescript-eslint/no-explicit-any` and the `no-unsafe-*` family (`no-unsafe-assignment`, `no-unsafe-member-access`) expose where `any` leaks, including the implicit `any` from `JSON.parse` or `res.json()`.

**4. ❌ Wrong / ✅ Right**

```ts
const user = JSON.parse(text) as any as User; // ❌ double cast: any in disguise
const raw: unknown = JSON.parse(text); // ✅ then validate or narrow
```

`any` is **contagious**: anything derived from it becomes `any` too, and the errors vanish downstream. Remove `any` at the source (function boundaries), not at the leaves.

**Acceptable `any` uses (rare):** generic constraints that must match every function or constructor (`(...args: any[]) => any`, `Constructor` in mixins 6.11, `infer` patterns 5.12). Put a comment on each.

**7. Interview tip**
_"How do you remove `any` from a large codebase?"_ Start at function boundaries, use `unknown` plus validation for external data, define real interfaces, and enforce with lint rules.

---

## 12.3 `@ts-ignore` vs `@ts-expect-error` 🔴 MUST KNOW

**1. What it is**
Comments that silence the compiler for the **next line**.

|                                 | `// @ts-ignore`       | `// @ts-expect-error`                                     |
| ------------------------------- | --------------------- | --------------------------------------------------------- |
| Silences the next line's errors | Yes                   | Yes                                                       |
| If the line has **no error**    | Silent, stays forever | **Error**: `TS2578: Unused '@ts-expect-error' directive.` |
| Use for                         | Legacy code only      | The better choice for new code                            |

**3. Code**

```ts
// @ts-expect-error: legacy lib types are wrong, see issue #123. Remove when upgraded.
legacyLib.init({ mode: 'fast' });

// Typical legitimate use: testing that bad input is rejected
// @ts-expect-error: id must be a number
expect(() => getUser('abc')).toThrow();
```

The advantage: when someone fixes the real problem (upgrades the library), the directive itself becomes an error, so dead suppressions get cleaned up automatically.

**4. ❌ Wrong / ✅ Right**

```ts
// @ts-ignore
const total = price * qty; // ❌ no reason, hides future real bugs
// @ts-expect-error: third-party d.ts declares `mode` as required; PR open upstream
init({}); // ✅ reason + expectation
```

Suppressing hides **every** error on that line, not just the one you saw. Keep the suppressed expression tiny. Enforce reasons with the ESLint rule `@typescript-eslint/ban-ts-comment` (requires a description).

**5. Real-world use**
Counting them is a health metric (`grep -r "@ts-" src | wc -l`). A rising count means the codebase is decaying.

**7. Interview tip**
_"`@ts-ignore` vs `@ts-expect-error`?"_ The second fails when there is no error, so stale suppressions get caught.

---

## 12.4 Reading a large unfamiliar TS codebase 🔴 MUST KNOW

**1. What it is**
A method for building a mental map quickly, using the compiler as your guide.

**3. Where to start (in this order)**

1. **`package.json`**: scripts (how to run/build/test), dependencies (framework: Express? Nest? Next?), `"type"` (7.1).
2. **`tsconfig.json`** (and files it `extends`/`references`): `strict` level, `paths` aliases, `target`/`module`, `include`. Config explains many "weird" imports (0.4).
3. **Entry points:** `src/server.ts`, `src/main.ts`, `src/index.tsx`, `App.tsx`, route files.
4. **Types/contracts first:** `types/`, `schemas/`, `models/`, `*.d.ts`, `shared/`. Reading the data shapes tells you what the app is about.
5. **One feature, end to end:** pick a simple endpoint or page and follow it from route → controller → service → repository (Part 9), or page → hook → API client (Part 10/11).
6. **Tests:** they show intended behavior and example inputs.

**VS Code tools you must know**

| Action                    | Shortcut (Win/Linux · Mac)                        | Use for                                             |
| ------------------------- | ------------------------------------------------- | --------------------------------------------------- |
| Go to Definition          | `F12` · `F12`                                     | Jump to where something is declared                 |
| Go to Type Definition     | (right-click menu)                                | Jump to the **type** of a variable                  |
| Peek Definition           | `Alt+F12` · `⌥F12`                                | Look without leaving the file                       |
| Find All References       | `Shift+F12` · `⇧F12`                              | Who uses this? Impact of a change                   |
| Go to Implementation      | `Ctrl+F12` · `⌘F12`                               | From an interface to its classes                    |
| Rename Symbol             | `F2` · `F2`                                       | Safe rename across the project                      |
| Quick Fix / Refactor      | `Ctrl+.` · `⌘.`                                   | Auto-import, add missing property, extract function |
| Go to Symbol in Workspace | `Ctrl+T` · `⌘T`                                   | Search all types and functions                      |
| Call Hierarchy            | right-click → Show Call Hierarchy                 | Who calls this, and what does it call               |
| Restart TS Server         | Command Palette → "TypeScript: Restart TS Server" | Fix stale or wrong errors (7.7)                     |

**4. Tracing types**

```ts
const result = await orderService.checkout(cart); // hover `result` to see its inferred type
// Hover shows a huge expanded type? Use the Prettify trick (5.16), or:
type Result = Awaited<ReturnType<typeof orderService.checkout>>; // name it, then hover Result
```

- **"Where does this value come from?"** Ctrl/Cmd+click each identifier backwards.
- **"What can this be?"** Hover; if it is a union, check where each member is created.
- **"Is this the type I think it is?"** Add `const _check: number = value;` temporarily. The error message prints the real type. Delete it afterwards.
- **Types from libraries:** Go to Definition on a library function opens its `.d.ts`. Read the signature and its JSDoc comments there.
- **Generics hell:** break the type into named aliases and test with small samples (5.16).

**5. Useful diagnostics**

```bash
npx tsc --noEmit                        # all errors in the project
npx tsc --noEmit --listFiles            # which files are included (find files `include` missed)
npx tsc --showConfig                    # the final config after extends/defaults
npx tsc --noEmit --explainFiles         # WHY each file is included
npx tsc --noEmit --extendedDiagnostics  # where the compiler spends time (slow projects)
```

**6. Common mistakes**

- Reading files top to bottom like a book. Follow one flow instead.
- Changing code before running the tests and `tsc --noEmit` to know the baseline.
- Trusting comments over types. Types are checked; comments rot.

**7. Interview tip**
_"How do you approach an unfamiliar codebase?"_ Config and entry points first, then the types, then trace one feature with Go to Definition and Find References, and read the tests.

---

## 12.5 Debugging type errors 🔴 MUST KNOW

**1. What it is**
A routine for resolving errors without guessing.

**3. The routine**

1. **Fix the first error in the file first.** Later ones are often side effects.
2. **Read the last nested line** of a long message (0.7). It states the actual mismatch.
3. **Hover** over both sides. Is the inferred type what you assumed?
4. **Find where the type was decided**: follow it to its source (declaration, function return, library).
5. **Decide which side is wrong:** the value, the type, or your assumption.
6. **Narrow rather than cast:** add a check, a guard (3.9), or validation before reaching for `as`.
7. **Minimize:** copy the problem into a scratch file with a tiny version. If the small one works, the difference is your answer.

**Common patterns**

```ts
// Pattern A: "possibly undefined" after a check that "should" narrow
function f(items: Item[]) {
  let current: Item | undefined;
  items.forEach((i) => {
    current = i;
  });
  current.id;
  // ❌ TS18048 / "Property 'id' does not exist on type 'never'"
  // TS doesn't track assignments made inside callbacks. Use a for...of loop.
}

// Pattern B: narrowing lost in a closure
function g(user: User | null) {
  if (!user) return;
  setTimeout(() => user.name, 0); // ✅ OK: parameters that are never reassigned stay narrowed (TS 5.4+)
}
// With `let user`, or older TS, copy into a const first: const u = user;

// Pattern C: object literal widening
const config = { method: 'GET' };
request(config);
// ❌ TS2345: ... Type 'string' is not assignable to type '"GET" | "POST"'
const config2 = { method: 'GET' } as const; // ✅ or `satisfies RequestInit`
```

**Tools:** `// @ts-expect-error` (for testing), the TypeScript Playground (typescriptlang.org/play) to share and bisect a problem, and the "Show Compiler Output" setting there to see erased output.

**Version-skew warning:** if an error appears only on CI or one colleague's machine, compare TS versions. VS Code can use its own bundled TS instead of the workspace one. Command Palette → "TypeScript: Select TypeScript Version" → "Use Workspace Version".

**7. Interview tip**
_"How do you debug a confusing type error?"_ Fix the first error, read the last line, hover to see real types, trace the source, and reproduce it in a minimal example.

---

## 12.6 Top 25 compiler errors and exact fixes 🔴 MUST KNOW

| #   | Code and message                                                                                                 | Cause                                                                  | Fix                                                                                  |
| --- | ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| 1   | **TS2322** `Type 'X' is not assignable to type 'Y'.`                                                             | Wrong value for a declared type                                        | Fix the value, widen the type, or convert (`Number(x)`)                              |
| 2   | **TS2339** `Property 'x' does not exist on type 'Y'.`                                                            | Typo, wrong type, or unnarrowed union                                  | Check spelling, narrow the union (`in`, discriminant), update the interface          |
| 3   | **TS2345** `Argument of type 'X' is not assignable to parameter of type 'Y'.`                                    | Wrong argument in a call                                               | Fix the argument; read the last nested line                                          |
| 4   | **TS7006** `Parameter 'x' implicitly has an 'any' type.`                                                         | Unannotated parameter (`noImplicitAny`)                                | Add a type; or define the callback where its type is known                           |
| 5   | **TS7031** `Binding element 'x' implicitly has an 'any' type.`                                                   | Destructured parameter with no type                                    | Type the whole parameter: `({ a }: Props)`                                           |
| 6   | **TS18047** `'x' is possibly 'null'.`                                                                            | `strictNullChecks`                                                     | Check `if (x)`, `x?.`, `??`, or throw earlier                                        |
| 7   | **TS18048** `'x' is possibly 'undefined'.`                                                                       | Optional value, array access, query `data`                             | Same as above; check `noUncheckedIndexedAccess`                                      |
| 8   | **TS2532** `Object is possibly 'undefined'.`                                                                     | Older wording of 7 (TS < 4.9 era)                                      | Same fixes                                                                           |
| 9   | **TS2304** `Cannot find name 'x'.`                                                                               | Missing import, typo, or missing global types                          | Import it; install `@types/node` for `process`; check `lib` for `document`           |
| 10  | **TS2307** `Cannot find module 'x' or its corresponding type declarations.`                                      | Wrong path, missing `.js` (ESM), alias not configured, package missing | Fix the path/extension, `paths`, install the package/`@types` (7.1, 7.9)             |
| 11  | **TS7016** `Could not find a declaration file for module 'x'.`                                                   | JS package with no types                                               | `npm i -D @types/x` or `declare module "x"` (7.6)                                    |
| 12  | **TS2554** `Expected N arguments, but got M.`                                                                    | Wrong number of arguments                                              | Add or remove arguments; make parameters optional if truly optional                  |
| 13  | **TS2741** `Property 'x' is missing in type 'A' but required in type 'B'.`                                       | Object lacks a required field                                          | Add it; or make it optional; or use `Partial`/`Omit`                                 |
| 14  | **TS2353** `Object literal may only specify known properties, and 'x' does not exist in type 'Y'.`               | Excess property check (3.6)                                            | Remove the field, fix the typo, or add it to the type                                |
| 15  | **TS2769** `No overload matches this call.`                                                                      | No overload fits (React props, `fetch`, `Array.from`)                  | Read the **last** overload's reason; match the argument types                        |
| 16  | **TS2349** `This expression is not callable.`                                                                    | Calling a union of functions, or a non-function                        | Narrow, or annotate the tuple return of a hook (10.15)                               |
| 17  | **TS2531 / TS2533** `Object is possibly 'null'` variants                                                         | Older null wording                                                     | Same as 6                                                                            |
| 18  | **TS2564** `Property 'x' has no initializer and is not definitely assigned in the constructor.`                  | Class field not set                                                    | Initialize it, mark `x?:`/`\| undefined`, or `x!:` only if a framework sets it (6.1) |
| 19  | **TS2551** `Property 'x' does not exist on type 'Y'. Did you mean 'z'?`                                          | Typo                                                                   | Use the suggestion                                                                   |
| 20  | **TS2367** `This comparison appears to be unintentional because the types 'A' and 'B' have no overlap.`          | Comparing impossible values, a stale union                             | Fix the comparison; check if the union was narrowed already                          |
| 21  | **TS2740** `Type 'Promise<X>' is missing the following properties...`                                            | Forgot `await`                                                         | Add `await`                                                                          |
| 22  | **TS1259** `Module can only be default-imported using the 'esModuleInterop' flag.`                               | CJS default import                                                     | Enable `esModuleInterop`, or `import * as x`                                         |
| 23  | **TS2578** `Unused '@ts-expect-error' directive.`                                                                | The error is gone                                                      | Delete the directive (12.3)                                                          |
| 24  | **TS2589** `Type instantiation is excessively deep and possibly infinite.`                                       | Recursive or huge generic type                                         | Simplify, add a base case, annotate an intermediate type (5.15)                      |
| 25  | **TS2742** `The inferred type of 'x' cannot be named without a reference to '...'. This is likely not portable.` | Declaration emit needs a type from a nested package                    | Add an explicit type annotation or import that type directly                         |

**Others worth recognizing:** `TS2393 Duplicate function implementation`, `TS2300 Duplicate identifier` (two `type` declarations with one name, 3.2), `TS6133 'x' is declared but its value is never read` (`noUnusedLocals`), `TS1484` / `TS1205` (type-only import/export rules, 7.2), `TS2835` (missing `.js` extension in ESM, 7.1), `TS5083`/`TS6059` (tsconfig problems, 0.4), and `TS2786 'X' cannot be used as a JSX component` (duplicate `@types/react`, 10.23).

**Example of fixing several**

```ts
function loadUser(id) {
  // TS7006
  const raw = localStorage.getItem('user'); // string | null
  const user = JSON.parse(raw); // TS2345: 'string | null' not assignable to 'string'
  return user.name; // `any`: no error, but unsafe
}
// FIX
const UserSchema = z.object({ name: z.string() });
function loadUserFixed(): string | null {
  const raw = localStorage.getItem('user');
  if (raw === null) return null;
  const parsed = UserSchema.safeParse(JSON.parse(raw));
  return parsed.success ? parsed.data.name : null;
}
```

**Interview tip:** you will not be asked to memorize codes. Interviewers want to see that you can read the message, find the source, and fix it without reaching for `any`.

---

## 12.7 Version differences worth knowing (TS 4.x → 5.x) 🟡 GOOD TO KNOW

Old codebases mix several TS versions. Check `typescript` in `package.json` and the `npx tsc -v` output first. Dates below are approximate; verify against the release notes when it matters.

| Version | Feature                                                                                                                                    | What to know                                                       |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| 4.0     | Variadic tuple types, labeled tuples                                                                                                       | `type T = [...A, ...B]`                                            |
| 4.1     | Template literal types, key remapping (`as`)                                                                                               | 5.13, 5.14                                                         |
| 4.2     | Leading/middle rest elements in tuples                                                                                                     | Rare                                                               |
| 4.4     | `useUnknownInCatchVariables` in `strict`                                                                                                   | `catch (e)` is `unknown` (8.3). Old code relies on `any`.          |
| 4.5     | `Awaited<T>`                                                                                                                               | 5.7                                                                |
| 4.7     | `infer U extends X`, `moduleResolution: node16/nodenext`                                                                                   | 5.12, 7.1                                                          |
| 4.8     | Better narrowing, `{}` intersection behavior                                                                                               | Minor                                                              |
| 4.9     | **`satisfies`**, `in` narrowing for unlisted keys                                                                                          | 1.10                                                               |
| 5.0     | **Standard (TC39) decorators**, `const` type parameters, `moduleResolution: bundler`, `verbatimModuleSyntax`, all enums become union enums | Details below                                                      |
| 5.1     | Easier `undefined`-returning functions, unrelated getter/setter types                                                                      | 2.6, 6.5                                                           |
| 5.2     | `using` declarations (explicit resource management)                                                                                        | Rare in app code                                                   |
| 5.3     | Narrowing improvements, `import` attributes                                                                                                | Minor                                                              |
| 5.4     | **`NoInfer<T>`**, preserved narrowing in closures after last assignment                                                                    | 5.8, 12.5                                                          |
| 5.5     | **Inferred type predicates** (`.filter(x => x !== null)` narrows), regex syntax checking, `isolatedDeclarations`                           | 3.9                                                                |
| 5.6     | Stricter checks for always-true/false conditions on truthy/nullish checks, `strictBuiltinIteratorReturn`                                   | Can surface new errors on upgrade                                  |
| 5.7     | Checks for uninitialized variables in closures, `--rewriteRelativeImportExtensions`                                                        | Helps with `.ts` imports in Node                                   |
| 5.8     | `--erasableSyntaxOnly`, better checks on conditional-return expressions                                                                    | Bans enums/namespaces/parameter properties (1.7, 6.3, 7.3)         |
| 5.9+    | More Node/ESM config work (check the current release notes)                                                                                | I am less sure of the exact details; verify before relying on them |

**`const` type parameters (5.0)**

```ts
function route<const T extends readonly string[]>(paths: T): T {
  return paths;
}
const r = route(['/home', '/about']); // readonly ["/home", "/about"] (no `as const` at the call site)
```

**Decorators changed in 5.0** (6.10): standard decorators need no flag and work differently from `experimentalDecorators`. Frameworks like NestJS stay on the legacy kind, so never remove `experimentalDecorators` from such a project.

**Upgrade advice:** upgrade one minor version at a time, run `tsc --noEmit`, and fix new errors before moving on. A new TS version often reports **real bugs** that older versions missed (5.6's truthiness checks are an example). Also upgrade `typescript-eslint` and `@types/*` with it, since mismatches cause odd errors. If a project is stuck on old TS, check which language features your editor offers that its compiler does not support.

---

### 📌 Key Takeaways

- Migrate gradually: `allowJs`/`checkJs`, `// @ts-check` per file, rename leaf files first, enable strict flags one at a time, and enforce with CI.
- Remove `any` at function boundaries, use `unknown` plus validation for external data, and lint `no-explicit-any` and `no-unsafe-*`.
- Prefer `@ts-expect-error` with a reason over `@ts-ignore`; track suppression counts.
- Read unfamiliar code via config → entry points → types → one feature end to end, using Go to Definition, Find References and hover.
- Debug errors by fixing the first one, reading the last nested line, hovering for real types and reproducing minimally. Check TS versions when errors differ between machines.
- Know the 25 common error codes by pattern, and check the TS version before assuming a feature (`satisfies` 4.9, `NoInfer` 5.4, inferred predicates 5.5) exists.

✅ Part 12 complete.
