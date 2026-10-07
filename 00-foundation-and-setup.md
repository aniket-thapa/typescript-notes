# Part 0: Foundation & Setup

## 0.1 What TypeScript is and JS vs TS 🔴 MUST KNOW

**1. What it is**
TypeScript (TS) is JavaScript plus a **type system**. A type says what kind of value something holds (text, number, a `User` object). TS checks your code against these types _before_ it runs.

**2. Why it exists**
JavaScript finds mistakes at runtime, often in front of users: `undefined is not a function`, `Cannot read properties of undefined`. TS finds them while you type. It also powers autocomplete, safe renaming and "go to definition" in VS Code. In big codebases the types act as documentation that cannot go stale.

|                               | JavaScript                   | TypeScript                                    |
| ----------------------------- | ---------------------------- | --------------------------------------------- |
| Types                         | Checked at runtime (dynamic) | Checked at compile time (static)              |
| Errors found                  | When the code runs           | In the editor, before running                 |
| Runs in Node/browser directly | Yes                          | No, must be converted to JS first             |
| File extension                | `.js`, `.jsx`                | `.ts`, `.tsx`                                 |
| Relationship                  | Base language                | **Superset**: all valid JS is valid TS syntax |

**3. Code**

```ts
// JavaScript: no error until the code runs, then you silently get "510"
function add(a, b) {
  return a + b;
}
add(5, '10');

// TypeScript: error appears in the editor immediately
function addTs(a: number, b: number): number {
  return a + b;
}
addTs(5, '10'); // ❌ Argument of type 'string' is not assignable to parameter of type 'number'.
```

**5. Real-world use**
A backend returns `{ id, name, email }`. In React you write `user.fullName`. JS shows `undefined` on screen. TS refuses to compile because `fullName` does not exist on `User`.

**7. Interview tip**
_"Is TypeScript a different language from JavaScript?"_ It is a superset. It adds static types, and the compiler turns it into plain JS. Browsers and Node never see TS.

---

## 0.2 How compilation works and type erasure 🔴 MUST KNOW

**1. What it is**
The TypeScript compiler (`tsc`) does two separate jobs:

1. **Type checking**: finds type errors.
2. **Emitting**: converts `.ts` to `.js` by deleting the type syntax.

Deleting the types is called **type erasure**. Types exist _only at compile time_. At runtime they are gone.

**2. Why it matters**
Many beginner bugs come from forgetting this. You cannot do `if (x instanceof User)` where `User` is an interface, and you cannot check a type at runtime. Part 8 covers the fix (Zod).

**3. Code**

```ts
// input: user.ts
interface User {
  id: number;
  name: string;
}

function greet(user: User): string {
  return `Hello ${user.name}`;
}
```

```js
// output: user.js (interface and annotations are completely removed)
function greet(user) {
  return `Hello ${user.name}`;
}
```

The pipeline: `.ts` → type check → erase types → `.js` → run with Node or the browser.

**4. ❌ Wrong / ✅ Right**

```ts
interface User {
  id: number;
}

// ❌ Wrong: 'User' only exists as a type, not a runtime value
// if (data instanceof User) {}
// Error TS2693: 'User' only refers to a type, but is being used as a value here.

// ✅ Right: check the actual runtime shape
function isUser(data: unknown): data is User {
  return typeof data === 'object' && data !== null && 'id' in data;
}
```

(Type guards are covered fully in Part 3.)

**5. Real-world use**
`res.json()` in the browser returns data typed by _you_. TS cannot verify the server really sent that shape. If the API changes, TS stays silent and your app breaks at runtime.

**6. Common mistakes**
Believing "my code has no TS errors, so the API data is safe." Types describe what you _expect_, not what actually arrives.

**7. Interview tip**
_"What is type erasure?"_ Types are removed during compilation, so there is zero runtime cost and no runtime type information.

---

## 0.3 Installing and running TypeScript 🔴 MUST KNOW

**1. What it is**
You install TS **per project** as a dev dependency, not globally. Then you choose how to run code:

