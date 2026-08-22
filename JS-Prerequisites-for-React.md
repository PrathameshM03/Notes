# JavaScript Prerequisites for React (Complete Notes)
### Everything you need before starting React — in one file

---

## Why This File Exists
React is "just JavaScript" with a UI layer on top. If these JS concepts feel shaky, React's syntax (hooks, JSX expressions, state updates) will feel confusing for the wrong reason — not because React is hard, but because the underlying JS is unfamiliar. Learn this first, then the React notes will click much faster.

## Table of Contents
1. [`let` / `const` and Scope](#1-let--const-and-scope)
2. [Arrow Functions](#2-arrow-functions)
3. [Template Literals](#3-template-literals)
4. [Destructuring](#4-destructuring)
5. [Spread & Rest Operators](#5-spread--rest-operators)
6. [Array Methods (map, filter, reduce, find)](#6-array-methods-map-filter-reduce-find)
7. [Objects & Object Shorthand](#7-objects--object-shorthand)
8. [Modules — import/export](#8-modules--importexport)
9. [Promises & Async/Await](#9-promises--asyncawait)
10. [`fetch` API](#10-fetch-api)
11. [Ternary & Logical Operators](#11-ternary--logical-operators)
12. [Optional Chaining & Nullish Coalescing](#12-optional-chaining--nullish-coalescing)
13. [`this`, Closures & Callbacks (just enough)](#13-this-closures--callbacks-just-enough)
14. [Classes (just enough to recognize them)](#14-classes-just-enough-to-recognize-them)
15. [Checklist: What Maps to What in React](#15-checklist-what-maps-to-what-in-react)

---

## 1. `let` / `const` and Scope

Stop using `var`. Use `const` by default, `let` only when you need to reassign.

```js
const name = "Prathamesh";   // cannot be reassigned
let count = 0;                 // can be reassigned
count = count + 1;             // OK

name = "New Name";             // ❌ Error: Assignment to constant variable
```

> Note: `const` prevents **reassignment**, not mutation. You can still change contents of an object/array declared with `const`:
```js
const user = { name: "A" };
user.name = "B"; // ✅ allowed — object itself wasn't reassigned, just its property
```

### Block Scope
```js
if (true) {
  let x = 10;
}
console.log(x); // ❌ ReferenceError — x only exists inside the block
```
**Why it matters for React:** every variable inside a component function follows these scoping rules — including hooks like `useState` results.

---

## 2. Arrow Functions

React code is full of arrow functions — event handlers, `.map()` callbacks, component definitions.

```js
// Regular function
function add(a, b) {
  return a + b;
}

// Arrow function equivalent
const add = (a, b) => {
  return a + b;
};

// Implicit return (no braces = automatic return, common in React)
const add = (a, b) => a + b;

// Single parameter — parentheses optional
const square = x => x * x;

// No parameters
const sayHi = () => console.log("Hi");
```

### Arrow Functions Returning an Object (common gotcha)
```js
// ❌ Wrong — JS thinks { } is a function body, not an object
const makeUser = (name) => { name: name };

// ✅ Correct — wrap object in parentheses
const makeUser = (name) => ({ name: name });
```
**Why it matters for React:** event handlers (`onClick={() => doSomething()}`), and callbacks inside `useEffect`, `.map()`, `useCallback` are almost always arrow functions.

---

## 3. Template Literals

Backticks (`` ` ``) let you embed variables directly in strings — used constantly for dynamic `className`, URLs, messages.

```js
const name = "Prathamesh";
const age = 23;

// Old way
const msg1 = "Hello, " + name + ". You are " + age + " years old.";

// Template literal
const msg2 = `Hello, ${name}. You are ${age} years old.`;

// Multi-line strings
const html = `
  <div>
    <p>${name}</p>
  </div>
`;
```
**Why it matters for React:** building dynamic class names, API URLs (`` `/api/users/${id}` ``), and inline messages.

---

## 4. Destructuring

Extract values out of objects/arrays into variables. **This is everywhere in React** — props, `useState`, imports.

### Object Destructuring
```js
const user = { name: "Prathamesh", age: 23, city: "Pune" };

// Without destructuring
const name = user.name;
const age = user.age;

// With destructuring
const { name, age } = user;

// Renaming while destructuring
const { name: userName } = user;

// Default values
const { country = "India" } = user; // "country" doesn't exist on user → uses default
```

### Array Destructuring (this is literally how `useState` works!)
```js
const colors = ["red", "green", "blue"];
const [first, second] = colors;
console.log(first);  // "red"
console.log(second); // "green"

// This is exactly the pattern React's useState returns:
// const [count, setCount] = useState(0);
//        ↑ value  ↑ setter function
```

### Destructuring Function Parameters (used for props in React)
```js
function greet({ name, age }) {
  console.log(`${name} is ${age}`);
}
greet({ name: "Prathamesh", age: 23 });

// This is exactly how React components read props:
// function Greeting({ name, age }) { ... }
```

---

## 5. Spread & Rest Operators

The `...` syntax. **Critical for React** because state must never be mutated directly — you always create new copies.

### Spread — expand an array/object into individual items
```js
// Arrays
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5]; // [1, 2, 3, 4, 5] — new array, arr1 untouched

// Objects
const user = { name: "A", age: 20 };
const updatedUser = { ...user, age: 21 }; // { name: "A", age: 21 } — new object

// This is EXACTLY the pattern for updating React state safely:
// setUser(prev => ({ ...prev, age: 21 }));
```

### Rest — collect remaining items into one variable
```js
function sum(...numbers) { // gathers all args into an array
  return numbers.reduce((total, n) => total + n, 0);
}
sum(1, 2, 3, 4); // 10

const { name, ...otherProps } = { name: "A", age: 20, city: "Pune" };
// name = "A", otherProps = { age: 20, city: "Pune" }

// Common in React for passing through remaining props:
// function Button({ label, ...rest }) { return <button {...rest}>{label}</button>; }
```

---

## 6. Array Methods (map, filter, reduce, find)

React renders lists using `.map()`. You'll use these constantly.

```js
const numbers = [1, 2, 3, 4, 5];

// map — transform each item, returns a NEW array (same length)
const doubled = numbers.map(n => n * 2); // [2, 4, 6, 8, 10]

// filter — keep items that pass a test, returns a NEW (possibly shorter) array
const evens = numbers.filter(n => n % 2 === 0); // [2, 4]

// find — returns the FIRST matching item (or undefined)
const firstEven = numbers.find(n => n % 2 === 0); // 2

// reduce — combine all items into a single value
const total = numbers.reduce((sum, n) => sum + n, 0); // 15
```

### The Pattern You'll See Everywhere in React
```jsx
const todos = [
  { id: 1, text: "Learn JS", done: true },
  { id: 2, text: "Learn React", done: false },
];

// Rendering a list
todos.map(todo => <li key={todo.id}>{todo.text}</li>);

// Filtering before rendering
todos.filter(todo => !todo.done).map(todo => <li key={todo.id}>{todo.text}</li>);
```
> `.map()` and `.filter()` never mutate the original array — they return new ones. This matches React's "don't mutate state" rule perfectly.

---

## 7. Objects & Object Shorthand

```js
const name = "Prathamesh";
const age = 23;

// Long way
const user = { name: name, age: age };

// Shorthand (when key and variable name match)
const user = { name, age };

// Method shorthand
const obj = {
  greet() {          // instead of greet: function() {}
    console.log("Hi");
  }
};

// Computed property names (dynamic keys)
const key = "email";
const user2 = { [key]: "test@test.com" }; // { email: "test@test.com" }
```
**Why it matters for React:** form handling often uses computed keys:
```jsx
setForm(prev => ({ ...prev, [e.target.name]: e.target.value }));
```

---

## 8. Modules — import/export

React projects are split across many files; you must know how to share code between them.

```js
// mathUtils.js
export function add(a, b) { return a + b; }
export const PI = 3.14159;

export default function multiply(a, b) { return a * b; } // one default export per file

// app.js
import multiply, { add, PI } from './mathUtils.js';
//     ↑ default        ↑ named exports (must match names, in {})

console.log(add(2, 3));      // 5
console.log(multiply(2, 3)); // 6
```

**Why it matters for React:** every component file does this:
```jsx
import React, { useState, useEffect } from 'react';
export default function App() { ... }
```

---

## 9. Promises & Async/Await

Used for anything that takes time — API calls, timers. React's data fetching relies entirely on this.

### Promises (the underlying concept)
```js
const promise = new Promise((resolve, reject) => {
  setTimeout(() => resolve("Done!"), 1000);
});

promise.then(result => console.log(result)); // "Done!" after 1 second
```

### Async/Await (cleaner syntax for the same thing — use this)
```js
async function getData() {
  const result = await someAsyncFunction(); // pauses here until resolved
  console.log(result);
}
```

### Error Handling with try/catch
```js
async function loadUser() {
  try {
    const response = await fetch('/api/user');
    const data = await response.json();
    console.log(data);
  } catch (error) {
    console.error('Failed:', error);
  }
}
```
**Why it matters for React:** data fetching inside `useEffect`, form submissions, and any API call use this pattern.

---

## 10. `fetch` API

The built-in way to make HTTP requests (GET, POST, etc.) — how React apps talk to a backend.

```js
// GET request
fetch('https://api.example.com/users')
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error(error));

// Same thing with async/await (cleaner, used more in React)
async function getUsers() {
  const response = await fetch('https://api.example.com/users');
  const data = await response.json();
  return data;
}

// POST request (sending data)
async function createUser(user) {
  const response = await fetch('https://api.example.com/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(user),
  });
  return response.json();
}
```
**Why it matters for React:** this is exactly what goes inside a `useEffect` for data fetching (see React notes, Section 12).

---

## 11. Ternary & Logical Operators

React uses these constantly for inline conditional rendering.

```js
// Ternary: condition ? valueIfTrue : valueIfFalse
const status = age >= 18 ? "Adult" : "Minor";

// Logical AND (&&) — short-circuit: right side only evaluates if left is truthy
const canVote = age >= 18 && "Eligible to vote";

// Logical OR (||) — returns first truthy value (used for defaults, but see nullish below)
const displayName = user.name || "Guest";
```
**Why it matters for React:** exactly what React notes Section 9 (Conditional Rendering) relies on — `{isLoggedIn ? <A/> : <B/>}` and `{condition && <A/>}`.

---

## 12. Optional Chaining & Nullish Coalescing

Avoid crashes when accessing deeply nested or possibly-missing data (very common with API responses in React).

```js
const user = { profile: { name: "A" } };

// Without optional chaining — crashes if profile is undefined
// const city = user.profile.address.city; // ❌ TypeError if address is missing

// With optional chaining (?.) — returns undefined instead of crashing
const city = user.profile?.address?.city; // undefined, no crash

// Nullish coalescing (??) — default ONLY for null/undefined (not for 0, "", false)
const count = user.count ?? 0; // uses 0 only if user.count is null or undefined

// Difference from || :
const value1 = 0 || 10;  // 10 (0 is falsy, so || replaces it — often WRONG)
const value2 = 0 ?? 10;  // 0  (0 is not null/undefined, so ?? keeps it — usually RIGHT)
```
**Why it matters for React:** safely rendering data that might not have loaded yet (`user?.name`), and setting real default values without accidentally overriding valid `0` or `""` values.

---

## 13. `this`, Closures & Callbacks (just enough)

You don't need deep mastery, but understand these ideas:

### Closures — a function "remembers" variables from where it was created
```js
function makeCounter() {
  let count = 0;
  return function () {
    count += 1;
    return count;
  };
}

const counter = makeCounter();
counter(); // 1
counter(); // 2 — remembers `count` between calls
```
**Why it matters for React:** this is the mental model behind how `useState` and closures inside `useEffect` "remember" values from a specific render — a common source of confusion later ("stale closure" bugs).

### Callbacks — passing a function to run later
```js
button.addEventListener('click', function () {
  console.log('Clicked!');
});
```
**Why it matters for React:** `onClick={handleClick}`, `.map(item => ...)`, `useEffect(() => {...}, [])` — all are callbacks.

> `this` keyword: modern React (function components + hooks) barely uses `this` at all, unlike old class components. You can safely learn it later — don't block on it.

---

## 14. Classes (just enough to recognize them)

You chose Hooks-only React, so you won't write class components — but you may see this syntax in older tutorials/docs, and Error Boundaries (React notes, Section 21) still require a class.

```js
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  greet() {
    console.log(`Hi, I'm ${this.name}`);
  }
}

const p = new Person("Prathamesh", 23);
p.greet();

// Class inheritance (React class components extend React.Component)
class Student extends Person {
  constructor(name, age, course) {
    super(name, age); // calls Person's constructor
    this.course = course;
  }
}
```
**You just need to recognize this pattern**, not be fluent writing new class components.

---

## 15. Checklist: What Maps to What in React

| JS Concept | Where You'll See It in React |
|---|---|
| Arrow functions | Every event handler, every `.map()`, every component |
| Destructuring | Reading props: `function Card({ title, price })` |
| Array destructuring | `const [count, setCount] = useState(0)` |
| Spread operator | Updating state immutably: `{ ...prevState, key: value }` |
| `.map()` | Rendering lists: `items.map(item => <li key={item.id}>...</li>)` |
| `.filter()` | Conditional lists, search/filter UIs |
| Template literals | Dynamic class names, URLs: `` `/api/users/${id}` `` |
| Ternary / `&&` | Conditional rendering: `{loggedIn ? <A/> : <B/>}` |
| Optional chaining | Safe access to API data: `user?.address?.city` |
| Promises / async-await | Data fetching inside `useEffect` |
| `fetch` | Talking to a backend API |
| import/export | Every single component file |
| Closures | Understanding why `useState`/`useEffect` behave the way they do |

---

## Suggested Order to Learn Before React

```
1. let/const, scope
2. Arrow functions
3. Template literals
4. Destructuring (object + array)
5. Spread/rest operators
6. Array methods: map, filter, find, reduce
7. Ternary + && conditional expressions
8. Optional chaining (?.) and nullish coalescing (??)
9. import/export modules
10. Promises → async/await → fetch
11. (light) closures — just enough to not be confused later
```

Once these feel natural — you write them without pausing to think — you're ready to start the **ReactJS Complete Notes** file. Nothing else in core JS is strictly required to begin (classes, generators, `this` binding tricks, regex, etc. can all wait).

*Tip: Practice each concept in your browser console or a quick `.js` file before moving to React — don't try to learn JS and React syntax at the same time, it doubles the confusion.*
