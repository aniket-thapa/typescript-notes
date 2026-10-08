# Part 6: Classes & OOP

Classes are JavaScript (ES2015+) features. TS adds types, access modifiers and a few extras on top. In Node backends you will meet them in services, error classes and NestJS. In React, function components have mostly replaced classes, but old class components still exist.

---

## 6.1 Class typing 🔴 MUST KNOW

**1. What it is**
A class declares its **properties** (with types), a **constructor**, and **methods**. Every property must be declared in the class body.

**2. Why it exists**
In JS you can add properties anywhere (`this.x = 1`), which hides bugs. TS makes you declare them upfront, so the shape of every instance is known.

**3. Code**

```ts
class User {
  id: number;
  name: string;
  email?: string; // optional
  isActive = true; // type inferred as boolean

  constructor(id: number, name: string) {
    this.id = id;
    this.name = name;
  }

  describe(): string {
    return `${this.name} (#${this.id})`;
  }
}

const u = new User(1, 'Asha');
u.describe();
new User(1);
// ❌ TS2554: Expected 2 arguments, but got 1.
```

**4. ❌ Wrong / ✅ Right**
`strictPropertyInitialization` (part of `strict`) requires every property to be set by the end of the constructor.

```ts
class Session {
  token: string;
  // ❌ TS2564: Property 'token' has no initializer and is not definitely assigned in the constructor.
}

class SessionOk {
  token: string | undefined; // ✅ honest: may be missing
  userId = 0; // ✅ default value
  secret!: string; // ⚠️ "definitely assigned" promise (like ! from Part 1)
}
```

Use `!` only when something outside the constructor is guaranteed to set it (a DI framework, a lifecycle hook). It is an unchecked promise.

**5. Real-world use**
Service classes (`UserService`), custom errors (`HttpError`), and wrappers around SDKs (`Mailer`, `PaymentClient`).

**6. Common mistakes**

- Classes are **structurally typed** too (Part 3). Any object with `id`, `name` and `describe` fits the type `User`, even if it is not made with `new User`. The exception is classes with `private`/`protected` members (6.2).
- A class name is both a **type** and a **value**. `x: User` is the type; `new User()` uses the value. Interfaces are types only.

**7. Interview tip**
_"What does `strictPropertyInitialization` do?"_ It forces every class property to be initialized or typed as possibly `undefined`.

---

## 6.2 Access modifiers: `public`, `private`, `protected`, `#private` 🔴 MUST KNOW

**1. What it is**

| Modifier           | Accessible from              | Enforced at runtime?              |
| ------------------ | ---------------------------- | --------------------------------- |
| `public` (default) | Anywhere                     | n/a                               |
| `protected`        | The class and its subclasses | **No** (compile-time only)        |
| `private`          | The class only               | **No** (compile-time only)        |
| `#name`            | The class only               | **Yes** (real JavaScript feature) |

**2. Why it exists**
To hide internal state so outside code cannot corrupt it, such as changing a `balance` directly.

**3. Code**

```ts
class BankAccount {
  private balance = 0; // TS-only privacy
  protected owner: string; // visible to subclasses
  #pin: string; // truly private in JS

  constructor(owner: string, pin: string) {
    this.owner = owner;
    this.#pin = pin;
  }

  deposit(amount: number): void {
    this.balance += amount;
  }
  checkPin(pin: string): boolean {
    return this.#pin === pin;
  }
}

const acc = new BankAccount('Asha', '1234');
acc.balance;
// ❌ TS2341: Property 'balance' is private and only accessible within class 'BankAccount'.
acc.owner;
// ❌ TS2445: Property 'owner' is protected and only accessible within class 'BankAccount' and its subclasses.
acc.#pin;
// ❌ TS18013: Property '#pin' is not accessible outside class 'BankAccount' because it has a private identifier.

(acc as any).balance; // ⚠️ compiles and works at runtime: `private` is erased
```

**4. ❌ Wrong / ✅ Right**

```ts
class Config {
  private apiKey = 'secret'; // ❌ NOT secure: visible in JS, console.log and JSON.stringify
  #apiKey = 'secret'; // ✅ hidden at runtime too (but still not "encryption")
}
```

Neither one protects secrets from someone who can run your code. Privacy is a design tool, not a security tool.

**5. Real-world use**
`private` for injected dependencies and internal helpers; `protected` in base classes (6.7); `#private` in new code that must be truly hidden. Many teams use only `private` (it is more common in older code and works with decorators). `#private` needs target ES2015+ and adds slight overhead on older targets. This is a team-opinion topic.

**6. Common mistakes**

- Classes with `private` or `protected` members are compared **nominally**: two classes with identical private fields are not assignable to each other. This is the one place TS is not purely structural.
- `private` members are still visible in `JSON.stringify` and `Object.keys`. `#private` ones are not.

