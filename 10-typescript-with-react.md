# Part 10a: React + TypeScript Foundations

Part 10 is split into three replies to stay within the word limit:

- **10a** (this reply): Vite setup, props, `children`, default props, events, `ComponentProps`, `forwardRef`.
- **10b**: hooks (`useState`, `useReducer`, `useRef`, `useContext` and others), custom hooks, typed context, generic components.
- **10c**: data fetching, TanStack Query, React Hook Form + Zod, Redux Toolkit, React Router, styling props, common errors.

**Assumptions:** React 18 or 19, Vite, function components, `.tsx` files, and the frontend `tsconfig` from Part 0.6. Differences between React versions are flagged.

---

## 10.1 Vite + React + TS setup 🔴 MUST KNOW

**1. What it is**
Vite is the dev server and bundler for most new React projects. Its `react-ts` template gives you a working React + TS setup.

**3. Code**

```bash
npm create vite@latest shop-web -- --template react-ts
cd shop-web
npm install
npm run dev
```

```json
// package.json scripts (typical; exact template contents vary by Vite version)
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  }
}
```

```tsx
// src/main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

Here `!` is an acceptable promise: `index.html` defines `#root`, and the app cannot work without it.

**Typing environment variables (Vite)**

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

```ts
const baseUrl = import.meta.env.VITE_API_URL; // string
import.meta.env.VITE_API_URL_TYPO;
// ❌ TS2339: Property 'VITE_API_URL_TYPO' does not exist on type 'ImportMetaEnv'.
```

This is a **promise, not a check** (like 7.7). Only variables prefixed `VITE_` reach the browser. **Anything in a frontend env var is public.** Never put secrets there.

**4. ❌ Wrong / ✅ Right**

```
// ❌ Trusting `npm run dev`: Vite strips types and does NOT type check (Part 0.3).
// ✅ `npm run build` runs `tsc -b` first, so type errors fail the build.
//    Keep Error Lens on in VS Code and run the build/typecheck before pushing.
```

**5. Real-world use**
CI for the frontend runs `npm run lint && npm run build`. A green dev server does not mean the code type checks.

**6. Common mistakes & errors**

- `TS2307: Cannot find module './logo.svg' or its corresponding type declarations.` The `vite/client` reference is missing (7.4).
- `Cannot find name 'process'` in frontend code. Vite uses `import.meta.env`, not `process.env`.
- Vite templates split config into `tsconfig.json`, `tsconfig.app.json` and `tsconfig.node.json` (7.10). Edit `tsconfig.app.json` for your `src` code.

---

## 10.2 Typing props: `interface` vs `type`, and `React.FC` 🔴 MUST KNOW

**1. What it is**
A component is a function whose first parameter is a **props object**. You type that object like any object (Part 3).

**2. Why it exists**
Wrong or missing props become compile errors in the editor instead of a blank screen.

**3. Code**

```tsx
interface User {
  id: number;
  name: string;
  email: string;
}

interface UserCardProps {
  user: User; // required
  isSelected?: boolean; // optional
  onSelect: (id: number) => void; // callback prop
}

function UserCard({ user, isSelected = false, onSelect }: UserCardProps) {
  return (
    <li
      style={{ fontWeight: isSelected ? 'bold' : 'normal' }}
      onClick={() => onSelect(user.id)}
    >
      {user.name} ({user.email})
    </li>
  );
}

const users: User[] = [{ id: 1, name: 'Asha', email: 'asha@example.com' }];

<UserCard user={users[0]} onSelect={(id) => console.log(id)} />;
// ❌ TS2322: Type 'User | undefined' is not assignable to type 'User'.   (with noUncheckedIndexedAccess)

<UserCard onSelect={() => {}} />;
// ❌ TS2741: Property 'user' is missing in type '{ onSelect: () => void; }' but required in type 'UserCardProps'.

<UserCard user={users[0]!} onSelect={() => {}} color="red" />;
// ❌ TS2322: Type '{ ... color: string; }' is not assignable to type 'IntrinsicAttributes & UserCardProps'.
//    Property 'color' does not exist on type 'IntrinsicAttributes & UserCardProps'.
```

Read `IntrinsicAttributes` as "React's built-in extras like `key`". The real mismatch is the property named after it.

**`interface` or `type` for props?** Both work (3.2). `type` is needed for unions; `interface` extends cleanly. Pick one for the whole team. Examples here use `interface`.

**`React.FC`: should you use it?**

```tsx
const UserCard2: React.FC<UserCardProps> = ({ user }) => <li>{user.name}</li>;
```

|                     | Plain function (`function X(props: P)`) | `React.FC<P>`                                                                 |
| ------------------- | --------------------------------------- | ----------------------------------------------------------------------------- |
| Implicit `children` | No                                      | No (since `@types/react` 18; v17 added it, which you will see in legacy code) |
| Generic components  | ✅ Possible                             | ❌ Not possible                                                               |
| Verbosity           | Lower                                   | Higher                                                                        |
| Seen in             | Most modern codebases                   | Older tutorials and legacy code                                               |

Most modern teams skip `React.FC`. It is team opinion, so you must be able to read both.

**4. ❌ Wrong / ✅ Right**

```tsx
function Bad(props) {} // ❌ TS7006: Parameter 'props' implicitly has an 'any' type.
function Bad2({ user }) {} // ❌ TS7031: Binding element 'user' implicitly has an 'any' type.
function Good({ user }: UserCardProps) {} // ✅
```

**5. Real-world use**
Name props `ComponentNameProps` and keep them next to the component. Export the props type only if other files need it. Otherwise use `ComponentProps<typeof Comp>` (10.7).

**6. Common mistakes**

- Passing a whole object when the component expects a field (`user={user.name}`). The error is `TS2322: Type 'string' is not assignable to type 'User'`.
- Props are **read-only**. Do not mutate them.

**7. Interview tip**
_"How do you type props?"_ An interface or type for the props object, annotated on the destructured parameter. Mention that `React.FC` is optional and cannot be generic.

---

## 10.3 `children`, `ReactNode` vs `ReactElement` 🔴 MUST KNOW

**1. What it is**
`children` is whatever you put between a component's tags. You type it yourself; it is not automatic.

| Type                | What it accepts                                                                                  | Use for                                                             |
| ------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| `ReactNode`         | Anything renderable: strings, numbers, `null`, `undefined`, booleans, elements, arrays/fragments | **`children` and "something to render" props** (the default choice) |
| `ReactElement`      | Only a JSX element object (`<div />`, `<Foo />`)                                                 | When you must receive an element (for example, to clone it)         |
| `React.JSX.Element` | The type of a JSX expression                                                                     | Return types, rarely needed                                         |

**3. Code**

```tsx
import type { ReactNode, ReactElement, PropsWithChildren } from 'react';

interface CardProps {
  title: string;
  children: ReactNode;
  footer?: ReactNode; // a "slot" prop
}

function Card({ title, children, footer }: CardProps) {
  return (
    <section>
      <h2>{title}</h2>
      {children}
      {footer}
    </section>
  );
}

<Card title="Profile">Hello</Card>; // ✅ string
<Card title="Profile">{null}</Card>; // ✅
<Card title="Profile" />;
// ❌ TS2741: Property 'children' is missing in type '{ title: string; }' but required in type 'CardProps'.

// PropsWithChildren<P> = P & { children?: ReactNode }
function Page({ title, children }: PropsWithChildren<{ title: string }>) {
  return (
    <main>
      <h1>{title}</h1>
      {children}
    </main>
  );
}

// Needs an element specifically
function Tooltip({ trigger }: { trigger: ReactElement }) {
  return <span>{trigger}</span>;
}
<Tooltip trigger={<button>?</button>} />; // ✅
<Tooltip trigger="text" />;
// ❌ TS2322: Type 'string' is not assignable to type 'ReactElement<any, string | JSXElementConstructor<any>>'.
```

**Render props (children as a function)**

```tsx
interface ListProps<T> {
  items: T[];
  children: (item: T) => ReactNode; // function as children
}
```

Typing `children` as plain `ReactNode` and then passing a function gives `TS2322: Type '(item: User) => JSX.Element' is not assignable to type 'ReactNode'.`

**4. ❌ Wrong / ✅ Right**

