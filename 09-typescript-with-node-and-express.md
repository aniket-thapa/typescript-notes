# Part 9a: Backend Setup, Express Typing & Layers

Part 9 is split into three replies to stay within the word limit:

- **9a** (this reply): project setup, typing `Request`/`Response`/`NextFunction`, middleware, env validation, controllers/services/repositories.
- **9b**: Mongoose/Prisma, JWT auth, centralized error handler, Zod validation middleware, typed response format.
- **9c**: testing, NestJS intro, deployment.

**Assumptions:** Node 20+, **Express 5** (what `npm install express` gives now), ESM (`"type": "module"`), and the Node `tsconfig` from Part 0. Express 4 differences are flagged where they matter.

---

## 9.1 Project setup from scratch 🔴 MUST KNOW

**1. What it is**
A minimal, correct TS + Express project that you can type-check, run in dev, build and start in production.

**3. Code**

```bash
mkdir shop-api && cd shop-api
npm init -y
npm install express zod
npm install -D typescript tsx @types/node @types/express
npx tsc --init   # then replace its contents with the Node tsconfig from Part 0.6
```

```json
// package.json (relevant parts)
{
  "type": "module",
  "scripts": {
    "dev": "tsx watch --env-file=.env src/server.ts",
    "typecheck": "tsc --noEmit",
    "build": "tsc",
    "start": "node dist/server.js"
  }
}
```

```ts
// src/app.ts: builds the app, does NOT listen (so tests can import it)
import express from 'express';

export const app = express();
app.use(express.json());

app.get('/health', (_req, res) => {
  res.json({ status: 'ok' }); // _req: underscore = "intentionally unused"
});
```

```ts
// src/server.ts: the only file that starts the server
import { app } from './app.js'; // .js extension: ESM rule from 7.1

const port = Number(process.env.PORT ?? 3000);
app.listen(port, () => console.log(`Listening on :${port}`));
```

**Notes**

- `tsx watch` restarts on change but does **not** type-check (Part 0). Run `npm run typecheck` separately, or keep VS Code's Error Lens open.
- `--env-file` is built into Node 20.6+. Older projects use `import "dotenv/config"` or `nodemon` with `ts-node`; you will see both in legacy code.
- In production `.env` files are usually not shipped. The host (Docker, Render, etc.) injects variables, so `start` has no `--env-file`.

**6. Common mistakes & errors**

- `TS2307: Cannot find module './app' ...` → ESM needs `./app.js` (7.1).
- `TS7016: Could not find a declaration file for module 'express'` → install `@types/express` (7.5).
- Putting `app.listen` inside `app.ts`. Tests then open real ports.

---

## 9.2 Folder structure 🔴 MUST KNOW

```
src/
  app.ts            # express app + middleware + routes
  server.ts         # listen()
  config/env.ts     # validated env vars (9.7)
  routes/           # URL → controller method
  controllers/      # HTTP only: read req, call service, write res
  services/         # business rules; no Express types here
  repositories/     # database access only
  middleware/       # auth, validation, error handler
  schemas/          # Zod schemas (+ inferred types)
  errors/           # AppError classes (8.4)
  types/            # .d.ts augmentations (7.7)
```

**The rule that matters:** `Request` and `Response` stay in controllers and middleware. Services take plain typed values and return plain typed values. Then services are testable without HTTP and reusable from a CLI or queue worker. This is opinion-based (some teams use feature folders: `users/`, `orders/`), but the layering idea is universal.

---

## 9.3 Typing `Request`, `Response`, `NextFunction` 🔴 MUST KNOW

**1. What it is**
Express's types, imported **as types** (7.2):

- `Request`: the incoming request.
- `Response`: the outgoing response.
- `NextFunction`: calls the next middleware (or the error handler if passed an error).
- `RequestHandler`: the type of a whole handler function.

**3. Code**

```ts
import type { Request, Response, NextFunction, RequestHandler } from 'express';

// Style A: annotate the parameters
function ping(req: Request, res: Response, next: NextFunction): void {
  res.json({ pong: true });
}

// Style B: annotate the whole function. Parameters are inferred (contextual typing, 2.4)
const ping2: RequestHandler = (req, res, next) => {
  res.json({ pong: true });
};
```

**4. ❌ Wrong / ✅ Right**

```ts
import { Request } from 'express'; // ⚠️ works, but use `import type` (TS1484 with verbatimModuleSyntax)

// ❌ Forgetting that `Request` and `Response` also exist as browser globals (DOM lib)
// If your tsconfig has lib "DOM" and you forget the import, TS silently uses the fetch API types:
// TS2339: Property 'json' does not exist on type 'Response' ... confusing!
// ✅ Backend tsconfig should NOT include "DOM" (Part 0.6), and always import from express.
```

**Express 4 vs 5 (version-dependent):** with `@types/express` v5, a handler may return `void` or `Promise<void>`. So `return res.json(x);` inside an `async` handler is a type error, because it returns `Promise<Response>`. Write `res.json(x); return;` instead. Older v4 types were looser, and you will see `return res.status(404).json(...)` all over legacy code.

**5. Real-world use**
Every route handler and middleware. Library-style code also uses `RequestHandler` as the return type of middleware factories (9.6).

**6. Common mistakes & errors**

- `TS2769: No overload matches this call` on `app.get("/x", handler)`. Your handler's parameter types disagree with what Express provides. Read the **last** nested line (Part 0.7).
- Annotating `res` as `Response<User>` and then sending something else. `Response<T>` constrains `res.json()` input only if you write it.

**7. Interview tip**
_"What is `NextFunction`?"_ The callback that passes control to the next middleware; passing an argument sends the request to the error handler.

---

## 9.4 Typing `params`, `query` and `body` 🔴 MUST KNOW

**1. What it is**
`Request` has four generics, in order: `Request<Params, ResBody, ReqBody, ReqQuery>`.

**3. Code**

```ts
interface CreateProductBody {
  title: string;
  price: number;
}
interface ListQuery {
  page?: string;
  limit?: string;
} // query values are STRINGS

// GET /products/:id
const getProduct = async (
  req: Request<{ id: string }>,
  res: Response,
): Promise<void> => {
  const id = req.params.id; // string
  res.json({ id });
};

// POST /products
const createProduct = async (
  req: Request<unknown, unknown, CreateProductBody>,
  res: Response,
): Promise<void> => {
  const { title, price } = req.body; // typed (but see warning below)
  res.status(201).json({ title, price });
};

// GET /products?page=2
const listProducts = async (
  req: Request<unknown, unknown, unknown, ListQuery>,
  res: Response,
): Promise<void> => {
  const page = Number(req.query.page ?? '1');
  res.json({ page });
};
```