**7. Interview tip**
_"`private` vs `#private`?"_ `private` is erased and only checked by TS; `#private` is real JS privacy enforced at runtime.

---

## 6.3 `readonly` and parameter properties 🔴 MUST KNOW

**1. What it is**

- `readonly`: assignable only at declaration or in the constructor.
- **Parameter property:** adding a modifier to a constructor parameter (`private`, `public`, `protected`, `readonly`) declares the property **and** assigns it in one step.

**2. Why it exists**
The "declare, take in constructor, assign" routine is repetitive. Parameter properties remove it.

**3. Code**

```ts
// Long way
class UserService {
  private readonly repo: UserRepository;
  private readonly logger: Logger;
  constructor(repo: UserRepository, logger: Logger) {
    this.repo = repo;
    this.logger = logger;
  }
}

// Same thing, shorter (you will see this everywhere in Node/NestJS code)
class UserServiceShort {
  constructor(
    private readonly repo: UserRepository,
    private readonly logger: Logger,
  ) {}

  async find(id: string) {
    this.logger.info(`finding ${id}`);
    return this.repo.findById(id);
  }
}

class Point {
  constructor(
    readonly x: number,
    readonly y: number,
  ) {}
}
const p = new Point(1, 2);
p.x = 5;
// ❌ TS2540: Cannot assign to 'x' because it is a read-only property.
```

