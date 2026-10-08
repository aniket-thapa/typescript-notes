# Part 7: Modules, Declaration Files & Config Ecosystem

## 7.1 ES modules vs CommonJS in TypeScript 🔴 MUST KNOW

**1. What it is**
Two ways JavaScript splits code into files:

|                   | ES Modules (ESM)                                    | CommonJS (CJS)                      |
| ----------------- | --------------------------------------------------- | ----------------------------------- |
| Syntax            | `import x from "./x"` / `export`                    | `require("./x")` / `module.exports` |
| Loaded            | Statically (at parse time)                          | Dynamically (at runtime)            |
| Default in        | Browsers, Vite, modern Node with `"type": "module"` | Older Node projects                 |
| Top-level `await` | Yes                                                 | No                                  |

**2. Why it matters**
You always **write** ESM syntax in TS. What matters is what the compiler **emits** and how Node **runs** it. Mixing them wrongly is the #1 source of "works in tsx, crashes after build" errors.

**3. Code**

```ts
// Always write this in TS, in both worlds
import express from 'express';
import { readFile } from 'node:fs/promises';
import { add } from './math.js'; // note the .js (see below)
export const PI = 3.14;
export default function main() {}
```

**How Node decides (with `module: "NodeNext"`):**

- `package.json` has `"type": "module"` → `.ts` files are treated as **ESM**.
- No `"type"` (or `"commonjs"`) → treated as **CJS**; TS emits `require`.
- `.mts` / `.cts` force ESM / CJS for a single file.

**The `.js` extension rule (ESM only):** in ESM, Node requires file extensions. TS does not rewrite import paths, so you write the **output** extension even though the file on disk is `.ts`:

```ts
import { add } from './math.js'; // ✅ file is math.ts; Node will see math.js
import { add } from './math';
// ❌ TS2835: Relative import paths need explicit file extensions in ECMAScript imports when '--moduleResolution' is 'node16' or 'nodenext'. Did you mean './math.js'?
```

In a CJS project (no `"type": "module"`), extensionless imports are fine.

**Which should a fresher choose?**

- Backend: either works. Many tutorials and older codebases use CJS; new projects increasingly use ESM. Team and library choice decides. CJS avoids the `.js` extension friction; ESM is the long-term direction.
- Frontend (Vite): always ESM, `moduleResolution: "bundler"` (extensions optional).

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Mixing styles in one TS file
const express = require('express');
// ❌ TS2580: Cannot find name 'require'. Do you need to install type definitions for node?
// (fix: install @types/node, but prefer import syntax anyway)

import express2 from 'express'; // ✅
```

```ts
// ❌ __dirname in ESM
console.log(__dirname);
// ❌ ReferenceError at runtime (ESM has no __dirname)
// ✅ ESM replacement
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';
const __dirname2 = dirname(fileURLToPath(import.meta.url));
// (newer Node versions also offer import.meta.dirname)
```

**5. Real-world use**
Error `ERR_REQUIRE_ESM` or `Cannot use import statement outside a module` at runtime means the output module format and the package's `"type"` disagree. Check `module` in tsconfig and `"type"` in `package.json` first.

**6. Common mistakes**

- Using Vite's config (`moduleResolution: "bundler"`) for a plain Node app (Part 0).
- `esModuleInterop: true` lets you write `import express from "express"` for CJS packages that use `module.exports = ...`. Without it: `TS1259`.

**7. Interview tip**
_"Why do imports need `.js` in a TS file?"_ TS does not rewrite paths; Node ESM needs the extension of the file that will actually exist at runtime.

---

## 7.2 `import type` and `export type` 🔴 MUST KNOW

**1. What it is**
Imports/exports that exist **only for the type system** and are fully erased from the output.

**2. Why it exists**
Types are erased (Part 0), but a regular `import { User } from "./user"` may or may not be removed depending on settings. `import type` makes the intent explicit and guarantees removal. It also avoids circular-import problems and works with tools that compile one file at a time (Vite, esbuild, Babel).

**3. Code**

```ts
import type { User } from './types.js'; // whole import is type-only
import { type Order, createOrder } from './orders.js'; // inline: Order is type-only, createOrder is real

export type { User };
export type { Order } from './orders.js';