Unused slots: use `unknown`. Older code writes `{}` (which linters now flag as too broad), or `any`.

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ These generics are CLAIMS, not checks (Part 8.6).
// A client can send { "price": "free" } and TS will still say price is a number.
const { price } = req.body;
// ✅ Validate with Zod (9b) and use the parsed result.

// ❌ Typing a numeric id param as number: route params are always strings
Request<{ id: number }>; // compiles, but at runtime it is "42", not 42
// ✅ Request<{ id: string }> then Number(id) / schema coercion
```

**6. Common mistakes & errors**

- `req.body` is `undefined` if you forgot `app.use(express.json())`. Types will not warn you.
- `req.query.page` by default is a union of `string | string[] | ParsedQs | ParsedQs[] | undefined`. Narrow it, or validate it.
- Typing `req.query` as numbers (`{ page: number }`) is a lie: query strings are text.

**7. Interview tip**
_"How do you type the request body in Express?"_ Third generic on `Request`. Then add that it is unchecked, so you validate with Zod.

---

## 9.5 Extending `Request` (e.g. `req.user`) 🔴 MUST KNOW

Full setup is in **7.7**. Here is the minimum for this project:

```ts
// src/types/express.d.ts
import type { Role } from '../middleware/auth.js';

declare global {
  namespace Express {
    interface Request {
      user?: { id: string; role: Role };
    }
  }
}
export {};
```

Make sure `include: ["src"]` covers it. Symptom of a miss: `TS2339: Property 'user' does not exist on type 'Request'`. Restart the TS server after adding the file.

**Why optional:** public routes have no user. Inside protected handlers, TS still sees `user?`, so you either check it or use `req.user!`. The `!` is an unchecked promise that your auth middleware ran. A safer option is to throw if it is missing:

```ts
function requireUser(req: Request): NonNullable<Request['user']> {
  if (!req.user) throw new UnauthorizedError(); // defined in 9b
  return req.user;
}
```

Indexed access (`Request["user"]`, Part 4.4) plus `NonNullable` (5.5) means no repeated type.

---

## 9.6 Typing middleware 🔴 MUST KNOW

**1. What it is**
Middleware is a function `(req, res, next)` that either responds or calls `next()`. Type it as `RequestHandler`. A **factory** (a function returning middleware) returns `RequestHandler` too (Part 2.7).

**3. Code**

```ts
// src/middleware/auth.ts
import type { RequestHandler } from 'express';

export type Role = 'admin' | 'user';

// Plain middleware
export const requestLogger: RequestHandler = (req, _res, next) => {
  console.log(`${req.method} ${req.originalUrl}`);
  next();
};

// Factory: requireRole("admin") returns a middleware
export const requireRole =
  (...allowed: Role[]): RequestHandler =>
  (req, res, next) => {
    if (!req.user) {
      res.status(401).json({ success: false, error: 'Not authenticated' });
      return; // stop: do NOT call next()
    }
    if (!allowed.includes(req.user.role)) {
      res.status(403).json({ success: false, error: 'Forbidden' });
      return;
    }
    next();
  };

// Usage
// router.delete("/:id", requireRole("admin"), controller.remove);
// requireRole("superadmin");
// ❌ TS2345: Argument of type '"superadmin"' is not assignable to parameter of type 'Role'.
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Sends a response AND calls next(): "Cannot set headers after they are sent" at runtime
if (!req.user) res.status(401).json({ error: 'no' });
next();
// ✅ return after responding (as above)
```

The compiler cannot catch "responded twice." Discipline and tests do.

**5. Real-world use**
Auth, rate limiting, request IDs, validation (9b), and logging. Order matters: `express.json()` before routes, auth before protected routes, error handler **last**.

**6. Common mistakes**

- An error-handling middleware must have exactly **four** parameters `(err, req, res, next)`, or Express treats it as normal middleware. Typed with `ErrorRequestHandler` in 9b.
- Using `any` for the factory's return type loses the Request generics. Keep `RequestHandler`.

---

## 9.7 Typing and validating environment variables 🔴 MUST KNOW

**1. What it is**
`process.env.X` is always `string | undefined`. Instead of promising types (7.7), **validate once at startup** and export a typed object.

**3. Code**

```ts
// src/config/env.ts
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z
    .enum(['development', 'production', 'test'])
    .default('development'),
  PORT: z.coerce.number().int().positive().default(3000), // env values are strings → coerce
  DATABASE_URL: z.string().min(1),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_EXPIRES_IN: z.string().default('15m'),
});

const parsed = EnvSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('❌ Invalid environment variables:');
  for (const issue of parsed.error.issues) {
    console.error(`  ${issue.path.join('.')}: ${issue.message}`);
  }
  process.exit(1); // fail fast: a misconfigured server must not start
}

export const env = parsed.data; // fully typed
export type Env = z.infer<typeof EnvSchema>;
```

```ts
import { env } from './config/env.js';
app.listen(env.PORT); // number, not string | undefined
env.JWT_SECRET.length; // string
env.JWT_SCRET;
// ❌ TS2551: Property 'JWT_SCRET' does not exist on type '{...}'. Did you mean 'JWT_SECRET'?
```

**4. ❌ Wrong / ✅ Right**

```ts
const secret = process.env.JWT_SECRET as string; // ❌ if missing: "undefined" quietly used as the secret
const secret2 = env.JWT_SECRET; // ✅ checked at startup
```

**5. Real-world use**
Every deployed backend. Import `env` instead of touching `process.env` anywhere else (a lint rule can enforce it). Never log secrets, and never commit `.env`; commit a `.env.example`.

**6. Common mistakes**

- `z.coerce.boolean()` turns `"false"` into `true`. Parse booleans with `z.enum(["true","false"]).transform(v => v === "true")`.
- Validating env **after** other modules already read `process.env`. Import `env.ts` first.

**7. Interview tip**
_"How do you handle env vars in TS?"_ Validate with a schema at startup and export a typed object, instead of casting `process.env`.

---

## 9.8 Typed controllers, services and repositories 🔴 MUST KNOW

**1. What it is**
Three layers with one job each:

| Layer          | Knows about    | Does                                                    |
| -------------- | -------------- | ------------------------------------------------------- |
| **Controller** | Express        | Reads input from `req`, calls the service, writes `res` |
| **Service**    | Business rules | Decides what is allowed; throws `AppError`s (8.4)       |
| **Repository** | The database   | Reads and writes data; returns plain objects            |

**2. Why it exists**
Each layer can be changed or tested alone. Swapping the database touches only the repository.

**3. Code**

```ts
// src/repositories/user.repository.ts
import { randomUUID } from 'node:crypto';

