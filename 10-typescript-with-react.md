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