```tsx
interface BadProps {
  children: string;
} // ❌ rejects <b>bold</b>, numbers and multiple children
interface GoodProps {
  children: ReactNode;
} // ✅
```

**Version note:** React 19 types removed the global `JSX` namespace. Write `React.JSX.Element` (or avoid naming it). Legacy code uses a bare `JSX.Element`. Leave the return type off components; inference handles it.

**5. Real-world use**
Layout components (`Page`, `Modal`, `Card`), context providers (10b) and error boundaries.

**6. Common mistakes**

- Using `JSX.Element` for `children`. It rejects strings, `null` and arrays.
- Expecting `children` to type check what is inside. TS **cannot** restrict which child elements are allowed (a known limitation).

**7. Interview tip**
_"`ReactNode` vs `ReactElement`?"_ `ReactNode` is anything React can render; `ReactElement` is one specific JSX element. Use `ReactNode` for `children`.

---

## 10.4 Default and optional props 🔴 MUST KNOW

**1. What it is**
Give optional props defaults with **default parameter values** in the destructuring.

**3. Code**

```tsx
interface ButtonProps {
  label: string;
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
}

function Button({ label, size = 'md', disabled = false }: ButtonProps) {
  // inside, `size` is "sm" | "md" | "lg" (never undefined)
  return (
    <button disabled={disabled} className={`btn-${size}`}>
      {label}
    </button>
  );
}

<Button label="Save" />; // ✅
<Button label="Save" size="xl" />;
// ❌ TS2322: Type '"xl"' is not assignable to type '"sm" | "md" | "lg"'.
```

**4. ❌ Wrong / ✅ Right**

```tsx
// ❌ Legacy: defaultProps. You will see it in old class and function components.
Button.defaultProps = { size: 'md' };
// React 19 ignores defaultProps on function components (React 18 only warns).
// ✅ Default parameter values, as above.
```

Optional (`?`) vs a default: with `size?: ...` and no default, `size` is `... | undefined` inside the component. A default removes the `undefined`.

**5. Real-world use**
UI-library-style components: `variant`, `size`, `disabled`, `loading`. Union types for options (Part 1.8) give you autocomplete.

**6. Common mistakes**

- Using `||` for defaults (`size || "md"`), which also replaces `0` and `""`. Parameter defaults only replace `undefined`.
- Passing `null` to a prop typed `T | undefined`. Defaults do not apply to `null`.

---

## 10.5 Props as a discriminated union 🟡 GOOD TO KNOW

**1. What it is**
When valid prop combinations depend on one another, use a union (Part 3.7) instead of many optional props.

**3. Code**

```tsx
// ❌ Messy: href and onClick can both be set, or neither
interface BadProps {
  label: string;
  href?: string;
  onClick?: () => void;
}

// ✅ Each variant carries only what it needs
type ActionProps =
  | { kind: 'link'; label: string; href: string }
  | { kind: 'button'; label: string; onClick: () => void };

function Action(props: ActionProps) {
  if (props.kind === 'link') return <a href={props.href}>{props.label}</a>; // narrowed
  return <button onClick={props.onClick}>{props.label}</button>;
}

<Action kind="link" label="Docs" onClick={() => {}} />;
// ❌ TS2322: Type '{ kind: "link"; label: string; onClick: () => void; }' is not assignable to type 'IntrinsicAttributes & ActionProps'.
//    Property 'onClick' does not exist on type ...
```

**Gotcha:** destructuring in the parameter (`({ kind, href })`) can lose narrowing in older TS. Keep `props` whole, or narrow first and then destructure.

**5. Real-world use**
Link-or-button components, alerts by severity, and a form field with `type: "select"` requiring `options` while `type: "text"` does not.

---

## 10.6 Typing events 🔴 MUST KNOW

**1. What it is**
React wraps browser events in its own types: `ChangeEvent`, `FormEvent`, `MouseEvent`, `KeyboardEvent` and so on. Each takes the **element type** as a generic, which decides what `e.currentTarget` is.

**2. Why it exists**
`e.target.value` only exists on inputs. The generic tells TS which element you are on.

**3. Code**

```tsx
import { useState } from 'react';
import type {
  ChangeEvent,
  FormEvent,
  MouseEvent,
  KeyboardEvent,
  ChangeEventHandler,
} from 'react';

function SearchForm({ onSearch }: { onSearch: (query: string) => void }) {
  const [query, setQuery] = useState(''); // string, inferred (hooks: 10b)

  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value); // string
  };

  const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault(); // stop the page reload
    onSearch(query);
  };

  const handleClear = (e: MouseEvent<HTMLButtonElement>) => {
    console.log(e.currentTarget.name); // the button the handler is on
    setQuery('');
  };

  const handleKeyDown = (e: KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Escape') setQuery('');
  };

  // The "Handler" form types the whole function in one go
  const handleChange2: ChangeEventHandler<HTMLInputElement> = (e) =>
    setQuery(e.target.value);

  return (
    <form onSubmit={handleSubmit}>
      <input value={query} onChange={handleChange} onKeyDown={handleKeyDown} />
      <button type="button" name="clear" onClick={handleClear}>
        Clear
      </button>
      <button type="submit">Search</button>
    </form>
  );
}

// Reading a form without state: FormData
const onSubmitNative = (e: FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  const data = new FormData(e.currentTarget);
  const email = data.get('email'); // FormDataEntryValue | null (string | File | null)
  if (typeof email === 'string') {
    /* use it */
  }
};
```

| Element          | Change event type                  |
| ---------------- | ---------------------------------- |
| `<input>`        | `ChangeEvent<HTMLInputElement>`    |
| `<textarea>`     | `ChangeEvent<HTMLTextAreaElement>` |
| `<select>`       | `ChangeEvent<HTMLSelectElement>`   |
| `<form>`         | `FormEvent<HTMLFormElement>`       |
| `<button>` click | `MouseEvent<HTMLButtonElement>`    |

**Inline handlers need no annotation**, thanks to contextual typing (Part 2.4):

```tsx
<input onChange={(e) => setQuery(e.target.value)} /> // e is typed automatically
```

Annotate only when the handler is **defined separately**, as above.

**4. ❌ Wrong / ✅ Right**

```tsx
const onClick = (e: MouseEvent<HTMLButtonElement>) =>
  console.log(e.target.name);
// ❌ TS2339: Property 'name' does not exist on type 'EventTarget'.
// ✅ e.currentTarget is the element the handler is attached to. e.target may be any child inside it.

<input onChange={handleClear} />;
// ❌ TS2322: Type '(e: MouseEvent<HTMLButtonElement>) => void' is not assignable to type 'ChangeEventHandler<HTMLInputElement>'.
```

For `<select>`, `e.target.value` is a plain `string`, so `e.target.value as Role` is an unchecked claim (Part 1.10). Prefer a guard such as `isRole(value)`.

**Name clash:** `MouseEvent` and `KeyboardEvent` also exist as **browser DOM** types. Importing the React ones from `"react"` (as above) avoids mixing them up. Many codebases write `React.MouseEvent` instead. Both styles are common.

**5. Real-world use**
Controlled inputs, form submission, keyboard shortcuts, drag and drop (`DragEvent`), and file upload (`ChangeEvent<HTMLInputElement>` with `e.target.files`, typed `FileList | null`).

**6. Common mistakes**

- `e.target.files[0]` errors: `files` may be `null`, and indexing may be `undefined`. Use `e.target.files?.[0]`.
- Forgetting `e.preventDefault()` in `onSubmit` is a runtime bug that TS cannot detect.
- Reading an event after an `await` is fine in React 17+ (the old "event pooling" problem is gone), but your legacy code may still copy values defensively.

**7. Interview tip**
_"How do you type `onChange` for an input?"_ `ChangeEvent<HTMLInputElement>`, or `ChangeEventHandler<HTMLInputElement>` for the whole function. Know the difference between `target` and `currentTarget`.

---

## 10.7 `ComponentProps` and wrapping native elements 🔴 MUST KNOW

**1. What it is**
`ComponentProps<"button">` is **every prop a native `<button>` accepts** (`onClick`, `disabled`, `type`, `aria-*`, and more). `ComponentProps<typeof MyComponent>` gives the props of one of your own components.

**2. Why it exists**
Wrapper components (`Button`, `Input`) should accept everything the native element does without you listing each prop.