export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}
export type CreateUserInput = Omit<User, 'id' | 'createdAt'>; // Part 5

export interface UserRepository {
  findById(id: string): Promise<User | null>;
  findAll(): Promise<User[]>;
  create(data: CreateUserInput): Promise<User>;
}

export class InMemoryUserRepository implements UserRepository {
  private readonly users = new Map<string, User>();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }
  async findAll(): Promise<User[]> {
    return [...this.users.values()];
  }
  async create(data: CreateUserInput): Promise<User> {
    const user: User = { ...data, id: randomUUID(), createdAt: new Date() };
    this.users.set(user.id, user);
    return user;
  }
}
```

```ts
// src/services/user.service.ts: no Express imports
import { NotFoundError } from '../errors/app-error.js';
import type {
  CreateUserInput,
  User,
  UserRepository,
} from '../repositories/user.repository.js';

export class UserService {
  constructor(private readonly repo: UserRepository) {} // parameter property (6.3)

  async getById(id: string): Promise<User> {
    const user = await this.repo.findById(id);
    if (!user) throw new NotFoundError('User', id); // 8.4
    return user;
  }
  list(): Promise<User[]> {
    return this.repo.findAll();
  }
  create(input: CreateUserInput): Promise<User> {
    return this.repo.create(input);
  }
}
```

```ts
// src/controllers/user.controller.ts
import type { Request, Response } from 'express';
import type { CreateUserInput } from '../repositories/user.repository.js';
import type { UserService } from '../services/user.service.js';

export class UserController {
  constructor(private readonly service: UserService) {}

  // Arrow-function properties keep `this` when passed as `router.get("/", controller.list)` (2.11)
  list = async (_req: Request, res: Response): Promise<void> => {
    res.json({ success: true, data: await this.service.list() });
  };

  getById = async (
    req: Request<{ id: string }>,
    res: Response,
  ): Promise<void> => {
    res.json({
      success: true,
      data: await this.service.getById(req.params.id),
    });
  };

  create = async (
    req: Request<unknown, unknown, CreateUserInput>,
    res: Response,
  ): Promise<void> => {
    const user = await this.service.create(req.body); // validated in 9b
    res.status(201).json({ success: true, data: user });
  };
}
```

```ts
// src/routes/user.routes.ts
import { Router } from 'express';
import type { UserController } from '../controllers/user.controller.js';

export function createUserRouter(controller: UserController): Router {
  const router = Router();
  router.get('/', controller.list);
  router.get('/:id', controller.getById); // "/:id" → params typed as { id: string } automatically
  router.post('/', controller.create);
  return router;
}
```

```ts
// src/app.ts: wiring ("composition root"): the ONLY place that knows concrete classes
const userRepo = new InMemoryUserRepository();
const userService = new UserService(userRepo);
const userController = new UserController(userService);
app.use('/users', createUserRouter(userController));
// app.use(errorHandler);  ← added in 9b
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Method syntax loses `this` when detached:
async list(req: Request, res: Response) { this.service... }   // router.get("/", controller.list) → "Cannot read properties of undefined"
// ✅ arrow property (above), or router.get("/", (req, res) => controller.list(req, res))

// ❌ Service imports Request: now it cannot be reused or tested without HTTP
```

**5. Real-world use**
In 9b you replace `InMemoryUserRepository` with a Mongoose or Prisma one. The service and controller do not change, because they depend on the `UserRepository` interface.

**6. Common mistakes & errors**

- `TS2345` when passing a controller method to a router: its `Request` generics do not match the route path's params. Make `Request<{ id: string }>` match `/:id`.
- Business logic in controllers ("fat controllers"), or HTTP status codes in services. Services throw errors; the error handler maps them to codes.
- Express 4 only: an async handler that throws is **not** caught. Wrap it in `asyncHandler` (2.9). Express 5 forwards rejected promises to the error handler itself.

**7. Interview tip**
_"How do you structure an Express + TS backend?"_ Routes → controllers (HTTP) → services (rules) → repositories (data), wired via constructor injection against interfaces.

---

### 📌 Key Takeaways

- Keep `app.ts` (build the app) separate from `server.ts` (listen). Use `tsx` for dev and `tsc --noEmit` for checking.
- Import `Request`, `Response`, `NextFunction` and `RequestHandler` as types; `Request<Params, ResBody, ReqBody, ReqQuery>` generics are unchecked claims.
- Route params and query values are strings; convert and validate them.
- Middleware is `RequestHandler`; after responding, `return` and do not call `next()`.
- Validate env vars once at startup with Zod and export a typed `env`; never cast `process.env`.
- Controllers handle HTTP, services hold rules, repositories hold data access; depend on interfaces and wire them in one place.

✅ Part 9a complete.

---

# Part 9b: Database, Auth, Validation & Error Handling

**Assumptions:** same project as 9a (Express 5, ESM, `env` from 9.7). Database code is shown for **Mongoose** and **Prisma**. Pick one for your projects; you must be able to read both.

---

## 9.9 Typing Mongoose models 🔴 MUST KNOW

**1. What it is**
Mongoose (MongoDB) has built-in TS support (v6+). You write an interface for the **document shape**, pass it to `Schema<T>`, and Mongoose types queries from it. Old projects use `@types/mongoose` or `Document`-extending interfaces; avoid that style in new code.

**3. Code**

```bash
npm install mongoose
```

```ts
// src/models/user.model.ts
import {
  Schema,
  model,
  type HydratedDocument,
  type InferSchemaType,
} from 'mongoose';

// Option A (recommended for beginners): write the interface yourself
export interface IUser {
  name: string;
  email: string;
  passwordHash: string;
  role: 'admin' | 'user';
  createdAt: Date;
  updatedAt: Date;
}