const u: User = { id: 1, name: 'A' };
const x = new User();
// ❌ TS1361: 'User' cannot be used as a value because it was imported using 'import type'.
```

**Related flags:**

- `isolatedModules: true`: required for Vite/esbuild. Re-exporting a type without `type` is an error: `TS1205: Re-exporting a type when 'isolatedModules' is set requires using 'export type'.`
- `verbatimModuleSyntax: true` (TS 5.0+): imports without `type` are kept exactly as written, and type-only imports **must** use `type`. Default in many modern templates. Error: `TS1484: 'User' is a type and must be imported using a type-only import when 'verbatimModuleSyntax' is enabled.`
- `importsNotUsedAsValues` and `preserveValueImports` are older flags that `verbatimModuleSyntax` replaced. You will see them in legacy `tsconfig` files.

**4. ❌ Wrong / ✅ Right**

```ts
import { Request } from 'express'; // ❌ with verbatimModuleSyntax: TS1484 (Request is mostly a type)
import type { Request } from 'express'; // ✅
```

**5. Real-world use**
Almost every Express/React file starts with `import type { Request, Response } from "express"` or `import type { ReactNode } from "react"`.

**7. Interview tip**
_"What does `import type` do?"_ It imports only for type checking and is erased entirely.

---

## 7.3 Namespaces 🟢 LEGACY / READ-ONLY

**What it is**
An old TS feature (from before ES modules existed) that groups code under one name: `namespace Utils { export function f() {} }`. It generates a runtime object.

```ts
namespace Validation {
  export interface StringValidator {
    isValid(s: string): boolean;
  }
  export class EmailValidator implements StringValidator {
    isValid(s: string): boolean {
      return s.includes('@');
    }
  }
}
const v = new Validation.EmailValidator();
```

**Today:** use ES modules instead. You still meet namespaces in two places: older code, and `declare namespace` inside `.d.ts` files (7.4, 7.7). Like enums, `namespace` containing values is not "erasable syntax" and fails under `erasableSyntaxOnly` or Node type stripping.

---

## 7.4 Declaration files (`.d.ts`) and `declare` 🔴 MUST KNOW

**1. What it is**
A `.d.ts` file contains **only types**, no executable code. It describes the shape of JavaScript code that exists elsewhere. `declare` says "this exists at runtime; trust me, here is its type."

**2. Why it exists**
JS libraries have no types. A `.d.ts` file lets TS check your calls to them. Compiled TS libraries ship `.d.ts` files alongside their JS so you get types automatically.

**3. Code**

```ts
// globals.d.ts
declare const APP_VERSION: string; // a global injected by a bundler
declare function track(event: string): void; // a global function
declare interface Window {
  analytics: { track(e: string): void };
}

