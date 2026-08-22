# ReactJS Complete Notes (Beginner → Advanced)
### Modern React (Hooks only, React 18+)

---

## Table of Contents
1. [Introduction to React](#1-introduction-to-react)
2. [How React Works (Core Concepts)](#2-how-react-works-core-concepts)
3. [Setting Up a React Project](#3-setting-up-a-react-project)
4. [JSX](#4-jsx)
5. [Components](#5-components)
6. [Props](#6-props)
7. [State — `useState`](#7-state--usestate)
8. [Event Handling](#8-event-handling)
9. [Conditional Rendering](#9-conditional-rendering)
10. [Lists and Keys](#10-lists-and-keys)
11. [Forms & Controlled Components](#11-forms--controlled-components)
12. [Component Lifecycle & `useEffect`](#12-component-lifecycle--useeffect)
13. [`useRef`](#13-useref)
14. [`useContext`](#14-usecontext)
15. [`useReducer`](#15-usereducer)
16. [`useMemo` & `useCallback`](#16-usememo--usecallback)
17. [Custom Hooks](#17-custom-hooks)
18. [React Router](#18-react-router)
19. [State Management (Context vs Redux Toolkit)](#19-state-management-context-vs-redux-toolkit)
20. [Performance Optimization](#20-performance-optimization)
21. [Error Boundaries](#21-error-boundaries)
22. [React 18 Features](#22-react-18-features)
23. [Component Design Patterns](#23-component-design-patterns)
24. [Testing React Apps](#24-testing-react-apps)
25. [Common Interview Questions](#25-common-interview-questions)
26. [Cheat Sheet Summary](#26-cheat-sheet-summary)

---

## 1. Introduction to React

**React** is a JavaScript **library** (not a framework) for building user interfaces, created by Facebook (Meta). It lets you build encapsulated **components** that manage their own state, then compose them to make complex UIs.

### Why React?
- **Declarative**: You describe *what* the UI should look like for a given state; React figures out *how* to update the DOM.
- **Component-Based**: UI is broken into independent, reusable pieces.
- **Virtual DOM**: React uses an in-memory representation of the DOM to make updates fast.
- **Learn Once, Write Anywhere**: React Native for mobile, React DOM for web.

### React vs Vanilla JS (Imperative vs Declarative)
```
Vanilla JS (Imperative)              React (Declarative)
------------------------             ------------------------
1. Find the DOM node                 1. Describe the UI as a
2. Manually update it                   function of state
3. Track state yourself              2. Call setState
4. Repeat for every change           3. React re-renders automatically
```

---

## 2. How React Works (Core Concepts)

### The Virtual DOM (VDOM)
React keeps a lightweight copy of the real DOM in memory. When state changes:

```
 State Changes
      │
      ▼
 New Virtual DOM tree is created
      │
      ▼
 React "diffs" new VDOM vs old VDOM   (Reconciliation)
      │
      ▼
 Only the CHANGED nodes are updated in the Real DOM
```

**Diagram: Rendering Flow**
```
   ┌────────────┐      ┌───────────────┐      ┌────────────────┐
   │  Component  │ ---> │  Virtual DOM   │ ---> │   Real DOM      │
   │ (JSX + State)│     │ (JS Objects)   │      │ (Browser paints)│
   └────────────┘      └───────────────┘      └────────────────┘
          ▲                                            │
          └──────────── setState triggers re-render ───┘
```

### Reconciliation & Fiber
React's diffing algorithm (called **Fiber** since React 16) compares trees using these rules:
- Different element **types** → tear down old tree, build new one.
- Same type → keep the DOM node, only update changed attributes.
- Lists use **`key`** props to match items between renders (see Section 10).

### One-Way Data Flow
Data flows **parent → child** via props. Children communicate back to parents via **callback functions** passed as props.

```
        App (state)
        /        \
   props↓          ↓props
  Header          Content
                     │
                callback fn passed down
                     │
                Button (calls onClick prop)
```

---

## 3. Setting Up a React Project

### Option 1: Vite (recommended, fast)
```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev
```

### Option 2: Create React App (legacy, slower, still common in tutorials)
```bash
npx create-react-app my-app
cd my-app
npm start
```

### Project Structure (typical)
```
my-app/
├── public/
│   └── index.html
├── src/
│   ├── main.jsx          # Entry point
│   ├── App.jsx           # Root component
│   ├── components/       # Reusable components
│   ├── pages/             # Route-level components
│   ├── hooks/             # Custom hooks
│   └── assets/
├── package.json
└── vite.config.js
```

### Entry Point Example
```jsx
// main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App.jsx';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

---

## 4. JSX

**JSX** (JavaScript XML) lets you write HTML-like syntax inside JavaScript. It's syntactic sugar for `React.createElement()`.

```jsx
const element = <h1>Hello, world!</h1>;

// Compiles roughly to:
const element = React.createElement('h1', null, 'Hello, world!');
```

### JSX Rules
| Rule | Example |
|---|---|
| Must return a **single root element** (or Fragment) | `<>...</>` |
| Use `className` instead of `class` | `<div className="box">` |
| Use `htmlFor` instead of `for` | `<label htmlFor="name">` |
| camelCase for attributes | `onClick`, `tabIndex` |
| JS expressions go inside `{ }` | `<p>{2 + 2}</p>` |
| Self-close empty tags | `<img />`, `<br />` |

### Embedding Expressions
```jsx
const name = "Prathamesh";
const el = <h1>Hello, {name.toUpperCase()}!</h1>;
```

### Fragments (avoid extra wrapper divs)
```jsx
function List() {
  return (
    <>
      <li>Item 1</li>
      <li>Item 2</li>
    </>
  );
}
```

---

## 5. Components

Components are **JavaScript functions** that return JSX. Modern React uses **Function Components + Hooks** (class components are legacy — not covered here since you chose Hooks-only).

### Basic Function Component
```jsx
function Welcome() {
  return <h1>Welcome to React!</h1>;
}

// Arrow function equivalent
const Welcome = () => {
  return <h1>Welcome to React!</h1>;
};

export default Welcome;
```

### Composing Components
```jsx
function App() {
  return (
    <div>
      <Header />
      <Welcome />
      <Footer />
    </div>
  );
}
```

**Component Tree Diagram**
```
                App
              /  |  \
        Header Welcome Footer
```

### Naming Rule
Component names **must start with a capital letter** (`<Welcome />` not `<welcome />`), otherwise React treats it as an HTML tag.

---

## 6. Props

**Props** (properties) pass data from parent to child. They are **read-only** (immutable inside the child).

```jsx
function Greeting(props) {
  return <h2>Hello, {props.name}!</h2>;
}

// Usage
<Greeting name="Prathamesh" />
```

### Destructuring Props (common style)
```jsx
function Greeting({ name, age }) {
  return <h2>{name} is {age} years old</h2>;
}
```

### Default Props
```jsx
function Greeting({ name = "Guest" }) {
  return <h2>Hello, {name}</h2>;
}
```

### `children` Prop (composition)
```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}

// Usage
<Card>
  <p>This content becomes `props.children`</p>
</Card>
```

### Props Flow Diagram
```
 Parent Component
   state = { name: "Prathamesh" }
        │
        │  <Child name={state.name} />
        ▼
 Child Component receives props.name (READ-ONLY)
```

---

## 7. State — `useState`

**State** is data that changes over time and triggers a re-render when updated. Unlike props, state is **owned and managed inside** the component.

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0); // [value, setter] = useState(initialValue)

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(count - 1)}>-1</button>
    </div>
  );
}
```

### Key Rules
- Calling the setter **schedules a re-render**; it doesn't mutate state immediately.
- State updates may be **batched** (multiple `setState` calls in one event = one re-render).
- Use the **functional updater** form when new state depends on old state:
```jsx
setCount(prevCount => prevCount + 1); // safe even with batching
```
- **Never mutate state directly** (e.g., `state.push(x)` on an array) — always create a new object/array:
```jsx
setItems(prevItems => [...prevItems, newItem]);   // ✅ correct
setUser(prev => ({ ...prev, name: 'New Name' }));  // ✅ correct for objects
```

### State with Objects/Arrays
```jsx
const [user, setUser] = useState({ name: '', email: '' });

function updateName(newName) {
  setUser(prev => ({ ...prev, name: newName })); // spread to keep other fields
}
```

---

## 8. Event Handling

React wraps native events in **SyntheticEvents** for cross-browser consistency.

```jsx
function Button() {
  const handleClick = (e) => {
    console.log('Button clicked', e.target);
  };

  return <button onClick={handleClick}>Click Me</button>;
}
```

### Passing Arguments to Handlers
```jsx
function ItemList({ items }) {
  const handleDelete = (id) => {
    console.log('Deleting item', id);
  };

  return items.map(item => (
    <button key={item.id} onClick={() => handleDelete(item.id)}>
      Delete {item.name}
    </button>
  ));
}
```

### Common Events
| Event | Trigger |
|---|---|
| `onClick` | Mouse click |
| `onChange` | Input value changes |
| `onSubmit` | Form submission |
| `onMouseEnter` / `onMouseLeave` | Hover |
| `onKeyDown` / `onKeyUp` | Keyboard |
| `onFocus` / `onBlur` | Input focus/blur |

---

## 9. Conditional Rendering

### 1. `if` / early return
```jsx
function Greeting({ isLoggedIn }) {
  if (isLoggedIn) return <h1>Welcome back!</h1>;
  return <h1>Please sign in.</h1>;
}
```

### 2. Ternary Operator
```jsx
<div>{isLoggedIn ? <Dashboard /> : <Login />}</div>
```

### 3. Logical `&&` (render only if true)
```jsx
<div>{errors.length > 0 && <ErrorBanner errors={errors} />}</div>
```
> ⚠️ Gotcha: `{count && <p>Items: {count}</p>}` renders `0` if `count` is `0` (falsy but not "nothing"). Use `count > 0 && ...` instead.

### 4. Switch-like patterns with object maps
```jsx
const statusMessages = {
  loading: <Spinner />,
  error: <ErrorMessage />,
  success: <DataView />,
};
return statusMessages[status] ?? null;
```

---

## 10. Lists and Keys

Rendering arrays with `.map()`. Every list item needs a **unique, stable `key`** so React can track identity across re-renders.

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

### Why Keys Matter (Diagram)
```
Without stable keys (using array index and reordering items):
  Render 1:  [A, B, C]  → keys 0,1,2
  Render 2:  [C, A, B]  → keys 0,1,2  (React thinks item at index 0 changed A→C!)
  Result: React re-uses the wrong DOM node → bugs with inputs/animations

With stable unique keys (e.g. todo.id):
  Render 1: A(id:1) B(id:2) C(id:3)
  Render 2: C(id:3) A(id:1) B(id:2)
  Result: React correctly matches nodes by id → moves them, no bugs
```
> ❌ Avoid using array **index** as key if the list can be reordered, filtered, or items inserted/removed.
> ✅ Use a unique id from your data (`todo.id`, `user._id`, etc.)

---

## 11. Forms & Controlled Components

A **controlled component** ties an input's value to React state — React is the "single source of truth."

```jsx
function SignupForm() {
  const [form, setForm] = useState({ email: '', password: '' });

  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log('Submitting:', form);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" value={form.email} onChange={handleChange} />
      <input name="password" type="password" value={form.password} onChange={handleChange} />
      <button type="submit">Sign Up</button>
    </form>
  );
}
```

### Controlled vs Uncontrolled
| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | The DOM itself |
| Access value via | `value` + `onChange` | `ref` |
| Use case | Most forms, validation | Simple/one-off, file inputs |

```jsx
// Uncontrolled example (using useRef)
function UncontrolledInput() {
  const inputRef = useRef(null);
  const handleSubmit = () => alert(inputRef.current.value);
  return <><input ref={inputRef} /><button onClick={handleSubmit}>Show</button></>;
}
```

---

## 12. Component Lifecycle & `useEffect`

Function components don't have lifecycle methods like classes (`componentDidMount`), but `useEffect` covers all of them.

```jsx
import { useEffect, useState } from 'react';

function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    console.log('Effect runs after render (like componentDidMount + componentDidUpdate)');

    const id = setInterval(() => setSeconds(s => s + 1), 1000);

    // Cleanup function — runs before next effect & on unmount (like componentWillUnmount)
    return () => clearInterval(id);
  }, []); // dependency array

  return <p>Seconds: {seconds}</p>;
}
```

### The Dependency Array Controls WHEN the Effect Runs
| Dependency Array | Runs |
|---|---|
| *(omitted)* | After **every** render |
| `[]` | Only **once**, after first render (mount) |
| `[a, b]` | After first render **and** whenever `a` or `b` changes |

### Lifecycle Mapping (Class → Hooks)
```
componentDidMount        →  useEffect(() => {...}, [])
componentDidUpdate       →  useEffect(() => {...}, [dep1, dep2])
componentWillUnmount     →  useEffect(() => { return () => {...cleanup} }, [])
```

### `useEffect` Execution Flow Diagram
```
   Render 1 (mount)
        │
        ▼
   Paint to screen
        │
        ▼
   Run Effect
        │
   (state/props change → Render 2)
        │
        ▼
   Run CLEANUP of previous effect
        │
        ▼
   Run new Effect
        │
   (component unmounts)
        │
        ▼
   Run final CLEANUP
```

### Common `useEffect` Use Cases
```jsx
// 1. Data fetching
useEffect(() => {
  let ignore = false;
  fetch(`/api/users/${userId}`)
    .then(res => res.json())
    .then(data => { if (!ignore) setUser(data); });
  return () => { ignore = true; }; // avoid race conditions
}, [userId]);

// 2. Subscribing to events
useEffect(() => {
  const handleResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

---

## 13. `useRef`

`useRef` creates a mutable value that **persists across renders without causing re-renders** when changed. Common uses: accessing DOM nodes, storing previous values, timers.

```jsx
import { useRef, useEffect } from 'react';

function FocusInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus(); // direct DOM access
  }, []);

  return <input ref={inputRef} />;
}
```

### `useRef` vs `useState`
| | `useState` | `useRef` |
|---|---|---|
| Triggers re-render on change? | ✅ Yes | ❌ No |
| Value persists across renders? | ✅ Yes | ✅ Yes |
| Use for | UI-driving data | DOM refs, timers, mutable "instance" vars |

```jsx
// Storing a mutable value that doesn't need to re-render UI
function StopWatch() {
  const [, forceRender] = useState(0);
  const startTime = useRef(Date.now());
  const intervalRef = useRef(null); // stores interval ID
  // ...
}
```

---

## 14. `useContext`

**Context** lets you pass data through the component tree without manually passing props at every level ("prop drilling").

### The Prop Drilling Problem
```
Without Context:
App(theme) → Layout(theme) → Sidebar(theme) → Avatar(theme)
   (theme prop passed through EVERY level even if unused there)

With Context:
App (ThemeProvider value=theme)
        │
   ┌────┴─────┐
 Layout    (no props needed)
        │
    Sidebar
        │
    Avatar  ← useContext(ThemeContext) — direct access!
```

### Implementation
```jsx
// 1. Create the context
import { createContext, useContext, useState } from 'react';
const ThemeContext = createContext(null);

// 2. Provide it at a high level
function App() {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

// 3. Consume it anywhere below
function Avatar() {
  const { theme } = useContext(ThemeContext);
  return <img className={`avatar avatar--${theme}`} />;
}
```
> Best for: theming, authenticated user info, locale/language, global UI state. Not a full replacement for Redux in large apps with complex/frequent updates (can cause unnecessary re-renders of all consumers).

---

## 15. `useReducer`

An alternative to `useState` for **complex state logic** (multiple sub-values, or next state depends on previous state in complex ways). Inspired by Redux's reducer pattern.

```jsx
import { useReducer } from 'react';

const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    case 'reset': return initialState;
    default: throw new Error('Unknown action: ' + action.type);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

### Data Flow Diagram
```
   Component
      │  dispatch({ type: 'increment' })
      ▼
   Reducer Function (state, action) => newState
      │
      ▼
   New State ──> Component re-renders
```

### `useState` vs `useReducer`
| Use `useState` when | Use `useReducer` when |
|---|---|
| Simple, independent values | Multiple related values that update together |
| Few update variations | Many action types / complex transitions |
| No need to test logic separately | Want reducer as a pure, testable function |

---

## 16. `useMemo` & `useCallback`

Both are **memoization** hooks to avoid unnecessary recalculation/re-creation on every render — used for performance optimization.

### `useMemo` — memoize a **computed value**
```jsx
import { useMemo } from 'react';

function ExpensiveList({ items, filter }) {
  const filteredItems = useMemo(() => {
    console.log('Filtering...'); // only logs when items/filter change
    return items.filter(item => item.category === filter);
  }, [items, filter]);

  return <ul>{filteredItems.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

### `useCallback` — memoize a **function reference**
```jsx
import { useCallback } from 'react';

function Parent() {
  const [count, setCount] = useState(0);

  // Without useCallback, a NEW function is created every render,
  // causing React.memo(Child) to re-render unnecessarily.
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // stable reference across renders

  return <Child onClick={handleClick} />;
}

const Child = React.memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click</button>;
});
```

### When to Use (and When NOT To)
```
Rule of thumb:
- useMemo   → expensive calculations (sorting, filtering large lists, heavy math)
- useCallback → passing callbacks to memoized child components (React.memo)
- DON'T wrap everything — memoization itself has a cost (memory + comparison).
  Only optimize after profiling shows a real problem.
```

---

## 17. Custom Hooks

A **custom hook** is a JS function starting with `use` that calls other hooks — used to extract and reuse stateful logic.

```jsx
// hooks/useFetch.js
import { useState, useEffect } from 'react';

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let ignore = false;
    setLoading(true);
    fetch(url)
      .then(res => res.json())
      .then(data => { if (!ignore) { setData(data); setLoading(false); } })
      .catch(err => { if (!ignore) { setError(err); setLoading(false); } });
    return () => { ignore = true; };
  }, [url]);

  return { data, loading, error };
}

export default useFetch;
```

```jsx
// Usage in any component
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <h1>{user.name}</h1>;
}
```

### Rules of Hooks (applies to ALL hooks, custom or built-in)
1. Only call hooks at the **top level** (never inside loops, conditions, or nested functions).
2. Only call hooks from **React function components** or other **custom hooks**.

```
❌ WRONG:
if (isLoggedIn) {
  const [name, setName] = useState('');  // conditional hook call — breaks React
}

✅ CORRECT:
const [name, setName] = useState('');
if (isLoggedIn) {
  // use `name` here
}
```

---

## 18. React Router

`react-router-dom` (v6+) handles client-side routing (Single Page App navigation without full page reloads).

```bash
npm install react-router-dom
```

```jsx
import { BrowserRouter, Routes, Route, Link, useNavigate, useParams } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/users/:id" element={<UserProfile />} />
        <Route path="*" element={<NotFound />} /> {/* 404 catch-all */}
      </Routes>
    </BrowserRouter>
  );
}

function UserProfile() {
  const { id } = useParams();           // read URL param
  const navigate = useNavigate();        // programmatic navigation
  return (
    <div>
      <p>User ID: {id}</p>
      <button onClick={() => navigate('/')}>Go Home</button>
    </div>
  );
}
```

### Routing Diagram
```
 URL: /users/42
        │
        ▼
  <BrowserRouter> reads current URL
        │
        ▼
  <Routes> matches path pattern "/users/:id"
        │
        ▼
  Renders <UserProfile /> with useParams() → { id: "42" }
```

### Nested Routes & Layouts
```jsx
<Routes>
  <Route path="/" element={<Layout />}>       {/* parent layout */}
    <Route index element={<Home />} />
    <Route path="settings" element={<Settings />} />
  </Route>
</Routes>

// Layout.jsx uses <Outlet /> to render matched child route
import { Outlet } from 'react-router-dom';
function Layout() {
  return (
    <div>
      <Header />
      <Outlet /> {/* child route renders here */}
    </div>
  );
}
```

---

## 19. State Management (Context vs Redux Toolkit)

### When Local State / Context Is Enough
- Small-medium apps
- State doesn't change extremely frequently
- Not many deeply nested consumers with different update needs

### Redux Toolkit (RTK) — for larger apps
```bash
npm install @reduxjs/toolkit react-redux
```

```jsx
// store/counterSlice.js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },  // "mutating" syntax OK (uses Immer internally)
    decrement: (state) => { state.value -= 1; },
    addAmount: (state, action) => { state.value += action.payload; },
  },
});

export const { increment, decrement, addAmount } = counterSlice.actions;
export default counterSlice.reducer;
```

```jsx
// store/index.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';

export const store = configureStore({
  reducer: { counter: counterReducer },
});
```

```jsx
// main.jsx
import { Provider } from 'react-redux';
import { store } from './store';

<Provider store={store}>
  <App />
</Provider>
```

```jsx
// Any component
import { useSelector, useDispatch } from 'react-redux';
import { increment } from './store/counterSlice';

function Counter() {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();
  return <button onClick={() => dispatch(increment())}>{count}</button>;
}
```

### Architecture Comparison
```
Context API:                          Redux Toolkit:
App state in Provider                 Centralized "store"
   │                                       │
useContext() in any                   useSelector() reads
consumer component                    useDispatch() writes via actions
                                            │
                                       Reducers (pure functions)
                                       process actions → new state
```

| | Context API | Redux Toolkit |
|---|---|---|
| Setup complexity | Low | Medium |
| Dev tools / time-travel debugging | ❌ No | ✅ Yes |
| Best for | Theming, auth, simple global state | Large apps, complex/frequent state changes |
| Performance at scale | Can cause extra re-renders | Optimized with selectors |

---

## 20. Performance Optimization

### 1. `React.memo` — skip re-render if props unchanged
```jsx
const ExpensiveChild = React.memo(function ExpensiveChild({ value }) {
  console.log('Rendering ExpensiveChild');
  return <div>{value}</div>;
});
```

### 2. Code Splitting with `lazy` + `Suspense`
```jsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Dashboard />
    </Suspense>
  );
}
```

### 3. Virtualization for Long Lists
Only render items visible in the viewport (libraries: `react-window`, `react-virtualized`) — instead of rendering 10,000 DOM nodes at once.

```
Without virtualization:  [render all 10,000 rows] → slow, huge DOM
With virtualization:     [render only ~20 visible rows] → fast, small DOM
                          (rows recycled as user scrolls)
```

### 4. Avoid Unnecessary Re-renders — Common Causes
| Cause | Fix |
|---|---|
| New object/array/function created inline every render | `useMemo` / `useCallback` |
| Context value object recreated every render | Memoize the context value |
| Large component tree re-rendering from top state change | Move state closer to where it's used |
| Anonymous functions as props to memoized children | `useCallback` |

### 5. Profiling
Use **React DevTools Profiler** to record renders and identify which components re-render and why, before optimizing blindly.

---

## 21. Error Boundaries

Catch JavaScript errors in child component trees and show a fallback UI instead of crashing the whole app. (Error boundaries still require a **class component** — this is the one place classes remain necessary in modern React, since there's no Hook equivalent yet.)

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    console.error('Caught error:', error, info);
  }

  render() {
    if (this.state.hasError) {
      return <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>
```
> Note: Error boundaries do **not** catch errors in event handlers, async code, or SSR — use regular `try/catch` there.

---

## 22. React 18 Features

### 1. Automatic Batching
Before React 18, only React event handlers batched multiple `setState` calls into one render. React 18 batches them **everywhere** (promises, timeouts, native event handlers).

```jsx
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React 18: ONE re-render (batched)
  // React 17: TWO re-renders
}, 1000);
```

### 2. `createRoot` (Concurrent Rendering enabled)
```jsx
// React 18
import { createRoot } from 'react-dom/client';
createRoot(document.getElementById('root')).render(<App />);

// React 17 (old)
// ReactDOM.render(<App />, document.getElementById('root'));
```

### 3. `useTransition` — mark updates as non-urgent
```jsx
import { useTransition, useState } from 'react';

function SearchPage() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e) => {
    setQuery(e.target.value); // urgent — updates input instantly

    startTransition(() => {
      setResults(computeSearchResults(e.target.value)); // non-urgent, can be interrupted
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}
```

### 4. `Suspense` for Data Fetching (with compatible libraries)
Lets components "wait" for something (code or data) before rendering, showing a fallback meanwhile.

### 5. Strict Mode double-invoking
In development, React 18's `<StrictMode>` intentionally double-invokes component functions and effects to help surface bugs from impure code (side effects that aren't properly cleaned up).

---

## 23. Component Design Patterns

### 1. Container / Presentational Pattern
```
Container Component (logic, data fetching, state)
        │  passes data as props
        ▼
Presentational Component (pure UI, no state/logic)
```

### 2. Compound Components
```jsx
function Tabs({ children }) {
  const [active, setActive] = useState(0);
  return React.Children.map(children, (child, i) =>
    React.cloneElement(child, { isActive: i === active, onSelect: () => setActive(i) })
  );
}
// Usage: <Tabs><Tab>One</Tab><Tab>Two</Tab></Tabs>
```

### 3. Render Props
```jsx
function MouseTracker({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={e => setPos({ x: e.clientX, y: e.clientY })}>
      {render(pos)}
    </div>
  );
}
// Usage: <MouseTracker render={pos => <p>{pos.x}, {pos.y}</p>} />
```
> Custom Hooks have mostly replaced Render Props and Higher-Order Components as the preferred reuse pattern in modern React.

### 4. Higher-Order Components (HOC) — legacy but still seen
```jsx
function withLoading(Component) {
  return function WrappedComponent({ isLoading, ...props }) {
    if (isLoading) return <Spinner />;
    return <Component {...props} />;
  };
}
const UserListWithLoading = withLoading(UserList);
```

---

## 24. Testing React Apps

Using **React Testing Library** (RTL) + **Jest** or **Vitest** — philosophy: test components the way a user interacts with them, not implementation details.

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom
```

```jsx
// Counter.jsx
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

```jsx
// Counter.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from './Counter';

test('increments count when button clicked', () => {
  render(<Counter />);

  expect(screen.getByText('Count: 0')).toBeInTheDocument();

  fireEvent.click(screen.getByText('Increment'));

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

### Testing Pyramid for React Apps
```
        ▲
       /E2E\        (Cypress/Playwright) — few, full user flows
      /------\
     /Integr. \      (RTL) — component + interactions — MOST tests here
    /----------\
   /  Unit Tests \   (Jest/Vitest) — pure functions, hooks logic
  /----------------\
```

---

## 25. Common Interview Questions

1. **What is the Virtual DOM and why does it improve performance?**
   → It's an in-memory tree React diffs against the previous version, updating only changed real DOM nodes instead of re-rendering everything.

2. **Difference between state and props?**
   → Props: passed from parent, read-only. State: owned by the component, mutable via setters, triggers re-render.

3. **Why do list items need a `key`?**
   → Helps React match items between renders for correct updates (see Section 10).

4. **What's the difference between `useEffect` and `useLayoutEffect`?**
   → `useEffect` runs asynchronously after the browser paints. `useLayoutEffect` runs synchronously **before** paint — used when you need to measure/mutate the DOM before the user sees a flicker.

5. **What causes unnecessary re-renders and how do you prevent them?**
   → New object/function references each render, unmemoized context values, state too high in the tree. Fix with `React.memo`, `useMemo`, `useCallback`, or restructuring state.

6. **Controlled vs uncontrolled components?**
   → See Section 11 table.

7. **What are Hooks rules and why do they exist?**
   → Must be called at top level, unconditionally, so React can correctly match hook calls to internal state slots across renders (see Section 17).

8. **What problem does `useReducer`/Redux solve that `useState` doesn't?**
   → Predictable, centralized handling of complex or interrelated state transitions, easier debugging via pure reducer functions.

---

## 26. Cheat Sheet Summary

| Hook | Purpose |
|---|---|
| `useState` | Local component state |
| `useEffect` | Side effects (fetch, subscriptions, timers) |
| `useRef` | Mutable value / DOM reference, no re-render |
| `useContext` | Consume shared/global data without prop drilling |
| `useReducer` | Complex state logic via actions/reducer |
| `useMemo` | Memoize an expensive **computed value** |
| `useCallback` | Memoize a **function reference** |
| `useTransition` | Mark a state update as low priority (React 18) |
| Custom Hook (`useXxx`) | Extract & reuse stateful logic |

### Golden Rules
```
1. State is immutable  → always create new objects/arrays when updating.
2. Data flows down (props), events flow up (callbacks).
3. Keys must be stable & unique for list items.
4. Hooks: top-level only, same order every render.
5. Don't optimize (memo/useMemo/useCallback) before profiling shows a real problem.
6. Keep components small & focused — compose, don't bloat.
```

---

### Suggested Learning Path
```
JSX → Components → Props → State → Events
   → Conditional Rendering → Lists/Keys → Forms
   → useEffect → useRef → useContext → useReducer
   → Custom Hooks → React Router → State Management
   → Performance → Error Boundaries → React 18 → Testing
```

*End of Notes — happy learning! Build small projects (Todo App, Weather App, E-commerce cart) after each section to cement the concepts.*
