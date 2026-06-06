# React — Complete Fundamentals Guide

> React is a JavaScript library for building user interfaces. It uses a component-based architecture and a virtual DOM. Angular developers will find many similar concepts but different implementation.

---

## Table of Contents

1. [What is React?](#1-what-is-react)
2. [JSX](#2-jsx)
3. [Components](#3-components)
4. [Props](#4-props)
5. [State (useState)](#5-state-usestate)
6. [Event Handling](#6-event-handling)
7. [Conditional Rendering](#7-conditional-rendering)
8. [Lists & Keys](#8-lists--keys)
9. [Forms](#9-forms)
10. [useEffect Hook](#10-useeffect-hook)
11. [useRef Hook](#11-useref-hook)
12. [useMemo & useCallback](#12-usememo--usecallback)
13. [Custom Hooks](#13-custom-hooks)
14. [Context API](#14-context-api)
15. [useReducer](#15-usereducer)
16. [Component Patterns](#16-component-patterns)
17. [React Router](#17-react-router)
18. [State Management (Overview)](#18-state-management-overview)
19. [Performance Optimization](#19-performance-optimization)
20. [Miscellaneous Concepts](#20-miscellaneous-concepts)
21. [Interview Q&A](#21-interview-qa)

---

## 1. What is React?

- A **JavaScript library** (not a framework) for building UIs.
- Developed by **Facebook/Meta**.
- Uses a **component-based architecture** — UI is split into reusable components.
- Uses a **Virtual DOM** — React maintains a virtual representation of the DOM, diffs it on state changes, and makes minimal real DOM updates.
- **Declarative** — describe what the UI should look like, React handles the how.

### Angular vs React (for context)

| Feature | React | Angular |
|---------|-------|---------|
| Type | Library | Full framework |
| Language | JavaScript/JSX | TypeScript |
| Data Binding | One-way | Two-way (NgModel) |
| Opinionation | Low | High |
| State Management | External libs | Services/RxJS |
| Router | React Router (external) | Built-in |
| Forms | Controlled/Uncontrolled | Template/Reactive |

### Virtual DOM
When state changes, React:
1. Creates a new Virtual DOM tree
2. Diffs it against the previous Virtual DOM (reconciliation)
3. Computes the minimum number of real DOM operations needed
4. Applies those changes to the real DOM (commit phase)

---

## 2. JSX

JSX is syntax sugar for `React.createElement()`. It lets you write HTML-like code in JavaScript.

```jsx
// JSX
const element = <h1 className="title">Hello, {name}!</h1>;

// What JSX compiles to (under the hood)
const element = React.createElement("h1", { className: "title" }, `Hello, ${name}!`);
```

### JSX Rules

```jsx
// 1. Must return a single root element (or Fragment)
function App() {
  return (
    <div>         {/* single root */}
      <h1>Title</h1>
      <p>Paragraph</p>
    </div>
  );
}

// Or use Fragment (no extra DOM node)
function App() {
  return (
    <>
      <h1>Title</h1>
      <p>Paragraph</p>
    </>
  );
}

// 2. Use className instead of class, htmlFor instead of for
<div className="container">...</div>
<label htmlFor="name">Name:</label>

// 3. Self-close empty elements
<img src="url" alt="desc" />
<input type="text" />
<br />

// 4. Expressions in curly braces
const name = "Alice";
const age = 30;
const isLoggedIn = true;
<h1>Hello, {name}!</h1>
<p>Age: {age * 2}</p>
<p>{isLoggedIn ? "Welcome back!" : "Please log in"}</p>
<p>{isLoggedIn && "You have 3 messages"}</p>

// 5. camelCase for event handlers and CSS properties
<button onClick={handleClick}>Click me</button>
<div style={{ backgroundColor: "red", fontSize: "16px" }}>...</div>

// 6. Comments
{/* This is a JSX comment */}
```

---

## 3. Components

### Functional Components (modern — preferred)
```jsx
// Simple functional component
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// Arrow function style
const Button = ({ label, onClick, disabled = false }) => (
  <button onClick={onClick} disabled={disabled}>
    {label}
  </button>
);

// Usage
function App() {
  return (
    <div>
      <Greeting name="Alice" />
      <Button label="Submit" onClick={() => console.log("clicked")} />
    </div>
  );
}
```

### Component Naming
- Must start with an **uppercase letter** — `<User />` is a component, `<user />` is an HTML element.

### Component Tree
```
App
├── Header
│   └── NavBar
├── Main
│   ├── UserList
│   │   └── UserCard (repeated)
│   └── Sidebar
└── Footer
```

### Pure Components
A component is "pure" if it always returns the same JSX for the same props.

---

## 4. Props

Props (properties) are inputs to a component — **read-only, immutable** (like function parameters).

```jsx
// Parent passes props
function UserCard({ user, onSelect }) {
  return (
    <div className="card" onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}

// Default props
function Button({ label = "Click me", variant = "primary", size = "md" }) {
  return <button className={`btn btn-${variant} btn-${size}`}>{label}</button>;
}

// Children prop
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
}

// Usage with children
<Card title="User Profile">
  <p>Name: Alice</p>
  <p>Email: alice@example.com</p>
</Card>

// Spread props
const buttonProps = { disabled: false, type: "submit" };
<button {...buttonProps}>Submit</button>

// Props with TypeScript
interface UserProps {
  name: string;
  age: number;
  email?: string;  // optional
  onSelect: (id: number) => void;
}

function User({ name, age, email, onSelect }: UserProps) {
  return <div>{name}</div>;
}
```

### Prop Drilling Problem
When props need to be passed through many levels, use Context API (see section 14).

---

## 5. State (useState)

State is **private, mutable data** owned by a component. When state changes, React re-renders the component.

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);   // [value, setter]
  const [name, setName] = useState("");
  const [user, setUser] = useState(null);
  const [items, setItems] = useState([]);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(prev => prev - 1)}>Decrement</button>
      {/* Using functional update when new state depends on old state */}
    </div>
  );
}
```

### State Update Rules

```jsx
// ─── NEVER mutate state directly ───
const [items, setItems] = useState([1, 2, 3]);

// WRONG
items.push(4);       // mutation — React won't detect change
setItems(items);     // still won't re-render correctly

// CORRECT — create new array
setItems([...items, 4]);
setItems(prev => [...prev, 4]);    // functional update form

// ─── Object state — always spread ───
const [user, setUser] = useState({ name: "Alice", age: 30 });

// WRONG
user.age = 31; setUser(user);  // mutation

// CORRECT
setUser({ ...user, age: 31 });
setUser(prev => ({ ...prev, age: 31 }));

// ─── State updates are batched (React 18) ───
// Multiple setState calls in one event handler are batched into one re-render
function handleClick() {
  setCount(c => c + 1);  // batched
  setName("Bob");        // batched — only ONE re-render happens
}

// ─── Lazy initialization (expensive initial state) ───
const [data, setData] = useState(() => computeExpensiveInitialState());
```

---

## 6. Event Handling

```jsx
function Form() {
  const [value, setValue] = useState("");

  // Event handler as inline arrow
  const handleChange = (event) => {
    setValue(event.target.value);
  };

  // Prevent default browser behavior
  const handleSubmit = (event) => {
    event.preventDefault();
    console.log("Submitted:", value);
  };

  // Passing extra arguments
  const handleItemClick = (id, event) => {
    event.stopPropagation(); // stop event bubbling
    console.log("Item clicked:", id);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={value}
        onChange={handleChange}
        onFocus={() => console.log("focused")}
        onBlur={() => console.log("blurred")}
        onKeyDown={(e) => e.key === "Enter" && handleSubmit(e)}
      />
      <button type="submit">Submit</button>
      <button onClick={(e) => { e.stopPropagation(); handleClick(); }}>
        Click
      </button>
    </form>
  );
}
```

### Synthetic Events
React wraps native DOM events in **SyntheticEvent** — cross-browser compatible wrapper.
- Same API as native events
- Pooled for performance in older React versions (not pooled in React 17+)

---

## 7. Conditional Rendering

```jsx
function Notification({ type, message, isVisible }) {
  // 1. if-else (for complex conditions)
  if (!isVisible) return null; // render nothing

  // 2. Ternary
  return (
    <div>
      {type === "success" ? (
        <p className="success">✓ {message}</p>
      ) : (
        <p className="error">✗ {message}</p>
      )}

      {/* 3. Short-circuit (&&) — renders right side only if left is truthy */}
      {message && <p>{message}</p>}

      {/* Pitfall: 0 renders as "0" — use !! or null check */}
      {items.length > 0 && <List items={items} />}   // correct
      {items.length && <List items={items} />}        // WRONG — renders 0 when empty

      {/* 4. Nullish coalescing */}
      {user?.name ?? "Guest"}
    </div>
  );
}

// Conditional component rendering
function App({ isLoggedIn }) {
  if (isLoggedIn) return <Dashboard />;
  return <Login />;
}
```

---

## 8. Lists & Keys

```jsx
function UserList({ users }) {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>  {/* key must be unique among siblings */}
          <UserCard user={user} />
        </li>
      ))}
    </ul>
  );
}
```

### Keys — Important Rules
- Keys help React identify which items changed, added, or removed.
- Must be **unique among siblings** (not globally).
- Should be **stable** — don't use array index as key if list can reorder/filter.
- Keys are NOT passed as props to the component.

```jsx
// BAD — index as key (causes bugs when reordering/filtering)
{items.map((item, index) => <Item key={index} item={item} />)}

// GOOD — stable unique ID
{items.map((item) => <Item key={item.id} item={item} />)}

// If no ID exists, generate one on data creation, not in render
```

---

## 9. Forms

### Controlled Components (React manages input value)
```jsx
function LoginForm() {
  const [formData, setFormData] = useState({ email: "", password: "" });
  const [errors, setErrors] = useState({});

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
    // Clear error when user types
    if (errors[name]) setErrors(prev => ({ ...prev, [name]: "" }));
  };

  const validate = () => {
    const newErrors = {};
    if (!formData.email) newErrors.email = "Email is required";
    else if (!/\S+@\S+\.\S+/.test(formData.email)) newErrors.email = "Invalid email";
    if (!formData.password || formData.password.length < 6)
      newErrors.password = "Password must be at least 6 characters";
    return newErrors;
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const validationErrors = validate();
    if (Object.keys(validationErrors).length > 0) {
      setErrors(validationErrors);
      return;
    }
    console.log("Submit:", formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input type="email" name="email" value={formData.email} onChange={handleChange} />
        {errors.email && <span className="error">{errors.email}</span>}
      </div>
      <div>
        <input type="password" name="password" value={formData.password} onChange={handleChange} />
        {errors.password && <span className="error">{errors.password}</span>}
      </div>
      <button type="submit">Login</button>
    </form>
  );
}

// Checkbox & Select
const [agreed, setAgreed] = useState(false);
const [role, setRole] = useState("user");

<input type="checkbox" checked={agreed} onChange={(e) => setAgreed(e.target.checked)} />
<select value={role} onChange={(e) => setRole(e.target.value)}>
  <option value="user">User</option>
  <option value="admin">Admin</option>
</select>
```

### Uncontrolled Components (DOM manages value)
```jsx
import { useRef } from "react";

function UncontrolledForm() {
  const inputRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log(inputRef.current.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} type="text" defaultValue="initial" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## 10. useEffect Hook

Handles **side effects** — data fetching, subscriptions, DOM manipulation, timers.

```jsx
import { useState, useEffect } from "react";

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // ─── Runs after every render ───
  useEffect(() => {
    document.title = `User ${userId}`;
  }); // no dependency array

  // ─── Runs once on mount (componentDidMount equivalent) ───
  useEffect(() => {
    console.log("Component mounted");
    return () => console.log("Component unmounted"); // cleanup
  }, []); // empty array

  // ─── Runs when userId changes (componentDidUpdate equivalent) ───
  useEffect(() => {
    let cancelled = false; // prevent state update on unmounted component

    const fetchUser = async () => {
      try {
        setLoading(true);
        setError(null);
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) throw new Error("Failed to fetch");
        const data = await response.json();
        if (!cancelled) setUser(data);
      } catch (err) {
        if (!cancelled) setError(err.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetchUser();

    // Cleanup — runs before next effect or on unmount
    return () => { cancelled = true; };
  }, [userId]); // re-run when userId changes

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;
  return <div>{user.name}</div>;
}
```

### useEffect Common Patterns
```jsx
// ─── Event listener ───
useEffect(() => {
  const handleResize = () => setWidth(window.innerWidth);
  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize); // cleanup!
}, []);

// ─── Timer ───
useEffect(() => {
  const timer = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(timer); // cleanup!
}, []);

// ─── WebSocket ───
useEffect(() => {
  const ws = new WebSocket("ws://localhost:8080");
  ws.onmessage = (e) => setMessages(prev => [...prev, e.data]);
  return () => ws.close(); // cleanup!
}, []);
```

---

## 11. useRef Hook

`useRef` returns a mutable object `{ current: value }` that persists across renders **without causing re-renders**.

```jsx
import { useRef, useEffect } from "react";

function AutoFocusInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus(); // access DOM element
  }, []);

  return <input ref={inputRef} type="text" />;
}

function Stopwatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef(null); // store interval ID without triggering re-render

  const start = () => {
    intervalRef.current = setInterval(() => setTime(t => t + 1), 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
  };

  return (
    <div>
      <p>{time}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}

// Store previous value
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; });
  return ref.current; // returns previous render's value
}

// forwardRef — expose ref to parent
const Input = React.forwardRef((props, ref) => (
  <input ref={ref} {...props} />
));
```

---

## 12. useMemo & useCallback

### useMemo — Memoize computed values
```jsx
import { useMemo } from "react";

function ProductList({ products, searchTerm, sortBy }) {
  // Only recomputes when products, searchTerm, or sortBy changes
  const filteredProducts = useMemo(() => {
    console.log("Computing filtered products"); // expensive operation
    return products
      .filter(p => p.name.toLowerCase().includes(searchTerm.toLowerCase()))
      .sort((a, b) => {
        if (sortBy === "price") return a.price - b.price;
        return a.name.localeCompare(b.name);
      });
  }, [products, searchTerm, sortBy]);

  return (
    <ul>
      {filteredProducts.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

### useCallback — Memoize function reference
```jsx
import { useCallback } from "react";

function Parent() {
  const [count, setCount] = useState(0);

  // Without useCallback: new function reference on every render → Child re-renders
  // With useCallback: same reference unless dependencies change → no unnecessary re-render
  const handleClick = useCallback((id) => {
    console.log("Clicked:", id);
    setCount(c => c + 1);
  }, []); // no dependencies — stable reference

  return <Child onClick={handleClick} />;
}

// Child must be wrapped in React.memo for useCallback to have effect
const Child = React.memo(({ onClick }) => {
  console.log("Child rendered");
  return <button onClick={() => onClick(1)}>Click</button>;
});
```

### When to use
- `useMemo`: expensive calculations; derived state that's used in render
- `useCallback`: functions passed as props to memoized child components
- **Don't over-optimize** — premature optimization adds complexity

---

## 13. Custom Hooks

Custom hooks are functions that start with `use` and can call other hooks. They extract and reuse stateful logic.

```jsx
// ─── useFetch — data fetching ───
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    const controller = new AbortController();

    const fetchData = async () => {
      try {
        setLoading(true);
        const res = await fetch(url, { signal: controller.signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const json = await res.json();
        if (!cancelled) setData(json);
      } catch (err) {
        if (err.name !== "AbortError" && !cancelled) setError(err.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetchData();
    return () => { cancelled = true; controller.abort(); };
  }, [url]);

  return { data, loading, error };
}

// Usage
function UserProfile({ id }) {
  const { data: user, loading, error } = useFetch(`/api/users/${id}`);
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return <div>{user?.name}</div>;
}

// ─── useLocalStorage ───
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback((newValue) => {
    try {
      const valueToStore = newValue instanceof Function ? newValue(value) : newValue;
      setValue(valueToStore);
      localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (err) {
      console.error(err);
    }
  }, [key, value]);

  return [value, setStoredValue];
}

// ─── useDebounce ───
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// ─── useToggle ───
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}
```

---

## 14. Context API

Avoid prop drilling by sharing state across the component tree.

```jsx
import { createContext, useContext, useState, useMemo } from "react";

// 1. Create context
const AuthContext = createContext(null);

// 2. Create provider component
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = useCallback(async (credentials) => {
    const response = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(credentials)
    });
    const data = await response.json();
    setUser(data.user);
    localStorage.setItem("token", data.token);
  }, []);

  const logout = useCallback(() => {
    setUser(null);
    localStorage.removeItem("token");
  }, []);

  // Memoize context value to prevent unnecessary re-renders
  const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// 3. Custom hook for consuming context
function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error("useAuth must be used within AuthProvider");
  return context;
}

// 4. Use in any component
function Profile() {
  const { user, logout } = useAuth();
  return (
    <div>
      <h1>Welcome, {user?.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}

// 5. Wrap app with provider
function App() {
  return (
    <AuthProvider>
      <Router>
        <Profile />
      </Router>
    </AuthProvider>
  );
}
```

---

## 15. useReducer

For complex state logic with multiple sub-values, or when next state depends on previous.

```jsx
import { useReducer } from "react";

// Action types (as constants)
const ACTIONS = {
  INCREMENT: "INCREMENT",
  DECREMENT: "DECREMENT",
  RESET: "RESET",
  SET: "SET"
};

// Reducer — pure function
function counterReducer(state, action) {
  switch (action.type) {
    case ACTIONS.INCREMENT:
      return { ...state, count: state.count + (action.payload ?? 1) };
    case ACTIONS.DECREMENT:
      return { ...state, count: state.count - 1 };
    case ACTIONS.RESET:
      return { count: 0 };
    case ACTIONS.SET:
      return { ...state, count: action.payload };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: ACTIONS.INCREMENT })}>+1</button>
      <button onClick={() => dispatch({ type: ACTIONS.INCREMENT, payload: 5 })}>+5</button>
      <button onClick={() => dispatch({ type: ACTIONS.DECREMENT })}>-</button>
      <button onClick={() => dispatch({ type: ACTIONS.RESET })}>Reset</button>
    </div>
  );
}
```

### useReducer + Context (Redux-like pattern)
```jsx
// Combine useReducer with Context for global state management
const StoreContext = createContext();

function StoreProvider({ children }) {
  const [state, dispatch] = useReducer(rootReducer, initialState);
  return (
    <StoreContext.Provider value={{ state, dispatch }}>
      {children}
    </StoreContext.Provider>
  );
}

const useStore = () => useContext(StoreContext);
```

---

## 16. Component Patterns

### Higher-Order Component (HOC)
A function that takes a component and returns an enhanced component.
```jsx
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const { user } = useAuth();
    if (!user) return <Navigate to="/login" />;
    return <WrappedComponent {...props} user={user} />;
  };
}

const ProtectedDashboard = withAuth(Dashboard);
```

### Render Props
```jsx
function DataFetcher({ url, render }) {
  const { data, loading, error } = useFetch(url);
  return render({ data, loading, error });
}

<DataFetcher
  url="/api/users"
  render={({ data, loading }) =>
    loading ? <Spinner /> : <UserList users={data} />
  }
/>
```

### Compound Components
```jsx
function Accordion({ children }) {
  const [openIndex, setOpenIndex] = useState(null);
  return (
    <AccordionContext.Provider value={{ openIndex, setOpenIndex }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

Accordion.Item = function AccordionItem({ index, title, children }) {
  const { openIndex, setOpenIndex } = useContext(AccordionContext);
  const isOpen = openIndex === index;
  return (
    <div>
      <button onClick={() => setOpenIndex(isOpen ? null : index)}>{title}</button>
      {isOpen && <div>{children}</div>}
    </div>
  );
};

// Usage
<Accordion>
  <Accordion.Item index={0} title="Section 1">Content 1</Accordion.Item>
  <Accordion.Item index={1} title="Section 2">Content 2</Accordion.Item>
</Accordion>
```

### React.memo
```jsx
// Prevents re-render if props haven't changed (shallow comparison)
const UserCard = React.memo(function UserCard({ user, onSelect }) {
  console.log("UserCard rendered for:", user.id);
  return <div onClick={() => onSelect(user.id)}>{user.name}</div>;
});

// Custom comparison
const UserCard = React.memo(UserCardComponent, (prevProps, nextProps) => {
  return prevProps.user.id === nextProps.user.id; // return true to skip re-render
});
```

---

## 17. React Router

```bash
npm install react-router-dom
```

```jsx
import { BrowserRouter, Routes, Route, Link, NavLink, Navigate,
         useNavigate, useParams, useLocation, useSearchParams, Outlet } from "react-router-dom";

// Setup
function App() {
  return (
    <BrowserRouter>
      <nav>
        <NavLink to="/" end className={({ isActive }) => isActive ? "active" : ""}>Home</NavLink>
        <NavLink to="/users">Users</NavLink>
        <Link to="/about">About</Link>
      </nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/users" element={<UsersLayout />}>
          <Route index element={<UserList />} />              {/* /users */}
          <Route path=":id" element={<UserDetail />} />       {/* /users/:id */}
          <Route path=":id/edit" element={<EditUser />} />    {/* /users/:id/edit */}
        </Route>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute><Dashboard /></ProtectedRoute>
        } />
        <Route path="*" element={<NotFound />} />
        <Route path="/old-path" element={<Navigate to="/new-path" replace />} />
      </Routes>
    </BrowserRouter>
  );
}

// Layout component uses Outlet
function UsersLayout() {
  return (
    <div>
      <h1>Users Section</h1>
      <Outlet />  {/* renders matched child route */}
    </div>
  );
}

// Route params
function UserDetail() {
  const { id } = useParams(); // /users/123 → { id: "123" }
  const navigate = useNavigate();
  const location = useLocation();
  const [searchParams, setSearchParams] = useSearchParams();

  const page = searchParams.get("page") || "1";

  return (
    <div>
      <p>User ID: {id}</p>
      <p>Current path: {location.pathname}</p>
      <p>Page: {page}</p>
      <button onClick={() => navigate(-1)}>Back</button>
      <button onClick={() => navigate("/users")}>All Users</button>
      <button onClick={() => navigate(`/users/${id}/edit`, { state: { from: "detail" } })}>
        Edit
      </button>
    </div>
  );
}

// Protected route
function ProtectedRoute({ children }) {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" state={{ from: location }} replace />;
  return children;
}
```

---

## 18. State Management (Overview)

### When to use what

| Scope | Solution |
|-------|----------|
| Local UI state | `useState` |
| Complex local state | `useReducer` |
| Shared state (few components) | Context API + useReducer |
| Global app state | Redux Toolkit / Zustand |
| Server/cache state | React Query / SWR |
| Form state | React Hook Form |

### Redux Toolkit (brief)
```bash
npm install @reduxjs/toolkit react-redux
```

```jsx
// store/userSlice.js
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

export const fetchUsers = createAsyncThunk("users/fetchAll", async () => {
  const res = await fetch("/api/users");
  return res.json();
});

const usersSlice = createSlice({
  name: "users",
  initialState: { items: [], loading: false, error: null },
  reducers: {
    addUser: (state, action) => { state.items.push(action.payload); },
    removeUser: (state, action) => {
      state.items = state.items.filter(u => u.id !== action.payload);
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => { state.loading = true; })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message;
      });
  }
});

// store/index.js
import { configureStore } from "@reduxjs/toolkit";
const store = configureStore({ reducer: { users: usersSlice.reducer } });

// Component
import { useSelector, useDispatch } from "react-redux";
function Users() {
  const { items, loading } = useSelector(state => state.users);
  const dispatch = useDispatch();
  useEffect(() => { dispatch(fetchUsers()); }, []);
  return loading ? <Spinner /> : <UserList users={items} />;
}
```

---

## 19. Performance Optimization

```jsx
// 1. React.memo — skip re-render if props unchanged
const Child = React.memo(({ value }) => <div>{value}</div>);

// 2. useCallback — stable function reference for memoized children
const handleClick = useCallback(() => doSomething(id), [id]);

// 3. useMemo — expensive computations
const sorted = useMemo(() => [...items].sort(...), [items]);

// 4. Code Splitting (lazy loading)
import { lazy, Suspense } from "react";
const Dashboard = lazy(() => import("./Dashboard"));
<Suspense fallback={<Spinner />}>
  <Dashboard />
</Suspense>

// 5. Virtualization for long lists
// npm install react-window
import { FixedSizeList } from "react-window";
<FixedSizeList height={500} itemCount={10000} itemSize={35} width="100%">
  {({ index, style }) => <div style={style}>Row {index}</div>}
</FixedSizeList>

// 6. Key on components to reset state
// Changing key forces remount (fresh state)
<ExpensiveComponent key={userId} userId={userId} />

// 7. Avoid creating objects/arrays inline in JSX (new reference each render)
// BAD — new object on every render
<Component style={{ color: "red" }} />
// GOOD — stable reference
const style = { color: "red" };
<Component style={style} />
```

---

## 20. Miscellaneous Concepts

### Error Boundaries
Class component that catches JS errors in child components.
```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error("Error caught:", error, errorInfo);
  }

  render() {
    if (this.state.hasError) return <h2>Something went wrong.</h2>;
    return this.props.children;
  }
}

<ErrorBoundary><App /></ErrorBoundary>
```

### Portals
Render children outside parent DOM hierarchy.
```jsx
import { createPortal } from "react-dom";

function Modal({ isOpen, children }) {
  if (!isOpen) return null;
  return createPortal(
    <div className="modal-overlay">
      <div className="modal">{children}</div>
    </div>,
    document.getElementById("modal-root") // render outside #app
  );
}
```

### Strict Mode
```jsx
<React.StrictMode>
  <App />
</React.StrictMode>
// Helps detect side effects by running effects twice in development
// Warns about deprecated APIs
```

### React.Fragment
```jsx
<React.Fragment key={item.id}>  {/* Fragment with key */}
  <dt>{item.term}</dt>
  <dd>{item.def}</dd>
</React.Fragment>
// Short syntax <></> can't have key
```

### useId Hook (React 18)
```jsx
function TextField({ label }) {
  const id = useId(); // generates unique ID, stable across server/client
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

### Transitions (React 18)
```jsx
import { startTransition, useTransition, useDeferredValue } from "react";

// Mark updates as non-urgent
const [isPending, startTransition] = useTransition();
startTransition(() => {
  setSearchQuery(value); // non-urgent — can be interrupted
});

// Defer a value
const deferredQuery = useDeferredValue(searchQuery);
// deferredQuery updates after urgent state is applied
```

---

## 21. Interview Q&A

**Q: What is the Virtual DOM and how does it work?**
> React maintains an in-memory representation of the real DOM. On state/props change, React creates a new virtual DOM tree, diffs it against the previous one (reconciliation with Fiber algorithm), and applies only the minimal necessary changes to the real DOM.

**Q: What is the difference between state and props?**
> Props are read-only inputs passed from parent to child. State is mutable data owned and managed by the component itself. Props cannot be changed by the receiving component; state can be changed via setters.

**Q: Explain the rules of Hooks.**
> 1. Only call hooks at the top level (not inside loops, conditions, or nested functions). 2. Only call hooks from React function components or custom hooks. This ensures hooks are called in the same order on every render.

**Q: What is the useEffect cleanup function?**
> The function returned from useEffect runs before the next effect execution (on dependency change) and on component unmount. Used to clean up subscriptions, timers, event listeners, or async operations to prevent memory leaks.

**Q: What is the difference between `useMemo` and `useCallback`?**
> `useMemo` memoizes the result of a computation. `useCallback` memoizes a function reference. `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

**Q: When would you use useReducer over useState?**
> When state logic is complex (multiple sub-values), when next state depends on previous in complex ways, when state updates have many cases, or when you want Redux-like architecture locally.

**Q: What is prop drilling and how do you avoid it?**
> Prop drilling is passing props through many intermediate components that don't use them. Solutions: Context API for global/shared state, component composition (children prop), or state management libraries.

**Q: What is React.memo?**
> A HOC that wraps a component to skip re-rendering if props haven't changed (shallow comparison). Only effective when parent re-renders frequently but child's props are stable.

**Q: How does React 18's concurrent mode help?**
> React 18 introduced concurrent features that allow React to interrupt, pause, resume, or abandon renders. `useTransition` marks updates as non-urgent so React can deprioritize them. `useDeferredValue` defers non-critical updates to keep UI responsive.

**Q: What is the key prop and why is it important?**
> Keys help React identify which items in a list have changed, been added, or removed. React uses keys during reconciliation to reuse existing DOM nodes instead of recreating them. Using unstable keys (like array index with reordering) causes bugs and performance issues.