const userSchema = new Schema<IUser>(
  {
    name: { type: String, required: true, trim: true },
    email: { type: String, required: true, unique: true, lowercase: true },
    passwordHash: { type: String, required: true, select: false }, // hidden from queries by default
    role: { type: String, enum: ['admin', 'user'], default: 'user' },
  },
  { timestamps: true }, // adds createdAt/updatedAt; you still declare them in IUser
);

export const UserModel = model<IUser>('User', userSchema);
export type UserDoc = HydratedDocument<IUser>; // IUser + _id + save() + ...

// Option B: derive the type from the schema (no interface to maintain)
// type IUser2 = InferSchemaType<typeof userSchema>;
```

**Using it: `lean()` vs documents**

```ts
const doc = await UserModel.findOne({ email: 'a@x.com' }); // UserDoc | null
doc?.save(); // full Mongoose document

const plain = await UserModel.findById(id).lean();
// plain object, faster. _id is typed, but timestamps and defaults depend on your Mongoose version.
```

Check what `lean()` returns by hovering; the exact typing differs between Mongoose versions.

**Mapping to your domain type (important):** Mongo uses `_id` (an `ObjectId`), while your API and repository use `id: string`. Do not leak Mongoose documents out of the repository. Convert:

```ts
// src/repositories/mongo-user.repository.ts
import type {
  CreateUserInput,
  User,
  UserRepository,
} from './user.repository.js';
import { UserModel, type UserDoc } from '../models/user.model.js';

const toUser = (doc: UserDoc): User => ({
  id: doc._id.toString(),
  name: doc.name,
  email: doc.email,
  createdAt: doc.createdAt,
});

export class MongoUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    const doc = await UserModel.findById(id);
    return doc ? toUser(doc) : null;
  }
  async findAll(): Promise<User[]> {
    return (await UserModel.find()).map(toUser);
  }
  async create(data: CreateUserInput): Promise<User> {
    return toUser(
      await UserModel.create({ ...data, passwordHash: 'set-in-auth-service' }),
    );
  }
}
```

In 9a's `app.ts`, replace `new InMemoryUserRepository()` with `new MongoUserRepository()`. Nothing else changes. (The placeholder `passwordHash` is only to keep this demo compiling; real signup in 9.10 passes a real hash.)

**4. ❌ Wrong / ✅ Right**

```ts
const u = await UserModel.findById(id);
u.name;
// ❌ TS18047: 'u' is possibly 'null'.   → handle "not found" (throw NotFoundError)

userSchema.add({ nickname: String });
// ❌ Schema and interface drift: add the field to IUser too (TS cannot check the two match fully)
```

**5. Real-world use**
Connect once at startup: `await mongoose.connect(env.DATABASE_URL)` in `server.ts`, before `listen`.

**6. Common mistakes & errors**

- `TS2339: Property '_id' does not exist...` on plain objects. Use `HydratedDocument<IUser>` or `.lean()` and read the typed result.
- `Schema<IUser>` does not fully verify that `type: String` matches `name: string`. Mismatches can slip through to runtime.
- Forgetting `select: false` fields are `undefined` unless you query `.select("+passwordHash")`. The type still says `string`. Be careful.

---

## 9.10 Typing Prisma (and SQL basics) 🔴 MUST KNOW

**1. What it is**
Prisma is an ORM for SQL databases (PostgreSQL, MySQL, SQLite). You describe tables in `schema.prisma`; `prisma generate` creates **fully typed client code**. Types come from your schema, so there is no interface to maintain.

**SQL basics in 5 lines:** a table = rows and columns; `SELECT` reads, `INSERT` creates, `UPDATE` changes, `DELETE` removes; a **foreign key** links a row to another table (`Order.userId → User.id`); `JOIN` combines tables. Prisma writes this SQL for you.

**3. Code**

```bash
npm install @prisma/client
npm install -D prisma
npx prisma init
```

```prisma
// prisma/schema.prisma
model User {
  id           String   @id @default(uuid())
  name         String
  email        String   @unique
  passwordHash String
  role         Role     @default(USER)
  createdAt    DateTime @default(now())
  orders       Order[]
}
model Order {
  id     String @id @default(uuid())
  total  Int
  userId String
  user   User   @relation(fields: [userId], references: [id])
}
enum Role { ADMIN USER }
```

```bash
npx prisma migrate dev --name init   # creates tables + generates the client
```

```ts
// src/db.ts
import { PrismaClient, type User, type Prisma } from '@prisma/client';
export const prisma = new PrismaClient(); // create ONE instance for the whole app

// Types come generated: User, Prisma.UserCreateInput, Prisma.UserWhereInput ...
const user: User | null = await prisma.user.findUnique({
  where: { email: 'a@x.com' },
});

// `select`/`include` change the RETURN type automatically
const withOrders = await prisma.user.findMany({ include: { orders: true } });
withOrders[0]?.orders[0]?.total; // number
const names = await prisma.user.findMany({ select: { id: true, name: true } });
names[0]?.email;
// ❌ TS2339: Property 'email' does not exist on type '{ id: string; name: string; }'.

// Name a query's result type with the helpers from Part 5
type UserWithOrders = Prisma.UserGetPayload<{ include: { orders: true } }>;
```

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Passing req.body straight into Prisma: the client can inject fields like role: "ADMIN"
await prisma.user.create({ data: req.body });
// ✅ Validate with Zod (9.12), then pick exactly the fields you allow
await prisma.user.create({
  data: { name: input.name, email: input.email, passwordHash },
});
```

**5. Real-world use**
A `PrismaUserRepository implements UserRepository` mirrors the Mongo one. Prisma errors are `Prisma.PrismaClientKnownRequestError`; code `P2002` means unique constraint violated (e.g. duplicate email). Map it to a 409:

```ts
import { Prisma } from "@prisma/client";
catch (e) {
  if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === "P2002") throw new ConflictError("Email already used");
  throw e;
}
```

**6. Common mistakes & errors**

- `Module '"@prisma/client"' has no exported member 'User'.` You forgot `npx prisma generate` after editing the schema.
- Creating `new PrismaClient()` inside every request. It exhausts database connections.
- Prisma 6/7 changed generator settings and output paths. Check the docs for your version.

---

