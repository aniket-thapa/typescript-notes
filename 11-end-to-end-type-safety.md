# Part 11: End-to-End Full Stack Type Safety

**Assumptions:** Node 20+, Express 5 + Prisma backend from Part 9, React + Vite frontend from Part 10, TanStack Query v5, Zod (one major version across the whole repo).

---

## 11.1 The problem and the three strategies 🔴 MUST KNOW

**1. What it is**
End-to-end type safety means a change to your API shape produces a **compile error in both the backend and the frontend**, instead of a runtime bug in production.

**2. Why it exists**
Backend and frontend are separate programs that talk over HTTP. Without a shared contract, each side guesses the other's shape (Part 8.6, 10.17). The guesses drift apart silently.

**3. Your options**

| Strategy                                                     | How types travel                                            | Best for                                   | Cost                                 |
| ------------------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------ | ------------------------------------ |
| **Shared package + Zod schemas** (this Part's main approach) | Both sides import the same schema and `z.infer` type        | Monorepo, REST API, most job codebases     | You write routes and client by hand  |
| **OpenAPI + code generation**                                | Backend publishes a spec; a tool generates the client types | Public APIs, teams with non-TS clients     | Extra build step, generated files    |
| **tRPC** (11.5)                                              | Client imports the server's router **type**                 | TS on both sides, same repo, no public API | Not plain REST; couples the two apps |

**7. Interview tip**
_"How do you keep frontend and backend types in sync?"_ Share the contract (schemas) in one package, or generate it from a spec, or use tRPC. Never copy interfaces by hand.

---

## 11.2 Sharing code in a monorepo 🔴 MUST KNOW

**1. What it is**
One repository, several packages, linked by **npm workspaces**. A `shared` package holds the contract. `api` and `web` depend on it like any npm package.

**3. Code**

```
shop/
  package.json            # workspaces
  tsconfig.base.json      # common compiler options
  packages/
    shared/               # schemas, DTO types, error codes. No Express, no React, no Prisma
    api/                  # Express + Prisma
    web/                  # Vite + React
```

```json
// shop/package.json
{
  "private": true,
  "workspaces": ["packages/*"],
  "scripts": {
    "build:shared": "npm run build -w @shop/shared",
    "dev:api": "npm run dev -w @shop/api",
    "dev:web": "npm run dev -w @shop/web",
    "typecheck": "npm run build:shared && npm run typecheck -ws --if-present"
  }
}
```

```json
// packages/shared/package.json
{
  "name": "@shop/shared",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" }
  },
  "scripts": { "build": "tsc", "dev": "tsc --watch --preserveWatchOutput" },
  "dependencies": { "zod": "^3.23.0" }
}
```

```json
// packages/shared/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "declaration": true,
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

```json
// packages/api/package.json and packages/web/package.json (dependencies)
{ "dependencies": { "@shop/shared": "*" } }
```

After `npm install` at the root, npm links `node_modules/@shop/shared` to the local folder. (pnpm writes `"workspace:*"` instead of `"*"`.) Use the Zod major version that matches your project (the `^3.23.0` above is only an example).

**Simpler variant:** point `exports` at `./src/index.ts`. Vite and `tsx` can read TS source directly, so there is no build step for `shared`. A backend built with `tsc` and a strict `rootDir` cannot, so you would need a bundler. Which works depends on your tooling.

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Frontend imports backend code to reuse a type: pulls Prisma/Express into the browser bundle
import type { User } from '../../api/src/models/user'; // type-only is erased, but one missing `type` breaks the bundle

// ✅ Both sides import from the shared package; shared depends on nothing but Zod
import type { TodoDto } from '@shop/shared';
```

**What belongs in `shared`:** Zod schemas, inferred DTO types, the `ApiResponse` envelope, error codes, small pure helpers. **What does not:** database models, anything importing Node, Express or React.

**5. Real-world use**
This layout (or Turborepo/Nx on top of it) is common in full-stack TS jobs. Your Part 7.10 project references let `tsc -b` rebuild `shared` before its dependents.

**6. Common mistakes & errors**

- `TS2307: Cannot find module '@shop/shared' or its corresponding type declarations.` → you did not run `npm install` at the **root**, or `shared` was never built so `dist/index.d.ts` does not exist.
- Editing `shared` but the frontend shows old types → `dist` is stale. Run `npm run dev -w @shop/shared` (watch mode) in a terminal.
- Inside `shared`, relative imports need `.js` under `NodeNext` (7.1): `export * from "./schemas/todo.js"`.
- **Two copies of Zod.** `instanceof z.ZodError` fails, and you get errors like `Type 'ZodObject<...>' is not assignable to type 'ZodObject<...>'` between identical-looking types. Check `npm ls zod` and make every package use one version.

**7. Interview tip**
_"What do you put in a shared package?"_ The contract (schemas and DTO types), never implementation or database types.

---

## 11.3 API contract design 🔴 MUST KNOW

**1. What it is**
A **contract** says, for every endpoint: the URL and method, the params/query/body shapes, the success response shape, and the error shape. Here it is written as Zod schemas in `shared`.

**2. Why it exists**
A schema validates at runtime **and** gives a type, so one definition serves the server's input check, the client's response check, and both sides' compile-time types.

**3. Three kinds of types, kept separate**

| Type                          | Lives in                    | Example                                 | Rule                               |
| ----------------------------- | --------------------------- | --------------------------------------- | ---------------------------------- |
| **DB row**                    | `api` (generated by Prisma) | `Todo` with `userId`, `createdAt: Date` | Never leaves the repository/mapper |
| **DTO** (what goes over HTTP) | `shared`                    | `TodoDto` with `createdAt: string`      | The contract                       |
| **View state**                | `web`                       | `TodoDto & { isEditing: boolean }`      | Frontend-only                      |

**Code**

```ts
// packages/shared/src/api.ts
import { z } from 'zod';

export const ApiErrorSchema = z.object({
  code: z.string(),
  message: z.string(),
  details: z.unknown().optional(),
});
export const ApiFailureSchema = z.object({
  success: z.literal(false),
  error: ApiErrorSchema,
});

// A function that builds a success envelope for any data schema
export const apiSuccess = <S extends z.ZodType>(data: S) =>
  z.object({ success: z.literal(true), data });

export type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: z.infer<typeof ApiErrorSchema> };
```

```ts
// packages/shared/src/schemas/todo.ts
import { z } from 'zod';

export const TodoSchema = z.object({
  id: z.string(),
  title: z.string(),
  done: z.boolean(),
  createdAt: z.string().datetime(), // a STRING: JSON has no Date type (see below)
});
export type TodoDto = z.infer<typeof TodoSchema>;

export const CreateTodoSchema = z.object({
  title: z.string().trim().min(1).max(200),
});
export type CreateTodoInput = z.infer<typeof CreateTodoSchema>;

export const UpdateTodoSchema = z
  .object({
    title: z.string().trim().min(1).max(200),
    done: z.boolean(),
  })
  .partial();
export type UpdateTodoInput = z.infer<typeof UpdateTodoSchema>;
```

```ts
// packages/shared/src/schemas/auth.ts
import { z } from 'zod';

export const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});
export type LoginInput = z.infer<typeof LoginSchema>;

export const AuthUserSchema = z.object({
  id: z.string(),
  email: z.string(),
  name: z.string(),
});
export const LoginResponseSchema = z.object({
  token: z.string(),
  user: AuthUserSchema,
});
export type LoginResponse = z.infer<typeof LoginResponseSchema>;
```

```ts
// packages/shared/src/index.ts
export * from './api.js';
export * from './schemas/todo.js';
export * from './schemas/auth.js';
```

Version note: in Zod 4, prefer `z.iso.datetime()`; `z.string().datetime()` still works but is deprecated.

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Date in a DTO: after JSON.stringify/parse it is a string, so the type lies
interface TodoDto {
  createdAt: Date;
}
// ✅ Send an ISO string; convert to Date in the UI only if you need to (new Date(todo.createdAt))

// ❌ Reusing the DB row as the response: leaks passwordHash, userId, internal fields
res.json({ success: true, data: userRow });
// ✅ A mapper that whitelists fields (11.6)
```

Other JSON gotchas: `undefined` fields disappear, `bigint` cannot be serialized, `Map`/`Set` become `{}`.

**5. Real-world use**
Error `code`s (`"NOT_FOUND"`, `"VALIDATION_ERROR"`) belong in the contract too, so the UI can show specific messages. Version the API (`/v1/...`) if third parties depend on it.

**6. Common mistakes**

- Defining `interface Todo` by hand beside the schema (two sources of truth). Always `z.infer`.
- Making the DTO identical to the DB row "to save time". The first renamed column breaks the API.
- A `CreateInput` that includes `id` or `userId`. The server owns those.

**7. Interview tip**
_"Why separate DTOs from database models?"_ The API shape should not change when the database does, and DTOs stop sensitive fields leaking.

---

## 11.4 A typed API client layer 🔴 MUST KNOW

**1. What it is**
One module in `web` that is the **only** place that calls `fetch`. It exposes functions like `api.todos.list()`, whose types come from the shared schemas.

**2. Why it exists**
Components never see URLs, headers or `any`. If an endpoint changes, you fix one file and the compiler lists every component that breaks.

**3. Code**

```ts
// packages/web/src/api/client.ts
import { z } from 'zod';
import {
  apiSuccess,
  ApiFailureSchema,
  LoginSchema,
  LoginResponseSchema,
  TodoSchema,
  CreateTodoSchema,
  UpdateTodoSchema,
  type LoginInput,
  type CreateTodoInput,
  type UpdateTodoInput,
} from '@shop/shared';

export class ApiError extends Error {
  constructor(
    public readonly status: number,
    public readonly code: string,
    message: string,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

let token: string | null = null; // in memory; lost on refresh (see note)
export const setToken = (t: string | null): void => {
  token = t;
};

async function call<S extends z.ZodType>(
  path: string,
  schema: S,
  init: { method?: string; body?: unknown } = {},
): Promise<z.infer<S>> {
  const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
    method: init.method ?? 'GET',
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
    body: init.body === undefined ? undefined : JSON.stringify(init.body),
  });

  const json: unknown = await res.json().catch(() => null);
  const parsed = z
    .union([apiSuccess(schema), ApiFailureSchema])
    .safeParse(json);

  if (!parsed.success)
    throw new ApiError(
      res.status,
      'BAD_RESPONSE',
      'Server sent an unexpected response',
    );
  if (!parsed.data.success)
    throw new ApiError(
      res.status,
      parsed.data.error.code,
      parsed.data.error.message,
    );
  return parsed.data.data; // if your Zod version cannot simplify this generic type,
  // `as z.infer<S>` here is acceptable: the value was just validated
}

export const api = {
  auth: {
    login: (input: LoginInput) =>
      call('/auth/login', LoginResponseSchema, {
        method: 'POST',
        body: LoginSchema.parse(input),
      }),
  },
  todos: {
    list: () => call('/todos', z.array(TodoSchema)),
    create: (input: CreateTodoInput) =>
      call('/todos', TodoSchema, {
        method: 'POST',
        body: CreateTodoSchema.parse(input),
      }),
    update: (id: string, input: UpdateTodoInput) =>
      call(`/todos/${id}`, TodoSchema, {
        method: 'PATCH',
        body: UpdateTodoSchema.parse(input),
      }),
  },
};
```

Hover `api.todos.list` to see `() => Promise<{ id: string; title: string; done: boolean; createdAt: string }[]>`. Nobody wrote that type by hand.

**4. ❌ Wrong / ✅ Right**

```ts
const todos = (await fetch('/api/todos').then((r) => r.json())) as Todo[]; // ❌ unchecked, scattered in components
const todos2 = await api.todos.list(); // ✅ validated, one place
```

**5. Real-world use**
Add a global handler: on `ApiError` with status `401`, clear the token and redirect to login. For persistence, many apps store the token in an `httpOnly` cookie set by the server (not readable by JS, so safer against XSS) instead of memory or `localStorage`. That is a security design choice beyond this Part.

**6. Common mistakes**

- Throwing on `!res.ok` before reading the body, so the server's error `code` is lost.
- Parsing the response, but sending **unvalidated input**. `LoginSchema.parse(input)` above catches a bug before the network call.
- Importing `fetch` wrappers in many files instead of using `api`.

**7. Interview tip**
_"Where do you put API calls in a React app?"_ A typed client module that validates responses; components call it through TanStack Query.

---

## 11.5 Intro to tRPC 🟡 GOOD TO KNOW

**1. What it is**
tRPC lets the frontend call backend functions as if they were local functions. It shares **types, not schemas**: the client imports the _type_ of the server's router, and TS does the rest. There is no REST contract file and no code generation.

**3. Code (tRPC v11 style; check the docs for your version)**

```ts
// api/src/trpc.ts
import { initTRPC } from '@trpc/server';
import { z } from 'zod';

const t = initTRPC.create();

export const appRouter = t.router({
  todoList: t.procedure.query(() => listTodos()), // read = query
  todoCreate: t.procedure
    .input(z.object({ title: z.string().min(1) })) // validated at runtime by Zod
    .mutation(({ input }) => createTodo(input.title)), // write = mutation; input is typed
});

export type AppRouter = typeof appRouter; // the ONLY thing the frontend needs
```

```ts
// web/src/trpc.ts
import { createTRPCClient, httpBatchLink } from '@trpc/client';
import type { AppRouter } from '../../api/src/trpc'; // `import type`: erased, no server code in the bundle

export const trpc = createTRPCClient<AppRouter>({
  links: [httpBatchLink({ url: 'http://localhost:3000/trpc' })],
});

const todos = await trpc.todoList.query(); // fully typed, no codegen
await trpc.todoCreate.mutate({ title: 123 });
// ❌ TS2322: Type 'number' is not assignable to type 'string'.
```

Express mounts it through `@trpc/server/adapters/express`. There is also a TanStack Query integration, with different package names across versions.

**What changes vs the REST + Zod approach**

|                                          | REST + shared Zod           | tRPC                          |
| ---------------------------------------- | --------------------------- | ----------------------------- |
| Endpoint types                           | Defined by contract schemas | **Inferred** from server code |
| Response validated at runtime on client? | Yes (`parse`)               | No, trusted from the type     |
| Usable by non-TS clients / public API    | Yes                         | Awkward                       |
| Boilerplate                              | More                        | Much less                     |

**4. ❌ Wrong / ✅ Right**

```ts
import { appRouter } from '../../api/src/trpc'; // ❌ value import: bundles server code (and secrets) into the browser
import type { AppRouter } from '../../api/src/trpc'; // ✅
```

tRPC types are only as fresh as your last build. A **deployed** old frontend can call a new server that changed a procedure. Types cannot see deployment skew, so deploy both together or keep changes backwards-compatible.

**5. Real-world use**
Popular in Next.js and "T3-style" stacks. For a fresher portfolio, build the REST + Zod version (it shows you understand the layers), then add a tRPC variant.

**7. Interview tip**
_"What is tRPC?"_ End-to-end typesafe procedure calls that share the server router's type with the client. Mention it suits internal TS-only apps, not public APIs.

---

## 11.6 A complete mini feature: Login + Todos 🔴 MUST KNOW

**Goal:** follow one piece of data (`Todo`) through every layer and see which type guards each boundary.

| Layer             | Type comes from                        | Checked by                                    |
| ----------------- | -------------------------------------- | --------------------------------------------- |
| Database          | `schema.prisma` → generated `Todo` row | `prisma generate`                             |
| Repository        | `TodoRow`                              | Compiler                                      |
| Mapper            | `TodoDto` (shared `z.infer`)           | Compiler (return type)                        |
| HTTP request in   | `CreateTodoSchema`                     | **Runtime** (Zod middleware)                  |
| HTTP response out | `ApiResponse<TodoDto>`                 | Compiler (server), **runtime** parse (client) |
| Client            | `api.todos.*`                          | Compiler + runtime                            |
| UI                | `useQuery` data `TodoDto[]`            | Compiler                                      |

### Step 1: DB model

```prisma
// packages/api/prisma/schema.prisma (add to the 9.10 models)
model Todo {
  id        String   @id @default(uuid())
  title     String
  done      Boolean  @default(false)
  createdAt DateTime @default(now())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
}
```

Add `todos Todo[]` to `User`, then run `npx prisma migrate dev --name add-todo`.

### Step 2: Repository, mapper, service

```ts
// packages/api/src/repositories/todo.repository.ts
import type { Todo as TodoRow } from '@prisma/client'; // the DB row type
import { prisma } from '../db.js';

export interface TodoRepository {
  listByUser(userId: string): Promise<TodoRow[]>;
  create(userId: string, title: string): Promise<TodoRow>;
  update(
    id: string,
    userId: string,
    data: { title?: string; done?: boolean },
  ): Promise<TodoRow | null>;
}

export class PrismaTodoRepository implements TodoRepository {
  listByUser(userId: string) {
    return prisma.todo.findMany({
      where: { userId },
      orderBy: { createdAt: 'desc' },
    });
  }
  create(userId: string, title: string) {
    return prisma.todo.create({ data: { userId, title } });
  }
  async update(
    id: string,
    userId: string,
    data: { title?: string; done?: boolean },
  ) {
    // updateMany with BOTH id and userId: you can only change your own todos
    const result = await prisma.todo.updateMany({
      where: { id, userId },
      data,
    });
    return result.count === 0
      ? null
      : prisma.todo.findUnique({ where: { id } });
  }
}
```

```ts
// packages/api/src/mappers/todo.mapper.ts
import type { Todo as TodoRow } from '@prisma/client';
import type { TodoDto } from '@shop/shared';

// Return type = the contract. If the schema changes, this stops compiling.
export const toTodoDto = (row: TodoRow): TodoDto => ({
  id: row.id,
  title: row.title,
  done: row.done,
  createdAt: row.createdAt.toISOString(), // Date → string. userId is deliberately left out
});
```

```ts
// packages/api/src/services/todo.service.ts
import { NotFoundError } from '../errors/app-error.js';
import type { TodoRepository } from '../repositories/todo.repository.js';
import type { UpdateTodoInput } from '@shop/shared';

export class TodoService {
  constructor(private readonly repo: TodoRepository) {}

  list(userId: string) {
    return this.repo.listByUser(userId);
  }
  create(userId: string, title: string) {
    return this.repo.create(userId, title);
  }

  async update(id: string, userId: string, input: UpdateTodoInput) {
    const row = await this.repo.update(id, userId, input);
    if (!row) throw new NotFoundError('Todo', id);
    return row;
  }
}
```

Note: with `exactOptionalPropertyTypes` off (the default), `UpdateTodoInput` (`{ title?: string | undefined; ... }`) is assignable to the repository's `data` type.

### Step 3: Controller and routes (types from the contract)

```ts
// packages/api/src/controllers/todo.controller.ts
import type { Request, Response } from 'express';
import type {
  ApiResponse,
  CreateTodoInput,
  TodoDto,
  UpdateTodoInput,
} from '@shop/shared';
import { requireUser } from '../auth/require-user.js'; // the helper from 9.5
import { sendOk } from '../utils/respond.js';
import { toTodoDto } from '../mappers/todo.mapper.js';
import type { TodoService } from '../services/todo.service.js';

export class TodoController {
  constructor(private readonly service: TodoService) {}

  list = async (
    req: Request,
    res: Response<ApiResponse<TodoDto[]>>,
  ): Promise<void> => {
    const rows = await this.service.list(requireUser(req).id);
    sendOk(res, rows.map(toTodoDto));
  };

  create = async (
    req: Request<unknown, unknown, CreateTodoInput>,
    res: Response<ApiResponse<TodoDto>>,
  ): Promise<void> => {
    const row = await this.service.create(requireUser(req).id, req.body.title);
    sendOk(res, toTodoDto(row), 201);
  };

  update = async (
    req: Request<{ id: string }, unknown, UpdateTodoInput>,
    res: Response<ApiResponse<TodoDto>>,
  ): Promise<void> => {
    const row = await this.service.update(
      req.params.id,
      requireUser(req).id,
      req.body,
    );
    sendOk(res, toTodoDto(row));
  };
}
```

```ts
// routes: authentication first, then validation, then the controller
router.use(authenticate);
router.get('/', controller.list);
router.post('/', validateBody(CreateTodoSchema), controller.create);
router.patch('/:id', validateBody(UpdateTodoSchema), controller.update);
```

Try `sendOk(res, rows)` in `list` without mapping. It fails with an error like `TS2345`, because `TodoRow` has `createdAt: Date` and `userId`, while `TodoDto` has a string date. The compiler stops the leak.

### Step 4: Login form (shared schema, both sides)

```tsx
// packages/web/src/features/auth/LoginForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { LoginSchema, type LoginInput } from '@shop/shared';
import { api, setToken, ApiError } from '../../api/client';

export function LoginForm({ onLoggedIn }: { onLoggedIn: () => void }) {
  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm<LoginInput>({ resolver: zodResolver(LoginSchema) });

  const onSubmit = async (values: LoginInput) => {
    try {
      const { token } = await api.auth.login(values); // { token, user }, typed
      setToken(token);
      onLoggedIn();
    } catch (e) {
      if (e instanceof ApiError && e.status === 401)
        setError('root', { message: 'Wrong email or password' });
      else throw e;
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input type="email" {...register('email')} />
      {errors.email?.message}
      <input type="password" {...register('password')} />
      {errors.password?.message}
      {errors.root?.message}
      <button disabled={isSubmitting}>Log in</button>
    </form>
  );
}
```

The server validates with the **same** `LoginSchema`. A password rule changed in `shared` changes both sides.

### Step 5: Todo page

```tsx
// packages/web/src/features/todos/TodoPage.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import {
  queryOptions,
  useMutation,
  useQuery,
  useQueryClient,
} from '@tanstack/react-query';
import {
  CreateTodoSchema,
  type CreateTodoInput,
  type TodoDto,
} from '@shop/shared';
import { api } from '../../api/client';

const todosQuery = queryOptions({
  queryKey: ['todos'],
  queryFn: api.todos.list,
});

export function TodoPage() {
  const queryClient = useQueryClient();
  const invalidate = () =>
    queryClient.invalidateQueries({ queryKey: todosQuery.queryKey });

  const { data, isPending, isError, error } = useQuery(todosQuery);
  const create = useMutation({
    mutationFn: api.todos.create,
    onSuccess: invalidate,
  });
  const toggle = useMutation({
    mutationFn: (todo: TodoDto) =>
      api.todos.update(todo.id, { done: !todo.done }),
    onSuccess: invalidate,
  });

  const {
    register,
    handleSubmit,
    reset,
    formState: { errors },
  } = useForm<CreateTodoInput>({ resolver: zodResolver(CreateTodoSchema) });

  if (isPending) return <p>Loading…</p>;
  if (isError) return <p>{error.message}</p>;

  return (
    <>
      <form
        onSubmit={handleSubmit((values) =>
          create.mutate(values, { onSuccess: () => reset() }),
        )}
      >
        <input {...register('title')} placeholder="New todo" />
        {errors.title?.message}
        <button disabled={create.isPending}>Add</button>
      </form>
      <ul>
        {data.map((t) => (
          <li key={t.id}>
            <input
              type="checkbox"
              checked={t.done}
              onChange={() => toggle.mutate(t)}
            />
            {t.title}{' '}
            <small>{new Date(t.createdAt).toLocaleDateString()}</small>
          </li>
        ))}
      </ul>
    </>
  );
}
```

### Step 6: See the payoff

Rename `done` to `completed` in `TodoSchema`, rebuild `shared`, and run `npm run typecheck`:

- API: `toTodoDto` fails (`TS2353`: object literal may only specify known properties, `done` does not exist), and the `UpdateTodoSchema`-based service call breaks.
- Web: `t.done` fails with `TS2339: Property 'done' does not exist on type`, and the `api.todos.update(..., { done })` call fails.

One edit, and the compiler lists every place to fix. That is the goal of this entire Part.

**What types still do NOT protect (know these for interviews)**

- **Authorization:** TS cannot tell if a user may touch a todo. The `where: { id, userId }` check does.
- **Deployment skew:** an old frontend can hit a new API. Runtime parsing in `call` turns this into a clear `BAD_RESPONSE` error instead of a silent bug.
- **Database drift:** if the DB schema and Prisma client differ, run migrations and `generate`.
- **Behavior:** wrong logic with correct types. That is what tests (9.15) are for.

**Common mistakes in this feature**

- Forgetting `authenticate` before `validateBody`, so unauthenticated users can probe validation errors.
- `requireUser(req).id` repeated everywhere is fine, but `req.user!` hides a missing-middleware bug (9.5).
- A TanStack `queryKey` that omits user identity, so another account's cached todos appear after logging out. Call `queryClient.clear()` on logout.

**Interview tip**
_"Walk me through type safety from the database to the UI."_ DB schema generates row types; a mapper converts rows to DTOs typed by the shared Zod schema; the server validates input at runtime; the client validates responses and exposes typed functions; components consume inferred types.

---

### 📌 Key Takeaways

- Put the contract (Zod schemas, inferred DTO types, response envelope) in a shared workspace package. It must not import Express, Prisma or React.
- Keep DB rows, DTOs and UI state as separate types, and convert with a mapper whose **return type** is the contract.
- JSON has no `Date`: DTOs carry ISO strings. Only whitelisted fields leave the server.
- One typed API client module validates responses with the shared schemas; components never call `fetch` directly.
- tRPC shares the server router's **type** (use `import type`) and skips the REST contract; it is great for internal TS-only apps and does not cover deployment skew.
- Types prove shapes match, not that a user is allowed to do something. Keep runtime validation, ownership checks and tests.

✅ Part 11 complete.
