# React.js Cheatsheet

## Mental Model

React is a **UI library** built around one idea: UI is a function of state — `UI = f(state)`. When state changes, React re-renders the component tree efficiently via a virtual DOM diff. You compose small, reusable components and let React handle DOM updates. You never touch the DOM directly.

---

## Install & Minimal Setup

```bash
# Vite (recommended — faster than CRA)
npm create vite@latest my-app -- --template react
cd my-app && npm install && npm run dev

# Next.js (full-stack / SSR)
npx create-next-app@latest my-app
```

---

## Core Concepts

### 1. Components
Functions that return JSX. One responsibility per component.

```jsx
// Functional component — always use this, never class components
function UserCard({ name, role }) {
  return (
    <div className="card">
      <h2>{name}</h2>
      <p>{role}</p>
    </div>
  );
}
```

### 2. Props
Read-only inputs passed from parent to child. Destructure them directly.

```jsx
// Parent
<UserCard name="Joshua" role="AI Engineer" />

// Child — destructure in signature
function UserCard({ name, role }) { ... }

// children prop — for composition
function Card({ children }) {
  return <div className="card">{children}</div>;
}
```

### 3. State — `useState`
Local, mutable data. Triggers re-render on change. Never mutate state directly.

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(prev => prev + 1)}>
      Clicked {count} times
    </button>
  );
}
```

### 4. Side Effects — `useEffect`
Runs after render. Used for fetching, subscriptions, timers.

```jsx
import { useEffect, useState } from "react";

function UserList() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    // Runs once on mount (empty dependency array)
    fetch("/api/users")
      .then(res => res.json())
      .then(data => setUsers(data));

    // Return cleanup function if needed
    return () => { /* cancel subscription, clear timer */ };
  }, []); // [] = run once | [id] = run when id changes | no array = run every render

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### 5. Derived State & `useMemo`
Compute values from state without storing them separately.

```jsx
const filteredUsers = useMemo(
  () => users.filter(u => u.role === "admin"),
  [users] // recompute only when users changes
);
```

### 6. `useCallback`
Memoize functions passed to child components to avoid unnecessary re-renders.

```jsx
const handleDelete = useCallback((id) => {
  setUsers(prev => prev.filter(u => u.id !== id));
}, []); // stable reference across renders
```

### 7. `useRef`
Access DOM nodes or store mutable values that don't trigger re-renders.

```jsx
const inputRef = useRef(null);

// Focus an input programmatically
<input ref={inputRef} />
<button onClick={() => inputRef.current.focus()}>Focus</button>
```

### 8. Context — `useContext`
Share state across the tree without prop drilling. Not a replacement for a state manager.

```jsx
// 1. Create context
const ThemeContext = createContext("light");

// 2. Provide it high in the tree
<ThemeContext.Provider value="dark">
  <App />
</ThemeContext.Provider>

// 3. Consume anywhere below
function Button() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}
```

### 9. Custom Hooks
Extract and reuse stateful logic. Must start with `use`.

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => { setData(data); setLoading(false); });
  }, [url]);

  return { data, loading };
}

// Usage
const { data, loading } = useFetch("/api/users");
```

---

## Most-Used Patterns

### Conditional Rendering

```jsx
// Short-circuit (show or nothing)
{isLoggedIn && <Dashboard />}

// Ternary (show one or the other)
{isLoading ? <Spinner /> : <Content />}

// Guard clause in component body
if (error) return <ErrorMessage error={error} />;
if (loading) return <Spinner />;
return <Content data={data} />;
```

### Lists & Keys

```jsx
// key must be stable and unique — never use array index if list can reorder
<ul>
  {items.map(item => (
    <li key={item.id}>{item.name}</li>
  ))}
</ul>
```

### Controlled Inputs

```jsx
const [email, setEmail] = useState("");

<input
  value={email}
  onChange={e => setEmail(e.target.value)}
  type="email"
/>
```

### Lifting State Up

```jsx
// When two siblings need shared state, lift it to their common parent
function Parent() {
  const [value, setValue] = useState("");
  return (
    <>
      <InputChild value={value} onChange={setValue} />
      <DisplayChild value={value} />
    </>
  );
}
```

### Error Boundaries (class component — wrap around async-heavy trees)

```jsx
// Use react-error-boundary package instead of writing manually
import { ErrorBoundary } from "react-error-boundary";

<ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <RiskyComponent />
</ErrorBoundary>
```

---

## Gotchas

- **Never mutate state directly** — `state.push(x)` won't trigger a re-render. Use `setState([...state, x])`.
- **Stale closures in useEffect** — if you read state inside an effect, add it to the dependency array or use the functional updater form `setState(prev => ...)`.
- **Missing `key` on lists** — causes subtle bugs on reorder/delete. Always use a stable unique ID.
- **useEffect with no cleanup** — subscriptions and timers must return a cleanup function or you get memory leaks.
- **Overusing Context** — Context re-renders all consumers on every value change. For complex state, prefer Zustand or Jotai.
- **`useCallback` / `useMemo` are not free** — don't add them preemptively. Add them when you have a measured perf issue or a referentially stable dep is required.

---

## Quick Links

- [React Docs (beta)](https://react.dev) — the official docs are excellent and hook-based
- [useHooks](https://usehooks.com) — well-explained custom hook patterns
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app)
- [Zustand](https://github.com/pmndrs/zustand) — minimal state manager when Context isn't enough