## 9.11 JWT authentication typing 🔴 MUST KNOW

**1. What it is**
A **JWT** is a signed token the client sends in `Authorization: Bearer <token>`. You verify the signature and read its payload. `jwt.verify` returns `string | JwtPayload`, so you must validate the payload shape.

**3. Code**

```bash
npm install jsonwebtoken bcryptjs
npm install -D @types/jsonwebtoken
```

(`bcryptjs` is pure JS; v3 ships its own types, older versions need `@types/bcryptjs`. `argon2` is another common choice.)

```ts
// src/auth/jwt.ts
import jwt from 'jsonwebtoken';
import { z } from 'zod';
import { env } from '../config/env.js';

const PayloadSchema = z.object({
  sub: z.string(), // user id (standard "subject" claim)
  role: z.enum(['admin', 'user']),
});
export type JwtPayload = z.infer<typeof PayloadSchema>;

export function signToken(payload: JwtPayload): string {
  return jwt.sign(payload, env.JWT_SECRET, { expiresIn: '15m' });
}

export function verifyToken(token: string): JwtPayload {
  const decoded: unknown = jwt.verify(token, env.JWT_SECRET); // throws if invalid/expired
  return PayloadSchema.parse(decoded); // validates the shape, so no `as`
}
```

Note: `expiresIn` is typed as a number or a specific string format in newer `@types/jsonwebtoken`. If you pass `env.JWT_EXPIRES_IN` (a plain `string`) you may get `TS2769`. Use a literal as above, or cast to `jwt.SignOptions["expiresIn"]`. This varies by `@types` version.

```ts
// src/middleware/authenticate.ts
import type { RequestHandler } from 'express';
import { verifyToken } from '../auth/jwt.js';
import { UnauthorizedError } from '../errors/app-error.js';

export const authenticate: RequestHandler = (req, _res, next) => {
  const header = req.headers.authorization; // string | undefined
  if (!header?.startsWith('Bearer ')) throw new UnauthorizedError();
  try {
    const payload = verifyToken(header.slice(7));
    req.user = { id: payload.sub, role: payload.role }; // typed via 7.7 augmentation
    next();
  } catch {
    throw new UnauthorizedError('Invalid or expired token');
  }
};
```

(Sync `throw` inside Express middleware is caught by Express in both v4 and v5.)

Password handling, in the auth service:

```ts
import bcrypt from 'bcryptjs';
const hash = await bcrypt.hash(plainPassword, 12);
const ok = await bcrypt.compare(plainPassword, user.passwordHash); // boolean
```

**4. ❌ Wrong / ✅ Right**

```ts
const payload = jwt.verify(token, env.JWT_SECRET) as { sub: string }; // ❌ unchecked claim
const payload2 = verifyToken(token); // ✅ Zod-validated
```

**5. Real-world use**
Login returns `{ token }`; `authenticate` guards routes; `requireRole("admin")` (9.6) checks roles. Refresh tokens, cookies (`httpOnly`) and token rotation are real-world needs beyond this note. Keep `JWT_SECRET` out of git.

**6. Common mistakes**

- Storing sensitive data in the payload. JWTs are signed, **not encrypted**; anyone can read them.
- Comparing passwords with `===`. Always use the hash library's `compare`.
- Returning `passwordHash` to clients. Strip it in the mapper (`toUser` above omits it).

**7. Interview tip**
_"Why validate the JWT payload?"_ `jwt.verify` returns a loose type, and a correctly signed token can still have unexpected content (old tokens, other services).

---

## 9.12 DTOs and Zod validation middleware 🔴 MUST KNOW

**1. What it is**
A **DTO** (Data Transfer Object) is the typed shape of data crossing a boundary, like a request body. In this stack, a Zod schema **is** your DTO, and `z.infer` gives its type.

**3. Code**

```ts
// src/schemas/user.schema.ts
import { z } from 'zod';

export const CreateUserSchema = z.object({
  name: z.string().trim().min(1),
  email: z.string().email().toLowerCase(),
  password: z.string().min(8),
});
export type CreateUserDto = z.infer<typeof CreateUserSchema>;

export const IdParamsSchema = z.object({ id: z.string().min(1) });

export const ListQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});
```

```ts
// src/middleware/validate.ts
import type { RequestHandler } from 'express';
import type { z } from 'zod';
import { ValidationError } from '../errors/app-error.js';

interface Schemas {
  body?: z.ZodType;
  params?: z.ZodType;
  query?: z.ZodType;
}

export const validate =
  (schemas: Schemas): RequestHandler =>
  (req, _res, next) => {
    for (const key of ['body', 'params', 'query'] as const) {
      const schema = schemas[key];
      if (!schema) continue;
      const result = schema.safeParse(req[key]);
      if (!result.success) {
        throw new ValidationError('Invalid request', result.error.issues);
      }
      if (key === 'query') {
        res.locals.query = result.data; // see note: req.query is read-only in Express 5
      } else {
        req[key] = result.data; // replace with parsed (stripped, coerced) data
      }
    }
    next();
  };
```

**Express 5 caveat:** `req.query` is a getter and cannot be reassigned, so the code above would not compile as written for `query`. The clean pattern is to store parsed query in `res.locals` or just parse again in the controller. A simpler version many teams use:

```ts
// Simplest safe version: validate body only, and parse query inside the controller
export const validateBody =
  <S extends z.ZodType>(schema: S): RequestHandler =>
  (req, _res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success)
      throw new ValidationError('Invalid request', result.error.issues);
    req.body = result.data; // body is writable
    next();
  };

// In a controller: parse query with the same schema
const { page, limit } = ListQuerySchema.parse(req.query);
```

(I wrote the first block to show the idea; the second is the version I recommend you actually use. In the first block `res` is also not in scope. This is the sort of thing the compiler catches immediately: `TS2304: Cannot find name 'res'`.)

Typed controller using the validated body:

```ts
create = async (
  req: Request<unknown, unknown, CreateUserDto>,
  res: Response,
): Promise<void> => {
  const user = await this.service.create(req.body); // validated by middleware before this runs
  res.status(201).json({ success: true, data: user });
};
// router.post("/", validateBody(CreateUserSchema), controller.create);
```

The `Request<..., CreateUserDto>` generic is still a **claim**, but now the middleware makes it true. Keep the schema and the generic in sync by always deriving the DTO type from the schema.