| Tool            | What it does                       | Type checks?                  | Use for                                      |
| --------------- | ---------------------------------- | ----------------------------- | -------------------------------------------- |
| `tsc`           | Compiles `.ts` to `.js`            | **Yes**                       | Build, CI, `--noEmit` checks                 |
| `tsx`           | Runs `.ts` directly (uses esbuild) | **No**                        | Fast local dev (recommended)                 |
| `ts-node`       | Runs `.ts` directly                | Yes (unless `transpile-only`) | Older projects, you'll see it in legacy code |
| Vite / bundlers | Transpile TS for the browser       | **No**                        | React frontend                               |

**Key point:** `tsx` and Vite strip types but do **not** check them. Always run `tsc --noEmit` separately (in a script, in CI, or via your editor).

**2. Why per-project**
Different projects need different TS versions. A global install causes "works on my machine" problems.

**3. Code**

```bash
mkdir ts-notes && cd ts-notes
npm init -y
npm install -D typescript tsx @types/node
npx tsc --init          # creates tsconfig.json (contents vary by TS version)
```

```ts
// src/index.ts
const message: string = 'Hello TypeScript';
console.log(message);
```

```bash
npx tsx src/index.ts            # run directly
npx tsc --noEmit                # type check only, no files written
npx tsc                         # compile to outDir (set in tsconfig)
node dist/index.js              # run the compiled output
```

```json
// package.json scripts (typical)
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "typecheck": "tsc --noEmit",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

**5. Real-world use**
Production flow: build once with `tsc`, deploy `dist/`, run with plain `node`. Never ship `tsx` or `ts-node` as your production runner.

**6. Common mistakes & errors**

- `tsc: command not found` → use `npx tsc` or an npm script.
- `error TS5083: Cannot read file '.../tsconfig.json'` → you ran `tsc` outside the project folder.
- Code runs fine with `tsx` but the build fails → expected. `tsx` skipped type checking. Run `npm run typecheck`.

**7. Interview tip**
_"Does `ts-node`/`tsx`/Vite catch type errors?"_ `tsx` and Vite do not. Only `tsc` (or your editor) type checks.

---

## 0.4 `tsconfig.json` explained 🔴 MUST KNOW

**1. What it is**
`tsconfig.json` tells the compiler which files to include, how strict to be, and what JS to output. It also drives VS Code's error checking.

**2. Why it exists**
Without it, `tsc` uses loose defaults. The config is the single most important file in a TS project, and legacy codebases are often hard to read because of it.

**3. Options, one by one**

| Option                     | What it does                                                                  | Typical value                           |
| -------------------------- | ----------------------------------------------------------------------------- | --------------------------------------- |
| `target`                   | Which JS version the output uses. Newer = less conversion.                    | `"ES2022"` (Node 20+)                   |
| `module`                   | Module system of the output.                                                  | `"NodeNext"` (Node), `"ESNext"` (Vite)  |
| `moduleResolution`         | How TS finds the file behind `import ... from "x"`.                           | `"NodeNext"` (Node), `"bundler"` (Vite) |
| `strict`                   | Turns on all strict checks (see 0.5).                                         | `true`                                  |
| `outDir`                   | Folder for compiled `.js`.                                                    | `"dist"`                                |
| `rootDir`                  | Root of your source, so `src/` structure is kept in `dist/`.                  | `"src"`                                 |
| `baseUrl`                  | Base folder for non-relative imports. Optional in modern TS.                  | `"."`                                   |
| `paths`                    | Import aliases like `@/utils/x`.                                              | `{ "@/*": ["src/*"] }`                  |
| `esModuleInterop`          | Allows `import express from "express"` for CommonJS packages.                 | `true`                                  |
| `skipLibCheck`             | Skips type checking of `.d.ts` files (faster, avoids errors in node_modules). | `true`                                  |
| `resolveJsonModule`        | Lets you `import data from "./data.json"` with types.                         | `true`                                  |
| `noUncheckedIndexedAccess` | `arr[0]` and `obj[key]` become `T \| undefined`. Not in `strict`.             | `true`                                  |
| `include`                  | Files to compile (globs).                                                     | `["src"]`                               |
| `exclude`                  | Files to skip.                                                                | `["node_modules", "dist"]`              |

Related options you will see often: `jsx` (`"react-jsx"` for React), `noEmit` (check only), `sourceMap` (debug TS in Node), `lib` (built-in type libraries, e.g. `"DOM"` for browsers), `isolatedModules`, `forceConsistentCasingInFileNames`.

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Wrong: alias in tsconfig only
// "paths": { "@/*": ["src/*"] }
import { db } from '@/db';
// tsc compiles happily, but Node crashes: Cannot find module '@/db'

// ✅ Right: tsc does NOT rewrite aliases in output.
// Use a runtime that understands them: Vite (vite-tsconfig-paths), tsx (reads paths),
// or a tool like tsc-alias for tsc builds.
```