// Wildcard modules: import assets in Vite/webpack
declare module '*.svg' {
  const url: string;
  export default url;
}
declare module '*.css'; // any import of .css is allowed, typed as any
```

```ts
import logo from './logo.svg'; // ✅ string, thanks to the declaration above
```

Without the declaration:

```
error TS2307: Cannot find module './logo.svg' or its corresponding type declarations.
```

**Important rules**

- A `.d.ts` must be included by `include` in `tsconfig` (or imported/referenced) to be seen.
- A file that has **no top-level `import`/`export`** is a **global script**: its declarations are global. A file with any `import`/`export` is a **module**: declarations are local to it (this matters in 7.7).
- `declare` never emits code. If the value does not exist at runtime, you get a crash that TS cannot warn you about.
- Setting `"declaration": true` makes `tsc` generate `.d.ts` files for your own library code. Not needed for apps.

**4. ❌ Wrong / ✅ Right**

```ts
declare const config: { port: number }; // ❌ if config doesn't actually exist globally at runtime → ReferenceError
// ✅ Use `declare` only for things provided from outside TS (bundler defines, script tags, globals)
```

**5. Real-world use**
Typing `import.meta.env` in Vite (`vite-env.d.ts`), declaring image/CSS modules, and typing global variables set by a script tag.

**7. Interview tip**
_"What is a `.d.ts` file?"_ Type-only declarations that describe existing JavaScript, with no emitted code.

---

## 7.5 `@types/*` packages (DefinitelyTyped) 🔴 MUST KNOW

**1. What it is**
For popular JS libraries without bundled types, the community publishes types on npm under the `@types` scope, from the **DefinitelyTyped** repository.

**3. Code**

```bash
npm install express
npm install -D @types/express @types/node
```

```ts
import express from 'express';
// Without @types/express:
// ❌ TS7016: Could not find a declaration file for module 'express'. '.../express/index.js' implicitly has an 'any' type.
//    Try `npm i --save-dev @types/express` if it exists or add a new declaration (.d.ts) file containing `declare module 'express';`
```

**Rules of thumb**

- Install `@types/x` as a **dev dependency**; it is needed only at compile time.
- Many modern libraries (axios, zod, prisma, vite, react-query) **ship their own types**. Do not install `@types` for them.
- React is a special case: `react` has no types, so install `@types/react` and `@types/react-dom` (current Vite templates include them).
- Keep the `@types` major version close to the library version. A mismatch gives confusing errors.
- By default, TS includes **all** visible `@types/*` packages globally. The `"types": ["node"]` tsconfig option restricts this. If you set `types` and then globals like `describe` vanish (`TS2582: Cannot find name 'describe'`), add `"jest"` or `"vitest/globals"` to it.

**5. Real-world use**
`@types/node` gives you `process`, `Buffer`, `node:fs` and so on. `Cannot find name 'process'` (TS2580) nearly always means it is missing.

**6. Common mistake**
Installing `@types/express` but forgetting `@types/node` (or the reverse), and then hitting unrelated-looking errors. Install both.

---

## 7.6 Writing types for an untyped JS library 🟡 GOOD TO KNOW

**1. What it is**
When a package has no bundled types and no `@types/*`, you add your own declaration, from quick-and-dirty to proper.

**3. Code (three levels, pick the smallest that works)**

```ts
// Level 1: shut up the error (everything becomes `any`): src/types/untyped.d.ts
declare module 'legacy-lib';

// Level 2: type only what you use
declare module 'legacy-lib' {
  export function formatPrice(amount: number, currency?: string): string;
  export interface Options {
    locale: string;
  }
  export default function init(options: Options): void;
}

// Level 3: a global-style library
declare namespace LegacyLib {
  function run(): void;
}
```

```ts
import init, { formatPrice } from 'legacy-lib';
formatPrice('5');
// ❌ TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
```

**Where to put the file:** inside a folder covered by `include` (for example `src/types/`). You can also point `typeRoots` or `paths` to a custom folder, but that is rarely needed.

**4. ❌ Wrong / ✅ Right**

```ts
declare module 'legacy-lib'; // ⚠️ everything is `any`: no safety
// ✅ At least type the 2-3 functions you actually call, and wrap them:
export function formatPriceSafe(amount: number): string {
  return formatPrice(amount);
}
```

Wrapping an untyped library in one small typed module of your own keeps the `any` out of the rest of your code.

**5. Real-world use**
Old internal packages, small npm modules, and vendor SDKs. If you fix a popular library's types, contributing to DefinitelyTyped is a good portfolio item.

**6. Common mistake**
Typing it from guesswork. Read the library's source or docs, or runtime-check it with a log. Wrong types are worse than `any`, because they give false confidence.

---

## 7.7 Module augmentation and global augmentation 🔴 MUST KNOW

**1. What it is**
Using declaration merging (Part 3) to **add** to types that someone else defined.

- **Module augmentation:** add to a type exported by a module.
- **Global augmentation:** add to a type in the global scope (such as `Window` or `NodeJS.ProcessEnv`).

**2. Why it exists**
Libraries cannot know your app's custom fields (`req.user`, `process.env.JWT_SECRET`, `window.analytics`). Augmentation tells TS about them without editing `node_modules`.

**3. Code: the classic Express `req.user`**

```ts
// src/types/express.d.ts
import type { JwtPayload } from '../auth/jwt.js'; // this import makes the file a MODULE (see below)

declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
}
export {}; // alternative way to mark the file as a module, if you have no imports
```

```ts
// anywhere in your app
import type { Request, Response } from 'express';
function me(req: Request, res: Response) {
  res.json(req.user?.id); // ✅ typed, no cast
}
```

Why `declare global { namespace Express { ... } }`: `@types/express-serve-static-core` builds `Request` on top of a global `Express.Request` interface that is **meant** to be extended. Merging into it is the most reliable approach.

**Alternative: augment a module's interface directly**

```ts
declare module 'express-serve-static-core' {
  interface Request {
    requestId: string;
  }
}
```

This also works, but it needs the file to be a module and the module name to match exactly. Either style is common; the `Express` namespace one is the more widely used.

**Typing environment variables**

```ts
// src/types/env.d.ts
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: 'development' | 'production' | 'test';
      JWT_SECRET: string;
      PORT?: string;
    }
  }
}
export {};
```

⚠️ This is a **promise, not a check**: nothing guarantees `JWT_SECRET` is actually set at runtime. Part 9 shows validating env vars with Zod, which is the safer approach. Many teams use Zod and skip this augmentation.

**4. ❌ Wrong / ✅ Right: the file isn't picked up**

```ts
// ❌ Symptom: Property 'user' does not exist on type 'Request'. (TS2339)
// Common causes:
// 1. The .d.ts is not covered by tsconfig "include" (e.g. it lives outside src/).
// 2. Runtime tool (ts-node) doesn't load it. Fix: "files": true in ts-node options, or use tsx.
// 3. `declare global` used in a file with no import/export:
//    ❌ TS2669: Augmentations for the global scope can only be directly nested in external modules or ambient module declarations.
//    ✅ add `export {}` or an import.
// 4. Restart the TS Server in VS Code (Cmd/Ctrl+Shift+P → "TypeScript: Restart TS Server").
```

Also: `user?: JwtPayload` is optional, honest, because routes without auth middleware have no user. Some teams make it required and use `req.user!`; the optional version is safer.

**5. Real-world use**
`req.user`, `req.requestId`, typed `process.env`, `window.dataLayer`, and extending a UI library's theme (`declare module "@mui/material/styles"`).

**6. Common mistake**
Augmenting inside a file that is not a module, or in a place `include` does not cover. If an augmentation "does nothing", check these two first.

**7. Interview tip**
_"How do you add `req.user` to Express's `Request`?"_ A `.d.ts` file with `declare global { namespace Express { interface Request { user?: ... } } }`, included by tsconfig.

---

## 7.8 Triple-slash directives 🟢 LEGACY / READ-ONLY

**What it is**
Special comments at the top of a file that tell the compiler about dependencies: `/// <reference types="node" />`, `/// <reference path="./globals.d.ts" />`, `/// <reference lib="es2022" />`.

You will see them in:

- Vite's generated `vite-env.d.ts`: `/// <reference types="vite/client" />` (this is what types `import.meta.env`).
- Old projects and some `.d.ts` files.

**Today:** use `import`, `include` and the `types`/`lib` options instead. Do not write new ones, except in that Vite file, which you should leave alone. They are only valid at the very top of a file.

---

## 7.9 Path aliases 🔴 MUST KNOW

**1. What it is**
Short import names such as `@/services/user` instead of `../../../services/user`.

**3. Code**

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

```ts
import { UserService } from '@/services/user.service';
```

**Key rule (repeating Part 0 because it bites everyone):** `paths` only helps the **type checker**. Something must also teach the **runtime** about the alias:

| Environment                | How to make aliases work                                               |
| -------------------------- | ---------------------------------------------------------------------- |
| Vite / Vitest              | `vite-tsconfig-paths` plugin or `resolve.alias` in `vite.config.ts`    |
| `tsx`                      | Reads `paths` from tsconfig automatically                              |
| `tsc` build, run with Node | Rewrite after build with `tsc-alias`, or use a bundler (esbuild, tsup) |
| Jest                       | `moduleNameMapper` (or `pathsToModuleNameMapper` from ts-jest)         |

**4. ❌ Wrong / ✅ Right**

```
// ❌ Runtime: Error: Cannot find module '@/services/user.service'
// tsc compiled fine; Node cannot resolve the alias in dist/*.js
// ✅ Run tsc-alias after tsc, or avoid aliases in backend code and use relative imports
```

`baseUrl` is optional in modern TS (4.1+ allows `paths` alone, with paths relative to the tsconfig). Using `baseUrl` also makes bare imports like `import x from "services/x"` resolve, which can clash with real package names. This is why some teams skip it.

**6. Common mistake**
Adding aliases to a backend and forgetting the post-build step. It is the most common "works in dev, crashes in production" TS bug. Honestly, relative imports are the safest option for a small backend.

---

## 7.10 Project references 🟡 GOOD TO KNOW

**1. What it is**
A way to split one TS project into several smaller ones that depend on each other (`"references"`), compiled with `tsc --build` (`tsc -b`). Each referenced project must set `"composite": true`.

**2. Why it exists**
Faster incremental builds and enforced boundaries in monorepos (`shared`, `server`, `web`). It is also why Vite templates have `tsconfig.app.json` and `tsconfig.node.json`.

**3. Code**

```json
// packages/server/tsconfig.json
{
  "compilerOptions": { "composite": true, "outDir": "dist", "rootDir": "src" },
  "references": [{ "path": "../shared" }]
}
```

```bash
tsc -b                # build this project and the ones it references, in order
```

**6. Common mistakes & errors**

- `TS6306: Referenced project '.../shared' must have setting "composite": true.`
- `TS6307: File '...' is not listed within the file list of project. Projects must list all files or use an 'include' pattern.`
- With plain `tsc` (not `-b`), references are ignored and you may get stale output.

Treat this as read-and-copy-config knowledge until you work on a monorepo (Part 11).

---

## 7.11 ESLint + typescript-eslint + Prettier 🔴 MUST KNOW

**1. What it is**

- **TypeScript** checks types.
- **ESLint** finds code-quality problems and risky patterns. `typescript-eslint` teaches it TS syntax and adds TS-specific rules, including **type-aware** ones that use the type checker.
- **Prettier** only formats code (spacing, quotes, line width). Do not use ESLint for formatting.

**2. Why it exists**
The compiler allows many bad habits. For example, it does not complain about an un-awaited promise. ESLint catches those.

**3. Setup (current flat-config style; ESLint 9+)**

```bash
npm install -D eslint @eslint/js typescript-eslint prettier eslint-config-prettier
```

```js
// eslint.config.mjs
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import prettier from 'eslint-config-prettier';

export default tseslint.config(
  { ignores: ['dist', 'node_modules'] },
  eslint.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked, // type-aware rules
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/consistent-type-imports': 'error',
    },
  },
  prettier, // must be last: turns off style rules that clash
);
```

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": false,
  "printWidth": 100,
  "trailingComma": "all"
}
```

```json
// package.json scripts
{ "scripts": { "lint": "eslint .", "format": "prettier --write ." } }
```

**Rules every fresher should know**

| Rule                      | Catches                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------- |
| `no-floating-promises`    | Calling an async function without `await`/`.catch` (Part 2)                         |
| `no-misused-promises`     | Passing an async function where a sync callback is expected (e.g. `if (asyncFn())`) |
| `no-explicit-any`         | Writing `any`                                                                       |
| `consistent-type-imports` | Using `import type` where appropriate                                               |
| `no-unused-vars`          | Dead variables                                                                      |

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ ESLint: @typescript-eslint/no-floating-promises
saveUser(user);
// ✅
await saveUser(user);
```

```
// ❌ Error: Parsing error: ... "parserOptions.project" has been set but file is not included
// ✅ The file must be in a tsconfig "include" (or use projectService). Config files like eslint.config.mjs
//    are usually excluded from type-aware linting.
```

**Version notes**

- ESLint 9 uses **flat config** (`eslint.config.*`). Old projects use `.eslintrc.json` with `"extends": ["plugin:@typescript-eslint/recommended"]` and `parser: "@typescript-eslint/parser"`. You will meet both. Config option names (`projectService`, `tseslint.config`) change between typescript-eslint versions, so check its current docs.
- Rule names and the recommended presets also evolve; verify against the docs when upgrading.

**5. Real-world use**
Run `npm run typecheck && npm run lint` in CI and in a pre-commit hook (Husky + lint-staged). Many companies fail the build on any error.

**6. Common mistakes**

- Putting Prettier as an ESLint plugin that reports formatting as lint errors. The modern advice is to keep them separate and use `eslint-config-prettier` only to avoid conflicts.
- Disabling a rule globally to quiet one file. Use a targeted `// eslint-disable-next-line rule-name` with a reason.

**7. Interview tip**
_"Difference between `tsc` and ESLint?"_ `tsc` checks types; ESLint checks code quality and patterns; Prettier formats.

---

### 📌 Key Takeaways

- Write ESM syntax; `module`, `package.json` `"type"` and your runner must agree. In Node ESM, relative imports need the `.js` extension.
- Use `import type` for types; it is erased, and required under `verbatimModuleSyntax` and `isolatedModules`.
- `.d.ts` files hold types only; `declare` promises something exists at runtime. Install `@types/*` as dev dependencies, but only for libraries that do not ship types.
- Augmentation (`declare global { namespace Express { interface Request { user?: ... } } }`) extends third-party types; the file must be a module and be covered by `include`.
- `paths` aliases only fix type checking; the runtime or build tool needs separate configuration.
- Use `tsc --noEmit` + ESLint (`typescript-eslint`, with `no-floating-promises`) + Prettier together, as separate tools.

✅ Part 7 complete.