**4. ❌ Wrong / ✅ Right**

```ts
router.post('/', controller.create); // ❌ trusts the client
router.post('/', validateBody(CreateUserSchema), controller.create); // ✅
```

**5. Real-world use**
Every POST/PUT/PATCH route. `.strict()` on schemas rejects unknown fields; the default strips them, which blocks mass-assignment attacks (`role: "admin"`).

**6. Common mistakes**
Using the validated DTO type in the service layer when the service needs a different shape (for example `password` becomes `passwordHash`). Define a separate `CreateUserInput` for the service, as in 9a.

---

## 9.13 Centralized error handler 🔴 MUST KNOW

**1. What it is**
One middleware, registered **last**, that turns any thrown error into a consistent JSON response. Express recognizes it by its **four parameters**.

**3. Code**

```ts
// src/errors/app-error.ts (from 8.4, plus the extras used in this part)
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode = 500,
    public readonly code = 'INTERNAL_ERROR',
    public readonly details?: unknown,
  ) {
    super(message);
    this.name = new.target.name;
  }
}
export class NotFoundError extends AppError {
  constructor(resource: string, id: string | number) {
    super(`${resource} ${id} not found`, 404, 'NOT_FOUND');
  }
}
export class ValidationError extends AppError {
  constructor(message: string, details: unknown) {
    super(message, 400, 'VALIDATION_ERROR', details);
  }
}
export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}
export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, 'CONFLICT');
  }
}
```

```ts
// src/middleware/error-handler.ts
import type { ErrorRequestHandler } from 'express';
import { AppError } from '../errors/app-error.js';
import { env } from '../config/env.js';

export const errorHandler: ErrorRequestHandler = (
  err: unknown,
  _req,
  res,
  _next,
) => {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      success: false,
      error: { code: err.code, message: err.message, details: err.details },
    });
    return;
  }

  // Unexpected: log everything, tell the client nothing sensitive
  console.error(err);
  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_ERROR',
      message:
        env.NODE_ENV === 'production' ? 'Something went wrong' : String(err),
    },
  });
};
```

```ts
// src/app.ts: registered AFTER all routes
app.use('/users', createUserRouter(userController));
app.use((_req, _res, next) => next(new NotFoundError('Route', 'unknown'))); // 404 for unknown URLs
app.use(errorHandler);
```

**4. ❌ Wrong / ✅ Right**

```ts
const handler: ErrorRequestHandler = (err, req, res) => {};
// ❌ Express sees only 3 params, treats it as normal middleware, and errors skip it.
// ✅ always declare all four, even if `_next` is unused (prefix with _ to satisfy the linter)
```

`err` is `any` by default in `ErrorRequestHandler`; annotating it `unknown` forces safe narrowing.

**5. Real-world use**
Also map third-party errors here: `Prisma P2002 → 409`, Mongoose `ValidationError → 400`, `jwt.TokenExpiredError → 401`, malformed JSON from `express.json()` (`SyntaxError` with `status === 400`).

**6. Common mistakes**

- Registering the handler before the routes.
- Leaking `err.stack` or database messages to clients in production.
- Express 4: async errors never reach it unless you use `asyncHandler` or call `next(err)`.

**7. Interview tip**
_"How does Express know a middleware is an error handler?"_ It has four parameters.

---

## 9.14 Typed REST response format 🔴 MUST KNOW

**1. What it is**
One response envelope for every endpoint, defined **once** and shared with the frontend (Part 11). It reuses `ApiResponse<T>` from 4.2.

**3. Code**

```ts
// src/types/api.ts
export interface ApiErrorBody {
  code: string;
  message: string;
  details?: unknown;
}

export type ApiResponse<T> =
  | { success: true; data: T; meta?: PageMeta }
  | { success: false; error: ApiErrorBody };

export interface PageMeta {
  page: number;
  limit: number;
  total: number;
}

// src/utils/respond.ts: typed helper so handlers cannot send the wrong shape
import type { Response } from 'express';
export function sendOk<T>(
  res: Response<ApiResponse<T>>,
  data: T,
  status = 200,
  meta?: PageMeta,
): void {
  res.status(status).json({ success: true, data, ...(meta ? { meta } : {}) });
}

// Controller
getById = async (
  req: Request<{ id: string }>,
  res: Response<ApiResponse<User>>,
): Promise<void> => {
  sendOk(res, await this.service.getById(req.params.id));
};
res.json({ success: true, data: 123 });
// ❌ TS2345: ... 'number' is not assignable to type 'User'   (when res is Response<ApiResponse<User>>)
```

**4. ❌ Wrong / ✅ Right**
Inconsistent shapes (`{ users: [...] }` on one route, `[...]` on another, `{ message }` for errors elsewhere) force the frontend into guesswork. One envelope plus the discriminant `success` (Part 3) lets the client narrow cleanly.

**5. Real-world use**
Pagination: `sendOk(res, items, 200, { page, limit, total })`. Use proper HTTP status codes too (201 created, 204 no content, 400/401/403/404/409/422, 500).

**6. Common mistake**
Typing `res` as `Response<ApiResponse<User>>` helps only where you actually use `res.json` through it. Error responses from the handler (9.13) are separate; keep both shapes matching `ApiResponse`.

---

### 📌 Key Takeaways

- Mongoose: interface + `Schema<IUser>`; map documents to domain objects (`_id` → `id`) inside the repository. Prisma: types are generated from `schema.prisma`; run `generate` after changes and share one `PrismaClient`.
- Never pass `req.body` straight to the database; validate, then pick allowed fields.
- `jwt.verify` is loosely typed, so validate its payload with Zod. JWTs are signed, not encrypted; never put secrets in them.
- A Zod schema is your DTO. Derive the type with `z.infer`, validate in middleware, and remember Express 5's `req.query` is read-only.
- The error handler needs four parameters, goes last, maps `AppError` and known library errors, and hides internals in production.
- One typed `ApiResponse<T>` envelope for every endpoint keeps the frontend simple.

✅ Part 9b complete.