`paths` only teaches the _type checker_ about aliases. A bundler or extra tool must teach the _runtime_.

**5. Real-world use**
`rootDir: "src"` + `outDir: "dist"` gives `dist/index.js` instead of `dist/src/index.js`. Without `rootDir`, a stray `.ts` file outside `src` can change your output folder layout.

**6. Common mistakes & errors**

- `error TS6059: File is not under 'rootDir'` → a file in `include` is outside `rootDir`. Fix `include` or `rootDir`.
- `error TS1259: Module can only be default-imported using the 'esModuleInterop' flag` → turn on `esModuleInterop`.
- `error TS2307: Cannot find module './data.json'` → enable `resolveJsonModule`.
- With `module: "NodeNext"`, if `package.json` has `"type": "module"`, relative imports need the `.js` extension (`import x from "./x.js"`), even though the file is `x.ts`. Without `"type": "module"` the package is treated as CommonJS and extensionless imports work. Part 7 explains this in depth.

**7. Interview tip**
_"What does `skipLibCheck` do and is it safe?"_ It skips checking `.d.ts` files. It is safe for app code and standard practice. Your own `.ts` files are still fully checked.

---

## 0.5 Strict mode and each strict flag 🔴 MUST KNOW

**1. What it is**
`"strict": true` is a switch that enables a group of checks. Always start new projects with it on.

**2. Why it exists**
Without strict, TS lets many dangerous things through (implicit `any`, unchecked `null`). Strict mode is where most of TS's value comes from. Interviewers expect you to use it.

**3. The flags inside `strict`**

| Flag                           | What it catches                                       | Typical error                                         |
| ------------------------------ | ----------------------------------------------------- | ----------------------------------------------------- |
| `noImplicitAny`                | Variables/params with no type that TS can't infer     | `TS7006: Parameter 'x' implicitly has an 'any' type.` |
| `strictNullChecks`             | `null`/`undefined` used as if they were real values   | `TS18047: 'user' is possibly 'null'.`                 |
| `strictFunctionTypes`          | Unsafe function parameter assignments                 | `TS2322` on callback types                            |
| `strictBindCallApply`          | Wrong args to `.bind/.call/.apply`                    | `TS2345`                                              |
| `strictPropertyInitialization` | Class property never assigned                         | `TS2564: Property 'name' has no initializer...`       |
| `noImplicitThis`               | `this` with unknown type                              | `TS2683: 'this' implicitly has type 'any'`            |
| `useUnknownInCatchVariables`   | `catch (e)` is `unknown`, not `any`                   | `TS18046: 'e' is of type 'unknown'.`                  |
| `alwaysStrict`                 | Emits `"use strict"` in JS                            | n/a                                                   |
| `strictBuiltinIteratorReturn`  | More accurate iterator return types (added in TS 5.6) | rare                                                  |

_Version note:_ the exact list inside `strict` grows with TS versions, and newer major versions may change defaults. Always set `strict` explicitly and check release notes when upgrading.

**3. Code**

```ts
// With strict: true
function getLength(text) {
  // ❌ TS7006: Parameter 'text' implicitly has an 'any' type.
  return text.length;
}

function findUser(id: number): { name: string } | null {
  return id === 1 ? { name: 'Asha' } : null;
}

const user = findUser(2);
console.log(user.name); // ❌ TS18047: 'user' is possibly 'null'.
console.log(user?.name); // ✅ optional chaining
if (user) console.log(user.name); // ✅ narrowing
```

**5. Real-world use**
`strictNullChecks` alone prevents the most common production crash: `Cannot read properties of null (reading 'name')`. This is the exact bug it was built for.

**6. Common mistakes**
Turning `strict` off to "make errors go away." This hides the problems and makes later migration painful.