(`UserRepository` and `Logger` are interfaces you define; see Part 4's repository.)

**4. ❌ Wrong / ✅ Right**

```ts
class Bad {
  constructor(name: string) {}
  greet() {
    return this.name;
  }
  // ❌ TS2339: Property 'name' does not exist on type 'Bad'.
}
class Good {
  constructor(public name: string) {} // ✅ modifier makes it a property
  greet() {
    return this.name;
  }
}
```

**5. Real-world use**
Dependency injection: pass dependencies through the constructor as `private readonly`. This is the backbone of NestJS (6.11).

**6. Common mistakes**
Parameter properties are **not** plain "erasable" syntax: they generate runtime code. Tools or flags that only strip types (Node's built-in type stripping, TS 5.8's `erasableSyntaxOnly`) reject them, as they do enums. Check your tooling if you see this error.

**7. Interview tip**
_"What is a parameter property?"_ A constructor parameter with an access modifier or `readonly`; it creates and assigns the property automatically.

---

## 6.4 `static` members 🟡 GOOD TO KNOW

**1. What it is**
Properties and methods that belong to the **class itself**, not to instances. Access them as `ClassName.member`.

**3. Code**

```ts
class IdGenerator {
  private static counter = 0;

  static next(): number {
    return ++IdGenerator.counter;
  }
}
IdGenerator.next(); // 1

class Order {
  static readonly MAX_ITEMS = 50;
  constructor(public items: string[]) {}

  static fromJson(json: string): Order {
    // "named constructor" / factory
    const raw: { items: string[] } = JSON.parse(json);
    return new Order(raw.items);
  }
}
const o = new Order([]);
o.MAX_ITEMS;
// ❌ TS2576: Property 'MAX_ITEMS' does not exist on type 'Order'. Did you mean to access the static member 'Order.MAX_ITEMS' instead?
```

**5. Real-world use**
Factory methods (`User.fromDocument(doc)`), constants, and the Singleton pattern. In modern code a plain module-level function or constant is often simpler than a class with only statics.

**6. Common mistake**
Static members cannot use the class's type parameters (`TS2302`, see 4.6).

---

## 6.5 Getters and setters 🟡 GOOD TO KNOW

**1. What it is**
Methods that look like properties: `get` runs on read, `set` runs on write.

**3. Code**

```ts
class Temperature {
  private _celsius = 0;

  get celsius(): number {
    return this._celsius;
  }
  set celsius(value: number) {
    if (value < -273.15) throw new Error('Below absolute zero');
    this._celsius = value;
  }
  get fahrenheit(): number {
    // getter only = read-only property
    return (this._celsius * 9) / 5 + 32;
  }
}

const t = new Temperature();
t.celsius = 25;
t.fahrenheit = 100;
// ❌ TS2540: Cannot assign to 'fahrenheit' because it is a read-only property.
```

**6. Common mistakes**

- A getter and setter for the same name must be compatible. Before TS 5.1 the getter type had to be assignable to the setter type; 5.1+ allows unrelated types. Version-dependent.
- Getters run on every read. Do not put slow work in them.
- Getters are not own enumerable properties, so `JSON.stringify(instance)` omits them.

---

## 6.6 Inheritance and `override` 🔴 MUST KNOW

**1. What it is**
`class Child extends Parent` reuses the parent's properties and methods. `super(...)` calls the parent constructor; `super.method()` calls a parent method.

**3. Code**

```ts
class Animal {
  constructor(public name: string) {}
  speak(): string {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  constructor(
    name: string,
    public breed: string,
  ) {
    super(name); // must come before using `this`
  }
  override speak(): string {
    // `override` = "I am replacing a parent method"
    return `${super.speak()}: Woof`;
  }
}
```

**`override` and `noImplicitOverride`:** with `"noImplicitOverride": true`, a method that overrides a parent method **must** carry `override`. It is not part of `strict`, but it is a good flag to enable.

```ts
class Cat extends Animal {
  speak() {
    return 'Meow';
  }
  // ❌ TS4114: This member must have an 'override' modifier because it overrides a member in the base class 'Animal'.
  purr() {}
}
class Cow extends Animal {
  override moo() {}
  // ❌ TS4113: This member cannot have an 'override' modifier because it is not declared in the base class 'Animal'.
}
```

That second error catches the bug where the parent renames a method and your override silently stops overriding.

**4. ❌ Wrong / ✅ Right**

```ts
class Bad extends Animal {
  constructor(name: string) {
    this.name = name;
    // ❌ TS17009: 'super' must be called before accessing 'this' in the constructor of a derived class.
  }
}
class NoSuper extends Animal {
  constructor() {}
  // ❌ TS2377: Constructors for derived classes must contain a 'super' call.
}
```

**5. Real-world use: custom HTTP errors for Express**

```ts
class HttpError extends Error {
  constructor(
    public readonly statusCode: number,
    message: string,
  ) {
    super(message);
    this.name = 'HttpError';
  }
}
class NotFoundError extends HttpError {
  constructor(resource: string) {
    super(404, `${resource} not found`);
  }
}

// In an error-handling middleware:
// if (err instanceof HttpError) res.status(err.statusCode).json({ error: err.message });
```

With `target` ES2015 or higher, `instanceof` works for subclasses of `Error`. Only very old ES5 targets need `Object.setPrototypeOf(this, new.target.prototype)`. You will see it in older code. Part 8 builds on this.

**6. Common mistakes**

- Overusing inheritance. Deep trees (`A → B → C → D`) are hard to change. Prefer **composition** (pass collaborators in via the constructor) unless the "is-a" relationship is real.
- With `useDefineForClassFields` (default `true` when `target` is ES2022 or higher), redeclaring a parent's field in a child without an initializer resets it to `undefined`. Use `declare field: Type;` in the child if you only want to narrow its type.

**7. Interview tip**
_"Inheritance or composition?"_ Composition by default; inheritance for true "is-a" relationships and shared base behavior.

---

## 6.7 Abstract classes 🔴 MUST KNOW

**1. What it is**
An `abstract` class cannot be instantiated. It can contain real code plus `abstract` members that subclasses **must** implement.

**2. Why it exists**
A template: shared logic in the base, specific steps left to children.

**3. Code**

```ts
abstract class BaseNotifier {
  async notify(userId: string, message: string): Promise<void> {
    const formatted = this.format(message); // shared flow...
    await this.send(userId, formatted); // ...specific step supplied by child
    console.log(`Notified ${userId}`);
  }

  protected abstract format(message: string): string;
  protected abstract send(userId: string, body: string): Promise<void>;
}

class EmailNotifier extends BaseNotifier {
  protected format(message: string): string {
    return `<p>${message}</p>`;
  }
  protected async send(userId: string, body: string): Promise<void> {
    /* call email provider */
  }
}

new BaseNotifier();
// ❌ TS2511: Cannot create an instance of an abstract class.

class BrokenNotifier extends BaseNotifier {}
// ❌ TS2515: Non-abstract class 'BrokenNotifier' does not implement inherited abstract member 'format' from class 'BaseNotifier'.
```

**5. Real-world use**
Base controllers/services, payment providers (`abstract charge()`), and the "template method" pattern.

**7. Interview tip**
_"Abstract class vs interface?"_ An abstract class can hold implementation and exists at runtime; an interface is only a type and is erased.

---

## 6.8 `implements` 🔴 MUST KNOW

**1. What it is**
`class X implements Y` makes the compiler **check** that `X` has everything the interface `Y` requires. It adds nothing to the class.

**2. Why it exists**
It makes a class honor a contract, so you can swap implementations (real vs fake).

**3. Code**

```ts
interface PaymentGateway {
  charge(amountInPaise: number, currency: string): Promise<string>;
}

class StripeGateway implements PaymentGateway {
  async charge(amountInPaise: number, currency: string): Promise<string> {
    return `stripe_${amountInPaise}${currency}`;
  }
}

class BrokenGateway implements PaymentGateway {}
// ❌ TS2420: Class 'BrokenGateway' incorrectly implements interface 'PaymentGateway'.
//    Property 'charge' is missing in type 'BrokenGateway' but required in type 'PaymentGateway'.

class Checkout {
  constructor(private readonly gateway: PaymentGateway) {} // depends on the contract
}
new Checkout(new StripeGateway());
```

A class may implement several interfaces: `class A implements B, C`. It may `extend` only **one** class.

**4. ❌ Wrong / ✅ Right**
`implements` does **not** copy members or infer parameter types:

```ts
interface Logger {
  log(msg: string): void;
}
class ConsoleLogger implements Logger {
  log(msg) {}
  // ❌ TS7006: Parameter 'msg' implicitly has an 'any' type.
}
```

Always write the parameter types in the class.

**5. Real-world use**
The repository from 4.8 (`implements Repository<T>`), mock classes in tests, and swappable services.

**7. Interview tip**
_"What does `implements` actually do?"_ It is a compile-time check only, and generates no JavaScript.

---

## 6.9 Generics in classes 🔴 MUST KNOW

You met `Stack<T>` and `InMemoryRepository<T>` in Part 4. The pieces that matter here:

```ts
interface HasId {
  id: string;
}

class Cache<T extends HasId> {
  private store = new Map<string, T>();

  set(item: T): void {
    this.store.set(item.id, item);
  }
  get(id: string): T | undefined {
    return this.store.get(id);
  }
}

// Subclass fixes or forwards the type parameter
class UserCache extends Cache<User> {} // fixes T = User
class TimedCache<T extends HasId> extends Cache<T> {} // forwards T
```

**Remember:** constraints (`extends HasId`) go on the class; `static` members cannot use `T`; `T` is erased at runtime, so `new T()` is impossible. To create instances, pass a constructor: `create(ctor: new () => T): T`.

---

## 6.10 Decorators 🟢 LEGACY / READ-ONLY

**What it is**
A decorator is a function applied with `@name` above a class, method, property or parameter. It can attach metadata or wrap behavior. NestJS, TypeORM and Angular rely on them heavily.

**Two versions exist (this confuses many people):**

|                                    | Legacy decorators                | Standard decorators (TS 5.0+) |
| ---------------------------------- | -------------------------------- | ----------------------------- |
| Enabled by                         | `"experimentalDecorators": true` | Default, no flag              |
| Used by                            | NestJS, TypeORM, older Angular   | Newer code and libraries      |
| Parameter decorators               | Yes                              | No                            |
| Metadata (`emitDecoratorMetadata`) | Yes                              | Not supported                 |

They are **not compatible**. Which one your project uses depends on its `tsconfig`, so check it before debugging. NestJS has historically required the legacy mode; confirm with the NestJS docs for your version.

```ts
// NestJS-style code: what you will READ
@Controller('users') // class decorator: registers a route prefix
export class UsersController {
  constructor(private readonly usersService: UsersService) {} // injected automatically

  @Get(':id') // method decorator: GET /users/:id
  findOne(@Param('id') id: string) {
    // parameter decorator: pulls :id from the URL
    return this.usersService.findOne(id);
  }
}

// A simple method decorator, legacy style (with experimentalDecorators)
function Log(target: object, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function (...args: unknown[]) {
    console.log(`Calling ${key}`, args);
    return original.apply(this, args);
  };
}
```

You do not need to write decorators as a fresher. You need to read them as "labels that tell a framework what to do." Interview line: _decorators run at class-definition time and add metadata or wrap behavior._

---

## 6.11 Mixins 🟢 LEGACY / READ-ONLY

**What it is**
A way to "mix" behavior from several sources into one class, because a class can `extend` only one parent. A mixin is a function that takes a class and returns a subclass.

```ts
type Constructor<T = {}> = new (...args: any[]) => T; // "anything you can `new`"

function Timestamped<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
    touch() {
      return new Date();
    }
  };
}

class Product {
  constructor(public name: string) {}
}
const TimestampedProduct = Timestamped(Product);

const p = new TimestampedProduct('Keyboard');
p.createdAt; // Date
p.name; // string
```

`any[]` in `Constructor` is one of the accepted `any` uses (a pattern must match every constructor). Today most teams prefer **composition** or plain functions to mixins, so treat this as read-only knowledge for older libraries.

---

### 📌 Key Takeaways

- Declare every property; `strictPropertyInitialization` forces initialization (`!` is an unchecked promise).
- `private`/`protected` are erased at runtime; `#private` is truly private in JS.
- Parameter properties (`constructor(private readonly repo: Repo) {}`) are the standard DI style in Node and NestJS.
- Use `override` with `noImplicitOverride`, call `super()` first, and prefer composition over deep inheritance.
- Abstract classes share code and exist at runtime; interfaces with `implements` are compile-time contracts only.
- Decorators come in legacy (`experimentalDecorators`) and standard (TS 5+) versions; just learn to read them. Mixins are legacy-level knowledge.

✅ Part 6 complete.