Correction: in 9.12, the first `validate` block I showed has two real mistakes (`res` is not in scope, and `req.query` can't be reassigned in Express 5). Use the `validateBody` version below it, and parse query inside the controller.

---

# Part 9c: Testing, NestJS Intro & Deployment

**Assumptions:** same project as 9a/9b (Express 5, ESM, Zod `env`). Testing uses **Vitest**, which works with ESM and TypeScript with little setup. Jest is covered as the legacy option you will meet in older codebases.

---

## 9.15 Testing with Vitest (and Jest + ts-jest) 🔴 MUST KNOW

**1. What it is**
Automated tests that call your code and check the results. Layers 9a built (controller, service, repository) make this easy:

- **Unit tests:** one service with a fake repository, no HTTP.
- **Integration tests:** real Express `app` with `supertest`, no open port.

**2. Why it exists**
TS proves types match. Tests prove behavior is right (the right status code, the right error for a missing user).

**3. Code**

```bash
npm install -D vitest supertest @types/supertest
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    // env.ts (9.7) calls process.exit(1) on bad env, so give tests valid values:
    env: {
      NODE_ENV: 'test',
      DATABASE_URL: 'test',
      JWT_SECRET: 'test-secret-that-is-at-least-32-chars-long',
    },
  },
});
```

```ts
// src/services/user.service.test.ts: unit test with the in-memory repo
import { describe, it, expect } from 'vitest';
import { UserService } from './user.service.js';
import { InMemoryUserRepository } from '../repositories/user.repository.js';
import { NotFoundError } from '../errors/app-error.js';

describe('UserService', () => {
  it('creates and finds a user', async () => {
    const service = new UserService(new InMemoryUserRepository());
    const created = await service.create({
      name: 'Asha',
      email: 'asha@example.com',
    });
    const found = await service.getById(created.id);
    expect(found.email).toBe('asha@example.com');
  });

  it('throws NotFoundError for an unknown id', async () => {
    const service = new UserService(new InMemoryUserRepository());
    await expect(service.getById('missing')).rejects.toThrow(NotFoundError);
  });
});
```

```ts
// src/app.test.ts: integration test, no real port opened
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { app } from './app.js';

describe('GET /health', () => {
  it('returns ok', async () => {
    const res = await request(app).get('/health').expect(200);
    expect(res.body).toEqual({ status: 'ok' }); // res.body is `any`: validate shape in the test
  });
});
```

**Typed mocks (when a fake class is overkill)**

```ts
import { vi, type Mocked } from 'vitest';
import type { UserRepository } from '../repositories/user.repository.js';

const repo: Mocked<UserRepository> = {
  findById: vi.fn(),
  findAll: vi.fn(),
  create: vi.fn(),
};
repo.findById.mockResolvedValue(null); // typed: must match Promise<User | null>
repo.findById.mockResolvedValue({ id: 1 });
// ❌ TS2345: Argument of type '{ id: number; }' is not assignable to parameter of type 'User | null'.
const service = new UserService(repo);
```

The mock is checked against the interface. If you add a method to `UserRepository`, this object stops compiling until you add it. This is the payoff of depending on interfaces (9.8).

**Jest + ts-jest (what you will see in older projects)**

```bash
npm install -D jest ts-jest @types/jest
npx ts-jest config:init
```

```js
// jest.config.js
export default { preset: 'ts-jest', testEnvironment: 'node' };
```

Equivalents: `vi.fn()` is `jest.fn()`, `Mocked<T>` is `jest.Mocked<T>`, and `describe/it/expect` come from globals or `@jest/globals`. ts-jest **does** type check test files (slower, but catches errors). Jest with native ESM and `.js` import extensions needs extra setup (`moduleNameMapper`, experimental flags). This is the main reason newer projects pick Vitest. Config details vary by version, so check the docs.

**4. ❌ Wrong / ✅ Right**

```ts
// ❌ Tests compiled into dist/ and shipped (they sit in src/ and `include` covers src)
// ✅ Use a build config that excludes them (see 9.17)

// ❌ A "global" mock cast to hide a mismatch
const repo = {} as UserRepository; // compiles; crashes when a method is called
// ✅ Mocked<UserRepository>, or a complete in-memory fake
```

**5. Real-world use**
CI runs `npm run typecheck && npm run lint && npm test`. Test the service layer heavily (rules live there), controllers lightly through `supertest`, and repositories against a real test database (or an in-memory one such as `mongodb-memory-server`).

**6. Common mistakes & errors**

- `TS2582: Cannot find name 'describe'` (or `it`, `expect`) → import them from `vitest`, or enable globals: set `test.globals: true` and add `"types": ["vitest/globals"]` to tsconfig. Importing explicitly is the clearer choice.
- Process exits at import with "Invalid environment variables" → your `env` schema ran in tests without values. Provide them via config (above).
- Vitest, like `tsx`, **does not type check** (Part 0). Tests can pass while `tsc --noEmit` fails. Run both.
- Tests share state: `app.ts` creates one in-memory repo at import, so data leaks between tests. Build the app in a `createApp(deps)` function if you need isolation.

**7. Interview tip**
_"How do you test an Express + TS backend?"_ Unit-test services with fake or mocked repositories, integration-test routes with `supertest` against the exported `app`, and keep `app` separate from `listen`.

---

## 9.16 Intro to NestJS 🟡 GOOD TO KNOW

**1. What it is**
NestJS is an opinionated Node framework (running on Express or Fastify underneath) built around TypeScript, **decorators** (6.10) and **dependency injection (DI)**. It gives you the structure you built by hand in 9.8 as framework features.

**2. Why it exists**
Large teams want one standard layout instead of each project inventing its own. Many companies hiring TS backend developers use it, so you should be able to read it.

**3. What changes**

| Your Express project                            | NestJS equivalent                                                                                          |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `router.get("/:id", ...)`                       | `@Get(":id")` on a controller method                                                                       |
| Controller class, manual wiring in `app.ts`     | `@Controller("users")` + `@Module({ controllers, providers })`                                             |
| `new UserService(repo)` in the composition root | Nest creates and injects it from the constructor types                                                     |
| `validateBody(Schema)` middleware               | **Pipes** (`ValidationPipe` with `class-validator` DTO classes; Zod via a custom pipe or a helper library) |
| Auth middleware                                 | **Guards** (`@UseGuards(AuthGuard)`)                                                                       |
| Error handler middleware                        | **Exception filters**; throw `NotFoundException` etc.                                                      |
| Middleware                                      | Middleware, **interceptors** (wrap handlers)                                                               |

```ts
// users.service.ts
@Injectable() // "Nest may inject this class"
export class UsersService {
  private readonly users = new Map<string, User>();

  findOne(id: string): User {
    const user = this.users.get(id);
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }
}

// users.controller.ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {} // injected by type

  @Get(':id')
  findOne(@Param('id') id: string): User {
    return this.usersService.findOne(id); // return a value; Nest sends the JSON
  }

  @Post()
  create(@Body() dto: CreateUserDto): User {
    /* ... */
  }
}

// create-user.dto.ts: a CLASS, not an interface (see below)
export class CreateUserDto {
  @IsString() @IsNotEmpty() name!: string;
  @IsEmail() email!: string;
}

// users.module.ts
@Module({ controllers: [UsersController], providers: [UsersService] })
export class UsersModule {}
```

**Why DTOs are classes here:** types are erased (Part 0), but class decorators emit **runtime metadata**. Nest reads it to validate. An `interface` DTO would vanish and nothing could be validated. This is the same reason `emitDecoratorMetadata` exists.

**tsconfig additions (legacy decorators, see 6.10):**

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

Nest's generated project sets these (and other options) for you. Confirm the current requirements in the NestJS docs for your version.

**4. ❌ Wrong / ✅ Right**

```ts
import type { UsersService } from "./users.service";   // ❌ type-only import of an injected class
constructor(private readonly usersService: UsersService) {}
// ❌ TS1272: A type referenced in a decorated signature must be imported with 'import type'
//    or a namespace import when 'isolatedModules' and 'emitDecoratorMetadata' are enabled.
// (and without the error, DI fails at runtime: Nest can't resolve the dependency)
import { UsersService } from "./users.service";        // ✅ normal import: Nest needs the real class
```

The error message wording is confusing, and the fix depends on your flags. Rule: **injected classes need a real (value) import**, not `import type`.

**5. Real-world use**
Nest fits teams and long-lived APIs. For a small project or a fresher portfolio, plain Express + TS (what you just built) shows you understand the layers; add Nest as a second project.

**6. Common mistakes**

- Forgetting to list a class in `providers`. Runtime error: `Nest can't resolve dependencies of the UsersController (?)`.
- Treating Nest's decorators as magic. Each one is a label that a framework reads (6.10).
- Mixing `class-validator` DTOs and Zod without a plan. Pick one validation approach per project.

**7. Interview tip**
_"What does NestJS add on top of Express?"_ Modules, DI, decorators for routing and validation, and a standard structure (controllers, providers, guards, pipes, filters).

---

## 9.17 Deployment build steps 🔴 MUST KNOW

**1. What it is**
Turning TS source into something a server runs with plain `node`. The production server never runs `tsx` or `tsc` on the fly (Part 0.3).

**3. The steps**

1. Type check and test (`tsc --noEmit`, `vitest run`).
2. Compile with `tsc` to `dist/`.
3. Install **production** dependencies only.
4. Run `node dist/server.js` with env vars injected by the host.

```json
// tsconfig.build.json: same config, but exclude tests from the output
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "dist", "src/**/*.test.ts"]
}
```

```json
// package.json scripts
{
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "start": "node --enable-source-maps dist/server.js",
    "test": "vitest run",
    "ci": "npm run typecheck && npm run lint && npm test && npm run build"
  }
}
```

`--enable-source-maps` makes stack traces point at your `.ts` lines (needs `"sourceMap": true`, already in the Part 0.6 config).

**Dockerfile (multi-stage, common in real jobs)**

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                      # installs devDependencies too (typescript, @types/*)
COPY . .
RUN npm run build               # with Prisma: also `npx prisma generate` before this

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev           # no typescript, no @types: not needed at runtime
COPY --from=build /app/dist ./dist
# With Prisma, also copy the generated client / prisma folder, or run `prisma generate` here
CMD ["node", "--enable-source-maps", "dist/server.js"]
```

This is a minimal pattern. Exact Prisma and package-manager steps depend on your versions.

**Hosting notes:** platforms like Render, Railway, Fly.io or AWS take a build command (`npm ci && npm run build`) and a start command (`npm start`). Set env vars in their dashboard, since `.env` is not committed.

**4. ❌ Wrong / ✅ Right**

```
// ❌ "start": "tsx src/server.ts" in production: slower startup, no type check, dev tool in prod
// ✅ "start": "node dist/server.js"

// ❌ Alias imports ("@/db") left in dist/*.js  → Error: Cannot find module '@/db' (7.9)
// ✅ run tsc-alias after tsc, or use relative imports

// ❌ Putting typescript only in devDependencies and then running `tsc` after `npm ci --omit=dev`
// ✅ build in one stage (with dev deps), run in another (without them)
```

**5. Real-world use**
A CI pipeline (GitHub Actions) runs `npm ci` then `npm run ci`, and deploys only if everything passes. Add a `/health` endpoint (9.1) for the host's health checks.

**6. Common mistakes & errors**

- `dist/src/server.js` instead of `dist/server.js` → missing or wrong `rootDir` (TS6059, Part 0.4).
- `Cannot find module './app'` at runtime in ESM → the import lacked `.js` (7.1). `tsx` can hide this locally; `tsc` flags it (TS2835), so run the build in CI.
- `ERR_MODULE_NOT_FOUND` for a package → it was in `devDependencies` but needed at runtime. Move it to `dependencies`.
- Type packages (`@types/*`) are dev-only. If runtime code imports a package, that package is a real dependency.

**7. Interview tip**
_"How do you deploy a TS Node app?"_ Compile with `tsc` in CI, ship `dist/` plus production dependencies, run with `node`, inject config through environment variables.

---

### 📌 Key Takeaways

- Test services with in-memory or `Mocked<Interface>` repositories, and routes with `supertest` against the exported `app`. Vitest and `tsx` do not type check, so run `tsc --noEmit` too.
- Give tests valid env values, or your Zod `env` module will exit the process on import.
- Typed mocks fail to compile when the interface changes. Avoid `{} as Interface`.
- NestJS formalizes the same layers with modules, DI, decorators, pipes, guards and filters. DTOs are classes (decorators emit metadata) and injected classes need real imports.
- Production runs compiled JS (`node dist/server.js`). Exclude tests from the build, install production dependencies only, and handle path aliases and Prisma generation as explicit build steps.
- Run `typecheck`, lint, tests and build in CI before every deploy.

✅ Part 9c complete.