**7. Interview tip**
_"What does `strict: true` enable?"_ Name `noImplicitAny` and `strictNullChecks` first. Those are the two that matter most.

### Extra strictness (not part of `strict`)

`noUncheckedIndexedAccess` is the one worth enabling:

```ts
const colors: string[] = ['red', 'green'];
const first = colors[0]; // type: string | undefined (with the flag)
console.log(first.toUpperCase()); // ❌ TS18048: 'first' is possibly 'undefined'.
console.log(first?.toUpperCase()); // ✅
```

It is a matter of team opinion. It adds safety but also more `undefined` checks. For a fresher project, enable it to build good habits.

---

## 0.6 Recommended `tsconfig` for Node and React (Vite) 🔴 MUST KNOW

**Node + Express backend**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true,
    "sourceMap": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

**React + Vite frontend (simplified)**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "noEmit": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

Notes:

- `noEmit: true` because Vite does the building; TS only checks.
- `lib` includes `DOM` because the frontend uses `document`, `window`, etc. The Node config omits it on purpose, so using `document` in backend code is an error.
- Current Vite templates split config into `tsconfig.json` (with `references`), `tsconfig.app.json` and `tsconfig.node.json`. The settings above are the same ideas in one file. Exact template contents change between Vite versions, so read the generated files.

**6. Common mistake**
Copying a Vite config into a Node project. `moduleResolution: "bundler"` assumes a bundler exists. Plain Node needs `NodeNext`.

---

## 0.7 How to read compiler errors 🔴 MUST KNOW

**1. What it is**
A TS error has three parts: **file and position**, **error code** (`TS2322`), and **message**. Long messages are nested: read the **last line first**, because it holds the actual mismatch.

**3. Code**

```ts
interface Product {
  id: number;
  name: string;
  price: number;
}

const item: Product = {
  id: 1,
  name: 'Keyboard',
  price: '999',
};
```

```
src/index.ts:8:3 - error TS2322: Type 'string' is not assignable to type 'number'.

8   price: "999",
    ~~~~~
  src/index.ts:4:3
    4   price: number;
        ~~~~~
    The expected type comes from property 'price' which is declared here on type 'Product'
```

**Reading steps**

1. Find the line and column. The squiggle marks the exact spot.
2. Read the message: "Type A is not assignable to type B" means you gave **A** where **B** was expected.
3. Follow the secondary location (`4:3`). It shows _where the expected type was declared_.
4. Decide: is the value wrong, or the type definition wrong?
5. For nested errors, scroll to the last indented line.

**Most common message shapes**

| Message                                                                    | Meaning                                                |
| -------------------------------------------------------------------------- | ------------------------------------------------------ |
| `Type 'X' is not assignable to type 'Y'` (TS2322)                          | Wrong value for a declared type                        |
| `Property 'x' does not exist on type 'Y'` (TS2339)                         | Typo, or the type lacks it                             |
| `Argument of type 'X' is not assignable to parameter of type 'Y'` (TS2345) | Wrong argument in a call                               |
| `Cannot find name 'x'` (TS2304)                                            | Missing import, typo, or missing types (`@types/node`) |
| `'x' is possibly 'undefined'` (TS18048)                                    | Needs a null/undefined check                           |

A longer list with exact fixes is in Part 12.

**Tips**

- Hover over any variable in VS Code to see its inferred type. This is your best debugging tool.
- Install Error Lens so errors show inline.
- If an error looks absurdly long, fix the **first** error in the file. Later ones are often side effects.
- Search the code (`TS2322`), not your own message text; it gives better results.

**6. Common mistake**
Silencing an error with `any` or `as` before understanding it. Understand first, then decide.

---

### 📌 Key Takeaways

- TS is JavaScript plus compile-time types. Browsers and Node only run the emitted JS.
- Types are erased, so they cannot validate runtime data. Part 8 solves this with Zod.
- `tsx` and Vite run code without type checking. Run `tsc --noEmit` to check.
- Always use `"strict": true`; `strictNullChecks` and `noImplicitAny` are the key flags.
- Node projects use `NodeNext` module settings; Vite projects use `ESNext` + `bundler`.
- Read errors by position, code, then the last nested line; hover in VS Code to see types.

✅ Part 0 complete.