**3. Code**

```tsx
import type { ComponentProps, ComponentPropsWithoutRef } from 'react';

// Wrap a native element and add your own props
interface ButtonProps extends ComponentPropsWithoutRef<'button'> {
  variant?: 'primary' | 'danger';
}

function Button({ variant = 'primary', className, ...rest }: ButtonProps) {
  return (
    <button className={`btn btn-${variant} ${className ?? ''}`} {...rest} />
  );
}

<Button variant="danger" onClick={() => {}} disabled aria-label="Delete" />; // ✅ all native props work

// Get props of a component you did not export a type for
function UserCard(props: { user: User; onSelect: (id: number) => void }) {
  return null;
}
type UserCardProps = ComponentProps<typeof UserCard>;
```

`...rest` forwards every other prop to the DOM element. TS knows its exact type.

**Which variant to use**

| Type                                | Includes `ref`?                               |
| ----------------------------------- | --------------------------------------------- |
| `ComponentProps<"input">`           | Yes                                           |
| `ComponentPropsWithoutRef<"input">` | No: use when you handle `ref` yourself (10.8) |
| `ComponentPropsWithRef<"input">`    | Yes, explicitly                               |

**4. ❌ Wrong / ✅ Right**

```tsx
interface BadProps extends ComponentPropsWithoutRef<'button'> {
  type: 'primary' | 'danger';
  // ❌ TS2430: Interface 'BadProps' incorrectly extends interface 'ComponentPropsWithoutRef<"button">'.
  //    Types of property 'type' are incompatible.
}
// ✅ Rename it (variant), or Omit the native one first:
type OkProps = Omit<ComponentPropsWithoutRef<'button'>, 'type'> & {
  type: 'primary' | 'danger';
};
```

Also watch out: `<input>` has a native `size` (number) prop. A custom `size: "sm" | "lg"` collides with it the same way. `Omit` it.

**5. Real-world use**
Design-system components (`Input`, `Select`, `Link`), thin wrappers over `<a>` and `<img>`, and the pattern behind libraries like shadcn/ui.

**6. Common mistakes**

- Typing every native prop by hand. It is error-prone and never complete.
- Spreading `{...rest}` **before** your own props, so a caller's `className` or `onClick` overrides yours. Order matters.

**7. Interview tip**
_"How do you type a wrapper around `<button>`?"_ `extends ComponentPropsWithoutRef<"button">`, destructure your own props and spread `...rest`.

---

## 10.8 `forwardRef` and `ref` as a prop 🔴 MUST KNOW

**1. What it is**
A `ref` gives a parent direct access to a DOM element (for example `.focus()`). In **React 18**, function components cannot receive `ref` as a normal prop, so you wrap them in `forwardRef`. In **React 19**, `ref` is a regular prop, and `forwardRef` is no longer needed (React's docs say it will be deprecated in a future release). You will see both.

**3. Code (React 18 style, very common in existing code)**

```tsx
import { forwardRef, useRef } from 'react';
import type { ComponentPropsWithoutRef } from 'react';

interface TextInputProps extends ComponentPropsWithoutRef<'input'> {
  label: string;
}

//                              ↓ element type FIRST, props SECOND
const TextInput = forwardRef<HTMLInputElement, TextInputProps>(
  function TextInput({ label, ...rest }, ref) {
    return (
      <label>
        {label}
        <input ref={ref} {...rest} />
      </label>
    );
  },
);

function LoginForm() {
  const emailRef = useRef<HTMLInputElement>(null); // hooks: 10b
  return (
    <>
      <TextInput label="Email" ref={emailRef} />
      <button onClick={() => emailRef.current?.focus()}>Focus email</button>
    </>
  );
}
```

**React 19 style (no `forwardRef`)**

```tsx
import type { ComponentPropsWithoutRef, Ref } from 'react';

function TextInput19({
  label,
  ref,
  ...rest
}: {
  label: string;
  ref?: Ref<HTMLInputElement>;
} & ComponentPropsWithoutRef<'input'>) {
  return (
    <label>
      {label}
      <input ref={ref} {...rest} />
    </label>
  );
}
```

`@types/react` 19 is needed for this version. Which style applies is decided by your React and `@types/react` versions.

**4. ❌ Wrong / ✅ Right**

```tsx
forwardRef<TextInputProps, HTMLInputElement>(/* ... */);
// ❌ Swapped type arguments. You get confusing errors, for example
// TS2322: Type 'ForwardedRef<TextInputProps>' is not assignable to type 'Ref<HTMLInputElement> | undefined'.
// ✅ forwardRef<RefElement, Props>: the ref element comes first.
```

Name the inner function (as above) so DevTools shows `TextInput`, not `ForwardRef`. This is also why older code sets `TextInput.displayName`.

**Limitation:** `forwardRef` loses generics. A generic component such as `Table<T>` wrapped in `forwardRef` needs a cast to restore its type parameter. Another reason React 19's plain `ref` prop is nicer.

**5. Real-world use**
Focusing the first invalid field, scrolling to an element, and integrating with libraries that need a DOM node. React Hook Form's `register` relies on refs (10c).

**6. Common mistakes & errors**

- `TS2322: Type 'MutableRefObject<HTMLInputElement | undefined>' is not assignable to type 'Ref<HTMLInputElement>'.` Start the ref with `null` (`useRef<HTMLInputElement>(null)`), not `undefined`.
- Forgetting `?.` on `ref.current`, because it is `null` before mount and after unmount.
- Using `ComponentProps<"input">` (which already contains `ref`) together with `forwardRef`. Use `ComponentPropsWithoutRef`.

**7. Interview tip**
_"How do you pass a ref through a custom component?"_ `forwardRef<ElementType, Props>` in React 18, or a plain `ref` prop in React 19. Mention the generic order.

---

### 📌 Key Takeaways

- Vite does not type check. `tsc -b` in the build script (and your editor) does, and `VITE_` env vars are public strings.
- Type props with an interface or type on the destructured parameter. Plain functions beat `React.FC` (it cannot be generic).
- Use `ReactNode` for `children` and slot props; `ReactElement` only when you need an actual element.
- Default optional props with parameter defaults. `defaultProps` is legacy, and React 19 ignores it on function components.
- Type events with the element generic (`ChangeEvent<HTMLInputElement>`). Use `currentTarget` for the element and annotate only separately defined handlers.
- Wrap native elements with `extends ComponentPropsWithoutRef<"tag">` and `...rest`. For refs, use `forwardRef<Element, Props>` (React 18) or a `ref` prop (React 19).

✅ Part 10a complete.

---

# Part 10b: Hooks, Custom Hooks, Context & Generic Components

**Assumptions:** same as 10a (React 18/19, Vite, strict, `noUncheckedIndexedAccess`). Examples reuse `User` from 10.2:

```ts
interface User {
  id: number;
  name: string;
  email: string;
}
```

---

## 10.9 `useState` 🔴 MUST KNOW

**1. What it is**
`useState<T>` stores a value that, when changed, re-renders the component. TS infers `T` from the initial value. You write it explicitly when the initial value does not tell the whole story.

**3. Code**

```tsx
import { useState } from 'react';

const [count, setCount] = useState(0); // number (inferred)
const [name, setName] = useState(''); // string

// Value that starts empty: say what it will hold
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<string[]>([]); // without <string[]>, [] infers never[]

// Updater function form: use when the new value depends on the old one
setCount((prev) => prev + 1); // prev: number

setCount('5');
// ❌ TS2345: Argument of type 'string' is not assignable to parameter of type 'SetStateAction<number>'.

// Lazy initial value: the function runs only once
const [data, setData] = useState<User[]>(() =>
  JSON.parse(localStorage.getItem('users') ?? '[]'),
);
// ⚠️ JSON.parse returns any: validate with Zod (Part 8) for real code
```

**Using state that may be `null`**

```tsx
user.name;
// ❌ TS18047: 'user' is possibly 'null'.
user?.name; // ✅
if (user) {
  user.name;
} // ✅ narrowing
```

**4. ❌ Wrong / ✅ Right**

```tsx
const [items, setItems] = useState([]); // ❌ never[]: setItems(["a"]) fails with TS2345
const [items2, setItems2] = useState<string[]>([]); // ✅

// ❌ Mutating state: React will not re-render
items.push('x');
// ✅ New array
setItems((prev) => [...prev, 'x']);
```

For a loading/error/data trio, prefer one **discriminated union** state (Part 3.7) over three separate booleans:

```tsx
type FetchState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; message: string };
const [state, setState] = useState<FetchState>({ status: 'idle' });
```

**5. Real-world use**
Form inputs, toggles, selected rows, modal open/closed.

**6. Common mistakes**

- Storing derived values in state. Compute them during render instead.
- Passing `setState` types around: use `Dispatch<SetStateAction<T>>` (from `"react"`) as the prop type.

**7. Interview tip**
_"When do you need to pass a type argument to `useState`?"_ When the initial value is `null`, `undefined` or an empty array/object, so inference cannot see the full type.

---

## 10.10 `useReducer` 🔴 MUST KNOW

**1. What it is**
State managed by a **reducer**: `(state, action) => newState`. It fits state with several related fields or many update types. Actions form a **discriminated union** (Part 3.7).

**3. Code**

```tsx
import { useReducer } from 'react';

interface Todo {
  id: number;
  text: string;
  done: boolean;
}

type State = { todos: Todo[] };

type Action =
  | { type: 'add'; text: string }
  | { type: 'toggle'; id: number }
  | { type: 'remove'; id: number }
  | { type: 'clear' };

function assertNever(x: never): never {
  throw new Error(`Unhandled action: ${JSON.stringify(x)}`);
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'add':
      return {
        todos: [
          ...state.todos,
          { id: Date.now(), text: action.text, done: false },
        ],
      };
    case 'toggle':
      return {
        todos: state.todos.map((t) =>
          t.id === action.id ? { ...t, done: !t.done } : t,
        ),
      };
    case 'remove':
      return { todos: state.todos.filter((t) => t.id !== action.id) };
    case 'clear':
      return { todos: [] };
    default:
      return assertNever(action); // adding a new action type breaks the build here (3.11)
  }
}

function TodoApp() {
  const [state, dispatch] = useReducer(reducer, { todos: [] }); // types inferred from reducer
  dispatch({ type: 'add', text: 'Learn TS' }); // ✅
  dispatch({ type: 'toggle' });
  // ❌ TS2345: Argument of type '{ type: "toggle"; }' is not assignable to parameter of type 'Action'.
  //    Property 'id' is missing in type '{ type: "toggle"; }' but required in type '{ type: "toggle"; id: number; }'.
  dispatch({ type: 'rename', id: 1 });
  // ❌ TS2345: Argument of type '{ type: "rename"; id: number; }' is not assignable to parameter of type 'Action'.
  return null;
}
```

**4. ❌ Wrong / ✅ Right**

```tsx
type BadAction = { type: string; payload?: unknown }; // ❌ accepts anything, no narrowing
// ✅ the union above: each action carries exactly its own fields
```

Annotate the reducer's **return type** (`: State`). Then forgetting a field in one branch is an error right there.

**5. Real-world use**
Multi-step forms, shopping carts, editors. The same action-union idea carries over to Redux Toolkit (10c).

**6. Common mistakes**

- Mutating `state` inside the reducer (`state.todos.push(...)`). Always return new objects.
- Forgetting the `default` branch, so a new action silently does nothing.
- The initial state's type must match `State` exactly. `useReducer(reducer, {})` gives `TS2345`.

**7. Interview tip**
_"How do you type a reducer?"_ A `State` type, a discriminated union `Action`, and `(state: State, action: Action): State`. End the switch with `assertNever`.

---

## 10.11 `useRef` 🔴 MUST KNOW

**1. What it is**
`useRef` gives an object `{ current: T }` that survives re-renders and does **not** cause one when changed. Two uses: **DOM element refs** and **mutable boxes** (timers, previous values).

**3. Code**

```tsx
import { useRef, useEffect } from 'react';

// 1) DOM ref: start with null, give the element type
const inputRef = useRef<HTMLInputElement>(null); // RefObject<HTMLInputElement | null> (React 19 types)
<input ref={inputRef} />;
inputRef.current?.focus(); // current is null until mounted: use ?.

// 2) Mutable value: no null, initial value given
const timerId = useRef<number | undefined>(undefined);
const renderCount = useRef(0); // number
renderCount.current += 1; // ✅ mutable

useEffect(() => {
  timerId.current = window.setTimeout(() => {}, 1000);
  return () => window.clearTimeout(timerId.current);
}, []);
```

**Why `null` matters:** the overloads decide what you get.

```tsx
useRef<HTMLInputElement>(null); // DOM ref: current is READ-ONLY by React's convention (types differ by version)
useRef<HTMLInputElement | null>(null); // mutable box that may hold an element
```

React 18 types made `useRef<T>(null)` return a read-only `RefObject<T>`, so you could not assign `ref.current = x`. React 19 types changed `RefObject.current` to be mutable and made `useRef` require an argument. Check hover in your editor.

**4. ❌ Wrong / ✅ Right**

```tsx
const ref = useRef<HTMLInputElement>();
// ❌ React 19 types: TS2554: Expected 1 arguments, but got 0.
// ❌ React 18 types: MutableRefObject<HTMLInputElement | undefined>, which is not assignable to the `ref` prop (TS2322)
const ref2 = useRef<HTMLInputElement>(null); // ✅

ref2.current.focus();
// ❌ TS18047: 'ref2.current' is possibly 'null'.
```

**5. Real-world use**
Focusing fields, scrolling, measuring elements, storing a `setTimeout` id, keeping the previous value, and an "is mounted" flag.

**6. Common mistakes**

- Using a ref as state. Changing `.current` does not re-render, so the UI will not update.
- `setTimeout` returns `number` in browser typing but `NodeJS.Timeout` if `@types/node` leaks into your frontend. Using `window.setTimeout` (returns `number`) avoids the clash, or write `ReturnType<typeof setTimeout>`.
- `ref.current!` hides a real "not mounted yet" bug. Prefer `?.`.

**7. Interview tip**
_"How do you type a DOM ref?"_ `useRef<HTMLInputElement>(null)`, with the element type as the generic and `null` as the initial value.

---

## 10.12 `useContext` and typing context safely 🔴 MUST KNOW

**1. What it is**
Context shares a value with a whole subtree without passing props. The challenge in TS: `createContext` needs a default, but a real value exists only inside the **Provider**.

**3. Code: the safe pattern**

```tsx
// src/context/AuthContext.tsx
import {
  createContext,
  useContext,
  useState,
  useMemo,
  useCallback,
} from 'react';
import type { ReactNode } from 'react';

interface AuthContextValue {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

// 1) Default is `null`: honest, since there is no value outside a Provider
const AuthContext = createContext<AuthContextValue | null>(null);

// 2) Provider
export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = useCallback(async (email: string, password: string) => {
    const res = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });
    if (!res.ok) throw new Error('Login failed');
    setUser((await res.json()) as User); // ⚠️ unchecked: validate with Zod (10c)
  }, []);

  const logout = useCallback(() => setUser(null), []);

  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
// (React 19 allows <AuthContext value={value}>; React 18 needs .Provider)

// 3) Custom hook: the ONLY way components read the context
export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (ctx === null)
    throw new Error('useAuth must be used inside <AuthProvider>');
  return ctx; // narrowed: AuthContextValue, no null checks in components
}
```

```tsx
function Navbar() {
  const { user, logout } = useAuth(); // fully typed, never null
  return <button onClick={logout}>{user?.name ?? 'Guest'}</button>;
}
```

**4. ❌ Wrong / ✅ Right**

```tsx
const AuthContext = createContext<AuthContextValue>({} as AuthContextValue);
// ❌ Compiles, but outside a Provider every field is undefined: "logout is not a function" at runtime
// ✅ createContext<AuthContextValue | null>(null) + a throwing hook, as above
```

Putting the Provider and the hook in the same file as components can break Fast Refresh (an ESLint warning from `react-refresh`). Many teams split `AuthContext.ts`, `AuthProvider.tsx` and `useAuth.ts`. It is a style choice.

**5. Real-world use**
Auth, theme, locale, and feature flags. For server data, TanStack Query (10c) is a better fit than context.

**6. Common mistakes**

- Not memoizing `value` (`useMemo`): a new object each render re-renders every consumer.
- Putting fast-changing state in context. Everything below re-renders.
- `TS2322: Type '{ children: Element; }' is not assignable to type 'IntrinsicAttributes & ...'` when the Provider component forgot to declare `children` (10.3).

**7. Interview tip**
_"How do you avoid `undefined` when using context in TS?"_ Default to `null`, then write a custom hook that throws when the context is missing and returns the non-null type.

---

## 10.13 `useMemo` and `useCallback` 🟡 GOOD TO KNOW

**1. What it is**
`useMemo` caches a **computed value**; `useCallback` caches a **function**. Both depend on a dependency array. Types are inferred, so you rarely annotate.

**3. Code**

```tsx
import { useMemo, useCallback } from 'react';

const sorted = useMemo(
  () => [...users].sort((a, b) => a.name.localeCompare(b.name)), // User[]
  [users],
);

const handleSelect = useCallback((id: number) => {
  // (id: number) => void
  console.log(id);
}, []);

// Annotate parameters of the callback: contextual typing has nothing to infer from
const bad = useCallback((id) => {}, []);
// ❌ TS7006: Parameter 'id' implicitly has an 'any' type.
```

**4. ❌ Wrong / ✅ Right**
Do not wrap everything. They help only when (a) the work is expensive, or (b) the value is a prop of a `React.memo` child or a dependency of another hook. Wrapping cheap values adds noise. React Compiler (newer React tooling) can do much of this automatically, so check your project's setup.

**6. Common mistake**
Missing dependencies cause stale values. The ESLint rule `react-hooks/exhaustive-deps` catches this. TS cannot.

---

## 10.14 `useEffect` 🔴 MUST KNOW

**1. What it is**
Runs code **after render** for side effects (fetching, subscriptions, timers). The effect function returns **nothing or a cleanup function**.

**3. Code**

```tsx
import { useEffect, useState } from 'react';

function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    async function load() {
      // the effect itself cannot be async (see below)
      try {
        const res = await fetch(`/api/users/${userId}`, {
          signal: controller.signal,
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        setUser((await res.json()) as User); // ⚠️ unchecked (10c uses validation)
      } catch (e) {
        if (e instanceof DOMException && e.name === 'AbortError') return; // ignore cancellations
        console.error(e);
      }
    }
    void load(); // `void` tells the linter the floating promise is intentional

    return () => controller.abort(); // cleanup runs on unmount or before the next run
  }, [userId]);

  return <p>{user?.name}</p>;
}
```

**4. ❌ Wrong / ✅ Right**

```tsx
useEffect(async () => {
  await load();
}, []);
// ❌ TS2345: Argument of type '() => Promise<void>' is not assignable to parameter of type 'EffectCallback'.
//    Type 'Promise<void>' is not assignable to type 'void | Destructor'.
// ✅ define an async function inside and call it (above)
```

Also, an arrow with an expression body can accidentally return a value:

```tsx
useEffect(() => setCount(1), []);
// ❌ TS2345: ... Type 'void' is not assignable to type 'void | Destructor'. (older types) / returns the value of setCount
useEffect(() => {
  setCount(1);
}, []); // ✅ braces, no accidental return
```

**5. Real-world use**
Subscriptions (WebSocket), `addEventListener`/`removeEventListener`, document title, and timers. For data fetching in real apps, TanStack Query (10c) replaces most `useEffect` fetches.

**6. Common mistakes**

- Forgetting cleanup, which causes leaks and updates on unmounted components.
- A fetch race: a slow old request overwriting a newer one. The `AbortController` above prevents it.
- Dependency array mistakes; again the `exhaustive-deps` lint rule is your friend.

**7. Interview tip**
_"Why can't the `useEffect` callback be `async`?"_ It would return a Promise, but React expects nothing or a cleanup function.

---

## 10.15 Typing custom hooks 🔴 MUST KNOW

**1. What it is**
A custom hook is a function starting with `use` that calls other hooks. Type it like any function: typed parameters, and an explicit return type for public hooks.

**3. Code**

```tsx
import { useState, useEffect, useCallback } from 'react';
import type { Dispatch, SetStateAction } from 'react';

// A) Returns a tuple: annotate it, or you get an array union
function useToggle(initial = false): [boolean, () => void] {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((v) => !v), []);
  return [on, toggle];
}
// Without the return type, `return [on, toggle]` infers (boolean | (() => void))[]
// → const [on, toggle] = useToggle(); toggle();  ❌ TS2349: This expression is not callable.
// Alternative: `return [on, toggle] as const;`

// B) Returns an object: easier to extend, no ordering problems
function useDebounce<T>(value: T, delayMs = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [value, delayMs]);
  return debounced;
}

// C) Generic hook with a discriminated-union result
type AsyncState<T> =
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function useAsync<T>(
  fn: () => Promise<T>,
  deps: readonly unknown[],
): AsyncState<T> {
  const [state, setState] = useState<AsyncState<T>>({ status: 'loading' });
  useEffect(() => {
    let cancelled = false;
    setState({ status: 'loading' });
    fn().then(
      (data) => {
        if (!cancelled) setState({ status: 'success', data });
      },
      (e: unknown) => {
        if (!cancelled)
          setState({
            status: 'error',
            error: e instanceof Error ? e : new Error(String(e)),
          });
      },
    );
    return () => {
      cancelled = true;
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps -- deps are supplied by the caller
  }, deps);
  return state;
}

// D) localStorage hook with a default: validates stored data with a type guard parameter
function useLocalStorage<T>(
  key: string,
  initial: T,
  isValid: (value: unknown) => value is T,
): [T, Dispatch<SetStateAction<T>>] {
  const [value, setValue] = useState<T>(() => {
    try {
      const raw = localStorage.getItem(key);
      if (raw === null) return initial;
      const parsed: unknown = JSON.parse(raw);
      return isValid(parsed) ? parsed : initial; // no `as T`
    } catch {
      return initial;
    }
  });
  useEffect(
    () => localStorage.setItem(key, JSON.stringify(value)),
    [value, key],
  );
  return [value, setValue];
}
```

Usage: `const state = useAsync(() => fetchUser(1), []);` then `if (state.status === "success") state.data` (narrowed, as in Part 3).

**4. ❌ Wrong / ✅ Right**

```tsx
function useCounter() {
  return [count, setCount];
} // ❌ infers an array union, breaks destructuring
function useCounter2(): [number, Dispatch<SetStateAction<number>>] {
  /* ... */
} // ✅
```

The `return-only generic` warning (4.9) applies: `useFetch<User>(url)` that just casts the response is the same hidden `as`. Take a **parser/schema** argument instead, or use TanStack Query.

**5. Real-world use**
`useDebounce` for search boxes, `useLocalStorage` for preferences, `useMediaQuery`, `useOnClickOutside`, and `useAuth` (10.12).

**6. Common mistakes**

- Hooks called conditionally or inside loops. A runtime rule enforced by the `rules-of-hooks` ESLint rule, not by TS.
- Returning a new object or array every render when callers use it as a dependency. Memoize if needed.

**7. Interview tip**
_"How do you type a hook that returns a tuple?"_ Annotate the return type or use `as const`, or TS infers an array union.

---

## 10.16 Generic components in practice 🔴 MUST KNOW

**1. What it is**
Part 4.7 introduced `Table<T>`. Here are the other patterns you will actually write, plus the pitfalls.

**3. Code**

```tsx
import type { ReactNode } from 'react';

// A) Generic Select: value type comes from the options
interface SelectProps<TValue extends string | number> {
  options: ReadonlyArray<{ value: TValue; label: string }>;
  value: TValue;
  onChange: (value: TValue) => void;
}

function Select<TValue extends string | number>({
  options,
  value,
  onChange,
}: SelectProps<TValue>) {
  return (
    <select
      value={value}
      onChange={(e) => {
        const match = options.find((o) => String(o.value) === e.target.value);
        if (match) onChange(match.value); // no `as TValue` needed
      }}
    >
      {options.map((o) => (
        <option key={o.value} value={o.value}>
          {o.label}
        </option>
      ))}
    </select>
  );
}

type Role = 'admin' | 'user';
const roleOptions = [
  { value: 'admin', label: 'Admin' },
  { value: 'user', label: 'User' },
] as const; // keeps "admin" | "user" literals

function Demo() {
  const [role, setRole] = useState<Role>('user');
  return <Select options={roleOptions} value={role} onChange={setRole} />; // TValue inferred as Role
}

// B) Render-prop list with a key extractor
interface ListProps<T> {
  items: readonly T[];
  getKey: (item: T) => string | number;
  renderItem: (item: T) => ReactNode;
  empty?: ReactNode;
}
function List<T>({
  items,
  getKey,
  renderItem,
  empty = 'Nothing here',
}: ListProps<T>) {
  if (items.length === 0) return <p>{empty}</p>;
  return (
    <ul>
      {items.map((item) => (
        <li key={getKey(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

<List
  items={[{ id: 1, name: 'Asha' }]}
  getKey={(u) => u.id}
  renderItem={(u) => u.name}
/>;
<List items={[{ id: 1 }]} getKey={(u) => u.id} renderItem={(u) => u.name} />;
// ❌ TS2339: Property 'name' does not exist on type '{ id: number; }'.
```

**4. ❌ Wrong / ✅ Right**

```tsx
const Table: React.FC<TableProps<T>> = (...) => {};
// ❌ TS2304: Cannot find name 'T'.   (a const arrow cannot declare T)
function Table2<T>(props: TableProps<T>) {}      // ✅ function declaration
const Table3 = <T,>(props: TableProps<T>) => null;   // ✅ arrow with the trailing comma in .tsx
```

When `T` cannot be inferred, pass it in JSX: `<Select<Role> ... />` (valid in TSX).

**Generics + `forwardRef`:** `forwardRef` erases `T`. The common workaround is a cast:

```tsx
const SelectWithRef = forwardRef(SelectInner) as <T extends string>(
  props: SelectProps<T> & { ref?: React.Ref<HTMLSelectElement> },
) => ReturnType<typeof SelectInner>;
```

It is ugly and an unchecked claim. React 19's `ref` prop avoids it.

**5. Real-world use**
Tables, selects, lists, autocompletes, and modals with typed results (`Modal<TResult>`).

**6. Common mistakes**

- Over-constraining (`T extends User`) in a component that works for anything. Keep constraints minimal.
- The generic parameter appearing once only (4.9): then it is probably just `unknown` or a concrete type.

---

### 📌 Key Takeaways

- `useState<T>`: pass `T` when starting from `null` or `[]`; prefer discriminated-union state over several booleans.
- `useReducer`: a `State` type, a union `Action`, an explicit return type, and `assertNever` in `default`.
- `useRef<HTMLElement>(null)` for DOM nodes (`?.` on `current`); a ref's `.current` change does not re-render. Ref typings differ between React 18 and 19.
- Type context as `Value | null` and read it through a custom hook that throws; avoid `{} as Value`.
- `useEffect` callbacks cannot be `async`; define an inner async function, and clean up with `AbortController`. Use `useMemo`/`useCallback` only when it helps.
- Custom hooks returning tuples need an annotated return type or `as const`; generic components must be function declarations or `<T,>` arrows, and `forwardRef` loses generics.

✅ Part 10b complete.

---

# Part 10c: Data Fetching, Forms, Redux, Router & Common Errors

**Assumptions:** same as 10a/10b. Library versions are flagged where APIs differ: **TanStack Query v5**, **React Hook Form v7**, **Redux Toolkit 2.x**, **React Router v6/v7**. The `User` type used earlier now comes from a Zod schema.

---

## 10.17 Fetching data with `fetch` and axios 🔴 MUST KNOW

**1. What it is**
Calling your backend from React. As in Part 8.6, **the compiler cannot know what the server sends**, so every "typed fetch" is either a claim or a validated parse.

**2. Why it matters**
`res.json()` is typed `any` (or `unknown` with some setups). Writing `as User[]` or `fetchJson<User[]>(url)` only silences the compiler. If the API changes, the UI breaks at runtime.

**3. Code**

```ts
// src/api/request.ts: ONE place that talks to the server
import { z } from 'zod';

export class ApiError extends Error {
  constructor(
    public readonly status: number,
    message: string,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export async function request<S extends z.ZodType>(
  path: string,
  schema: S,
  init?: RequestInit,
): Promise<z.infer<S>> {
  const res = await fetch(`${import.meta.env.VITE_API_URL}${path}`, {
    ...init,
    headers: { 'Content-Type': 'application/json', ...init?.headers },
  });
  if (!res.ok) throw new ApiError(res.status, `Request failed: ${res.status}`);
  const json: unknown = await res.json();
  return schema.parse(json); // throws if the shape is wrong
}

// src/api/users.ts
export const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string(),
});
export type User = z.infer<typeof UserSchema>;

export const fetchUsers = (): Promise<User[]> =>
  request('/users', z.array(UserSchema));
```

Your Part 9 backend wraps data in `{ success, data }`. Create a helper such as `envelope(schema)` that builds `z.object({ success: z.literal(true), data: schema })`, unwraps `.data`, and turns `{ success: false }` into an `ApiError`. The shape comes from the shared contract (Part 11).

**axios (very common in existing code)**

```ts
import axios, { isAxiosError } from 'axios';

const { data } = await axios.get<User[]>('/api/users'); // ⚠️ <User[]> is a CLAIM, not a check

try {
  await axios.post('/api/users', { name: 'A' });
} catch (e) {
  if (isAxiosError(e)) console.log(e.response?.status); // narrowed to AxiosError
}
```

axios ships its own types; do not install `@types/axios`. You can still validate: `UserSchema.array().parse(data)`.

**4. ❌ Wrong / ✅ Right**

```ts
const users = (await res.json()) as User[]; // ❌ unchecked
const users2 = await fetchUsers(); // ✅ validated by Zod
```

Also remember `fetch` does **not** reject on HTTP 404/500. You must check `res.ok`. axios rejects on non-2xx by default.

**5. Real-world use**
A base URL from `import.meta.env`, an auth header added in one place, and one `ApiError` that the UI maps to messages.

**6. Common mistakes & errors**

- `TS18046: 'e' is of type 'unknown'` in `catch`. Narrow with `instanceof ApiError` or `isAxiosError`.
- Parsing the response when the body is empty (204). `res.json()` throws. Handle it separately.
- Putting the auth token into a Zod-less global. Keep it in one request wrapper.

**7. Interview tip**
_"How do you type API responses in React?"_ Say that generics only claim a type; you validate with a schema so the type is backed by a runtime check.

---

## 10.18 TanStack Query typing 🔴 MUST KNOW

**1. What it is**
TanStack Query (formerly React Query) caches server data, handles loading and error states, and refetches for you. It replaces most `useEffect` + `useState` fetching. The examples use **v5**.

**2. Why it exists**
Hand-written fetch effects (10.14) need cancellation, races, caching, and retries. The library does these once, with types inferred from your `queryFn`.

**3. Code**

```bash
npm install @tanstack/react-query
```

```tsx
// main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
const queryClient = new QueryClient();
// <QueryClientProvider client={queryClient}><App /></QueryClientProvider>
```

```tsx
import {
  useQuery,
  useMutation,
  useQueryClient,
  queryOptions,
} from '@tanstack/react-query';

// queryOptions: define key + fn ONCE; reuse with useQuery, prefetch, invalidate
export const usersQuery = queryOptions({
  queryKey: ['users'],
  queryFn: fetchUsers, // data type is inferred: User[]
});

function UserList() {
  const { data, isPending, isError, error } = useQuery(usersQuery);

  if (isPending) return <p>Loading…</p>;
  if (isError) return <p>{error.message}</p>; // error: Error (v5 default)
  return (
    <ul>
      {data.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  ); // data: User[]
}
// Before the checks, `data` is `User[] | undefined`. After them, TS has narrowed it.
```

If narrowing of destructured values ever fails, use the whole object: `const q = useQuery(...); if (q.status === "success") q.data`.

**Query with a parameter**

```tsx
const userQuery = (id: number) =>
  queryOptions({
    queryKey: ['users', id],
    queryFn: () => request(`/users/${id}`, UserSchema),
  });

const { data } = useQuery(userQuery(5));
```

**Mutations: generics are `<TData, TError, TVariables, TContext>`, usually inferred**

```tsx
interface CreateUserInput {
  name: string;
  email: string;
}

function AddUser() {
  const queryClient = useQueryClient();
  const createUser = useMutation({
    mutationFn: (input: CreateUserInput) =>
      request('/users', UserSchema, {
        method: 'POST',
        body: JSON.stringify(input),
      }),
    onSuccess: () =>
      queryClient.invalidateQueries({ queryKey: usersQuery.queryKey }),
  });

  createUser.mutate({ name: 'Asha', email: 'a@x.com' }); // ✅
  createUser.mutate({ name: 'Asha' });
  // ❌ TS2345: Argument of type '{ name: string; }' is not assignable to parameter of type 'CreateUserInput'.
  //    Property 'email' is missing ...
  return <button disabled={createUser.isPending}>Add</button>;
}
```

**4. ❌ Wrong / ✅ Right**

```tsx
const { data } = useQuery(usersQuery);
data.map((u) => u.name);
// ❌ TS18048: 'data' is possibly 'undefined'.
// ✅ check isPending/isError first, or use `data?.map`

const { isLoading } = useQuery(usersQuery);
// ⚠️ v4 → v5: `isLoading` was renamed `isPending` (first load). In v5 `isLoading` means
//    "pending AND fetching". Many tutorials still show the v4 meaning.
```

**v4 vs v5 (you will meet both):** v5 uses a single options object (`useQuery({ queryKey, queryFn })`; the old `useQuery(key, fn)` signature is gone), removed `onSuccess`/`onError` callbacks from `useQuery`, and renamed `cacheTime` to `gcTime`. Import paths and defaults changed too, so read the migration notes when you open an old project.

**5. Real-world use**
Lists, detail pages, search with `useDebounce` (10.15) in the key, optimistic updates, and infinite scroll (`useInfiniteQuery`). Use `queryKey` factories to keep keys consistent.

**6. Common mistakes**

- `error` is typed `Error` by default. If you throw `ApiError`, narrow with `instanceof ApiError`, or register a global error type (see the TanStack docs for your version).
- Passing a `queryFn` that casts. The type is only as true as the function behind it. Use the validated `request` helper.
- Forgetting a variable in `queryKey`, so two different requests share one cache entry.

**7. Interview tip**
_"Why TanStack Query instead of `useEffect`?"_ Caching, deduplication, retries, and states handled for you, with `data` typed from the query function.

---

## 10.19 React Hook Form + Zod 🔴 MUST KNOW

**1. What it is**
React Hook Form (RHF) manages form state with little re-rendering. `zodResolver` plugs in a Zod schema, so the **same schema validates the form and types its values**. This is the same schema idea as your backend DTO (9.12).

**3. Code**

```bash
npm install react-hook-form zod @hookform/resolvers
```

```tsx
import { useForm, type SubmitHandler } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const SignupSchema = z.object({
  email: z.string().email('Enter a valid email'),
  password: z.string().min(8, 'At least 8 characters'),
});
type SignupValues = z.infer<typeof SignupSchema>;

function SignupForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<SignupValues>({
    resolver: zodResolver(SignupSchema),
    defaultValues: { email: '', password: '' },
  });

  const onSubmit: SubmitHandler<SignupValues> = async (values) => {
    await request('/signup', z.unknown(), {
      method: 'POST',
      body: JSON.stringify(values),
    });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input type="email" {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}

      <input type="password" {...register('password')} />
      {errors.password?.message}

      <input {...register('emial')} />
      {/* ❌ TS2345: Argument of type '"emial"' is not assignable to parameter of type '"email" | "password"'. */}

      <button disabled={isSubmitting}>Sign up</button>
    </form>
  );
}
```

Field names are checked against the schema, and `errors.email?.message` is typed `string | undefined`.

**Input vs output types:** if your schema uses `z.coerce`, `.default()` or `.transform()`, the form's **input** type differs from the validated **output** type (8.7). Then `useForm<SignupValues>` with `z.infer` can produce resolver type errors such as `TS2322: Type 'Resolver<...>' is not assignable to type 'Resolver<...>'`. Recent versions let you write `useForm<z.input<typeof S>, unknown, z.output<typeof S>>(...)`. The exact generics and the required `@hookform/resolvers` version depend on your RHF and Zod versions (Zod 4 needs a recent resolvers release), so check the docs when you see this error.

**4. ❌ Wrong / ✅ Right**

```tsx
// ❌ Number inputs return strings. z.number() will fail with "expected number, received string"
age: z.number();
// ✅ age: z.coerce.number()   (and remember the input/output type difference above)
//    or register("age", { valueAsNumber: true })
```

Use `handleSubmit(onSubmit)`, never `onSubmit={onSubmit}`. Otherwise validation and `preventDefault` are skipped.

**5. Real-world use**
Login, signup, checkout, and profile edit. Share the schema with the backend (Part 11) so client and server rules never drift. Server-side errors (such as "email taken") map back with `setError("email", { message })`.

**6. Common mistakes & errors**

- Missing `defaultValues`. Fields start `undefined`, which causes uncontrolled-to-controlled warnings.
- Using `register` on a custom component that does not forward refs (10.8). Use `Controller` for those.
- Duplicating the type by hand next to the schema. Always `z.infer`.

**7. Interview tip**
_"How do you type a form?"_ A Zod schema, `z.infer` for the values, `zodResolver` in `useForm<Values>`. Add that it gives one source of truth for validation and types.

---

## 10.20 Redux Toolkit typing 🟡 GOOD TO KNOW (🔴 if the job uses Redux)

**1. What it is**
Redux Toolkit (RTK) is the standard way to write Redux. You need four typed pieces: the **store types**, **typed hooks**, **slices**, and **async thunks**.

**3. Code**

```ts
// src/store/store.ts
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './cartSlice';

export const store = configureStore({ reducer: { cart: cartReducer } });

export type RootState = ReturnType<typeof store.getState>; // derived, never hand-written (5.6)
export type AppDispatch = typeof store.dispatch;

// src/store/hooks.ts: use these everywhere instead of plain useDispatch/useSelector
import { useDispatch, useSelector } from 'react-redux';
export const useAppDispatch = useDispatch.withTypes<AppDispatch>(); // react-redux 9.1+
export const useAppSelector = useSelector.withTypes<RootState>();
// Older code: useDispatch<AppDispatch>() and `const useAppSelector: TypedUseSelectorHook<RootState> = useSelector`
```

```ts
// src/store/cartSlice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

interface CartItem {
  productId: string;
  qty: number;
}
interface CartState {
  items: CartItem[];
}

const initialState: CartState = { items: [] }; // annotate, or `items` infers never[]

const cartSlice = createSlice({
  name: 'cart',
  initialState,
  reducers: {
    added(state, action: PayloadAction<CartItem>) {
      state.items.push(action.payload); // "mutation" is safe: RTK uses Immer
    },
    removed(state, action: PayloadAction<string>) {
      state.items = state.items.filter((i) => i.productId !== action.payload);
    },
    cleared(state) {
      state.items = [];
    },
  },
});

export const { added, removed, cleared } = cartSlice.actions;
export default cartSlice.reducer;
```

```tsx
function CartButton() {
  const count = useAppSelector((s) => s.cart.items.length); // s is RootState: autocomplete works
  const dispatch = useAppDispatch();
  dispatch(added({ productId: 'p1', qty: 1 })); // ✅
  dispatch(added({ productId: 'p1' }));
  // ❌ TS2345: Argument of type '{ productId: string; }' is not assignable to parameter of type 'CartItem'.
  return <span>{count}</span>;
}
```

**Async thunks**

```ts
import { createAsyncThunk } from '@reduxjs/toolkit';

interface Order {
  id: string;
  total: number;
}

//                                   Returned    Argument  Config
export const fetchOrders = createAsyncThunk<
  Order[],
  string,
  { rejectValue: string }
>('orders/fetch', async (userId, { rejectWithValue }) => {
  try {
    return await request(
      `/users/${userId}/orders`,
      z.array(z.object({ id: z.string(), total: z.number() })),
    );
  } catch (e) {
    return rejectWithValue(e instanceof Error ? e.message : 'Unknown error');
  }
});

// In the slice:
// extraReducers: (builder) => {
//   builder
//     .addCase(fetchOrders.pending, (state) => { state.status = "loading"; })
//     .addCase(fetchOrders.fulfilled, (state, action) => { state.items = action.payload; })   // Order[]
//     .addCase(fetchOrders.rejected, (state, action) => { state.error = action.payload ?? "Failed"; });  // string | undefined
// }
```

Use the **builder callback** form of `extraReducers`. The old object form is removed in RTK 2.

**4. ❌ Wrong / ✅ Right**

```ts
useDispatch()(fetchOrders('1'));
// ❌ TS2345: Argument of type 'AsyncThunkAction<...>' is not assignable to parameter of type 'UnknownAction'.
// ✅ the plain dispatch does not know thunks. Use useAppDispatch.

export type RootState = { cart: CartState }; // ❌ hand-written: drifts when you add a slice
```

**5. Real-world use**
Client-side state shared across pages: cart, auth session, UI flags. For **server data**, RTK Query (part of RTK) or TanStack Query is the better fit than a hand-rolled thunk.

**6. Common mistakes**

- Putting non-serializable values (Dates, class instances) in state, which triggers a dev-time warning.
- Returning a new value **and** mutating `state` in the same reducer. Do one or the other.
- Forgetting to add the slice reducer to `configureStore`.

**7. Interview tip**
_"How do you type Redux hooks?"_ Derive `RootState` and `AppDispatch` from the store and export typed `useAppDispatch`/`useAppSelector`.

---

## 10.21 React Router typing 🟡 GOOD TO KNOW

**1. What it is**
Router hooks read the URL. Everything in a URL is a **string**, so TS correctly makes params optional.

**3. Code**

```tsx
import {
  useParams,
  useNavigate,
  useSearchParams,
  Link,
} from 'react-router-dom';

function UserPage() {
  const { id } = useParams<{ id: string }>(); // { id: string | undefined }
  const navigate = useNavigate();
  const [searchParams, setSearchParams] = useSearchParams();

  if (!id) return <p>Missing id</p>; // narrow before use
  const userId = Number(id);
  if (Number.isNaN(userId)) return <p>Invalid id</p>;

  const page = Number(searchParams.get('page') ?? '1'); // get() returns string | null

  return (
    <>
      <Link to={`/users/${userId}/edit`}>Edit</Link>
      <button onClick={() => navigate('/users')}>Back</button>
    </>
  );
}
```

`useParams<...>()` is a **claim** about route params (like `Request<{ id: string }>` in 9.4), and the compiler cannot connect it to your route path string.

**Loaders (data routers)**

```tsx
import type { LoaderFunctionArgs } from 'react-router-dom';
import { useLoaderData } from 'react-router-dom';

export async function userLoader({
  params,
}: LoaderFunctionArgs): Promise<User> {
  return request(`/users/${params.id}`, UserSchema);
}
const user = useLoaderData() as User; // ⚠️ cast: useLoaderData returns unknown/any
```

**Version note:** React Router v7 is installed from `react-router` (with `react-router-dom` kept as a re-export). Its "framework mode" can **generate** route-aware types, so `loaderData` is typed automatically. That differs from the manual approach above, so check which mode and version your project uses.

**4. ❌ Wrong / ✅ Right**

```tsx
const { id } = useParams();
fetchUser(id);
// ❌ TS2345: Argument of type 'string | undefined' is not assignable to parameter of type 'string'.
// ✅ guard first (above), never `id!` unless the route guarantees it
```

**6. Common mistakes**
Typing a param as `number` (`useParams<{ id: number }>`). It fails the constraint, and it would be a lie anyway (`TS2344`, params must be string-valued).

---

## 10.22 Styling props 🟡 GOOD TO KNOW

**3. Code**

```tsx
import type { CSSProperties } from 'react';

interface BoxProps {
  className?: string;
  style?: CSSProperties; // typed CSS: autocompletes `display`, `gap`, ...
  tone?: 'info' | 'warning';
}

function Box({ className, style, tone = 'info' }: BoxProps) {
  return (
    <div
      className={`box box-${tone} ${className ?? ''}`}
      style={{ padding: 8, ...style }}
    />
  );
}

<Box style={{ colour: 'red' }} />;
// ❌ TS2322: Type '{ colour: string; }' is not assignable to type 'CSSProperties'. Did you mean 'color'?

// CSS custom properties are not in CSSProperties, so a cast is the usual fix
<div style={{ '--accent': 'tomato' } as CSSProperties} />; // ⚠️ small, accepted `as`
```

- **Tailwind:** class names are plain strings, so TS cannot check them. Use an editor plugin and a linter.
- **Variant libraries** (`class-variance-authority`) derive prop types: `type Props = VariantProps<typeof buttonStyles>`.
- **CSS Modules:** `import styles from "./x.module.css"` works through the `vite/client` types; keys are loosely typed (7.4).

---

## 10.23 Common React + TS errors and fixes 🔴 MUST KNOW

| Error                                                           | Usual cause                                                       | Fix                                                                |
| --------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------ |
| `TS7006` / `TS7031`: parameter/binding implicitly `any`         | Untyped props or handler defined away from its JSX                | Type the props object or the event (10.2, 10.6)                    |
| `TS2322` on `children`                                          | `children` missing, or typed as `string`/`JSX.Element`            | `children: ReactNode` (10.3)                                       |
| `TS18047` / `TS18048`: possibly `null`/`undefined`              | `useState<T \| null>`, `ref.current`, query `data`                | Narrow, `?.`, or early return                                      |
| `TS2339: Property 'value' does not exist on type 'EventTarget'` | Used `e.target` on a click event                                  | `e.currentTarget`, or `ChangeEvent<HTMLInputElement>`              |
| `TS2345` on `setState`                                          | Setter given the wrong type, or `useState([])` inferred `never[]` | `useState<Item[]>([])`                                             |
| `TS2769: No overload matches this call`                         | A prop value mismatches a native element's prop                   | Read the **last** nested line (0.7)                                |
| `TS2786: 'X' cannot be used as a JSX component`                 | Duplicate or mismatched `@types/react` versions in `node_modules` | One version: `npm ls @types/react`, then dedupe or add `overrides` |
| `TS7026: JSX element implicitly has type 'any'`                 | `@types/react` missing, or `jsx` not set in tsconfig              | Install types; `"jsx": "react-jsx"`                                |
| `TS2304: Cannot find name 'T'`                                  | Generic on a `const` arrow typed with `React.FC`                  | Function declaration or `<T,>` (10.16)                             |
| `TS2307: Cannot find module './x.svg'`                          | `vite/client` types missing                                       | `vite-env.d.ts` reference (10.1)                                   |
| `TS2554: Expected 1 arguments, but got 0` on `useRef`           | React 19 types require an initial value                           | `useRef<T>(null)` (10.11)                                          |
| `TS2740: Type 'Promise<User>' is missing...`                    | Missing `await`                                                   | Await, or use Query                                                |

**Errors TS cannot give you** (runtime or lint-only): hooks called conditionally (`react-hooks/rules-of-hooks`), missing `useEffect` dependencies (`exhaustive-deps`), missing `key` props on lists (React warns at runtime), and calling `setState` during render.

**Habit that saves hours:** when a React type error is huge, check the **first** error in the file, hover over the variable to see its real inferred type, and look at the **last** nested line.

**Interview tip:** _"What are common React + TypeScript mistakes?"_ Typing `children` too narrowly, trusting `as` on API data, forgetting `null` states for refs and queries, and using `e.target` instead of `e.currentTarget`.

---

### 📌 Key Takeaways

- A typed `fetch<T>`/`axios.get<T>` is only a claim. Validate responses with a Zod schema in one `request` helper, and check `res.ok`.
- TanStack Query v5: `data` is `T | undefined` until `isPending`/`isError` are handled; define `queryOptions` once; know the v4→v5 renames.
- React Hook Form + `zodResolver`: derive values with `z.infer`; when the schema coerces or defaults, input and output types differ, so check your versions' docs.
- Redux Toolkit: derive `RootState` and `AppDispatch` from the store, export typed hooks, use `PayloadAction<T>` and the builder form of `extraReducers`.
- URL params and search params are strings; `useParams` gives `string | undefined`, so guard and convert.
- Most React type errors repeat the same patterns (null states, event types, `children`, `@types/react` duplicates); keep the table above handy.

✅ Part 10c complete.
