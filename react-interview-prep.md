# React Interview Preparation Guide

> Comprehensive interview-ready answers for experienced React developers.

---

## Table of Contents

1. [[#1 Virtual DOM & Reconciliation]]
2. [[#2 JSX]]
3. [[#3 Components — Class vs Function]]
4. [[#4 State and Props]]
5. [[#5 React Hooks (Core)]]
6. [[#6 Custom Hooks]]
7. [[#7 Component Lifecycle]]
8. [[#8 Context API]]
9. [[#9 State Management — Redux]]
10. [[#10 Performance Optimization]]
11. [[#11 React Fiber Architecture]]
12. [[#12 Concurrent Features]]
13. [[#13 Error Boundaries]]
14. [[#14 Controlled vs Uncontrolled Components]]
15. [[#15 Higher-Order Components, Render Props & Hooks]]
16. [[#16 Portals]]
17. [[#17 Server-Side Rendering (SSR) & Next.js]]
18. [[#18 React Router]]
19. [[#19 Testing]]
20. [[#20 TypeScript with React]]
21. [[#21 Common Coding Patterns]]

---

## 1 Virtual DOM & Reconciliation

### Q: What is the Virtual DOM and how does it work?

**Answer:**

The Virtual DOM (VDOM) is a **lightweight JavaScript representation** of the actual DOM. React uses it to minimize direct DOM manipulations, which are expensive.

**How it works:**
```
1. State/prop changes → React creates a NEW Virtual DOM tree
2. React DIFFS the new VDOM against the previous VDOM (Reconciliation)
3. React calculates the MINIMUM set of changes needed
4. React BATCHES and applies those changes to the REAL DOM
```

**Why it's fast:**
- JavaScript object operations are much cheaper than DOM operations.
- React batches multiple updates into a single DOM write.
- The diffing algorithm is O(n), not O(n³) — it uses heuristics.

### Q: What is Reconciliation? Explain the diffing algorithm.

**Answer:**

Reconciliation is React's process of comparing two VDOM trees to determine what changed. It uses two key heuristics:

**1. Elements of different types produce different trees:**
```jsx
// Old                     // New
<div><Counter /></div>  →  <span><Counter /></span>
// React destroys entire <div> subtree and rebuilds from scratch
```

**2. Keys identify which items changed in lists:**
```jsx
// Without keys — React re-renders ALL items when list changes
<ul>
  <li>A</li>
  <li>B</li>
</ul>

// With keys — React only updates what actually changed
<ul>
  <li key="a">A</li>
  <li key="b">B</li>
</ul>
```

**Why index as key is bad:**
```jsx
// BAD — if items are reordered/removed, index shifts and React re-renders wrong elements
{items.map((item, index) => <Item key={index} data={item} />)}

// GOOD — use unique, stable identifiers
{items.map(item => <Item key={item.id} data={item} />)}
```

---

## 2 JSX

### Q: What is JSX and how does it work under the hood?

**Answer:**

JSX is a syntax extension that lets you write HTML-like code in JavaScript. **It is NOT valid JavaScript** — it gets transpiled by Babel.

```jsx
// JSX
const element = <h1 className="title">Hello, {name}!</h1>;

// Transpiles to (React 17+ with new transform):
import { jsx as _jsx } from 'react/jsx-runtime';
const element = _jsx('h1', { className: 'title', children: ['Hello, ', name, '!'] });

// Old transform (React 16):
const element = React.createElement('h1', { className: 'title' }, `Hello, ${name}!`);
```

**Key Rules:**
- `className` instead of `class` (reserved word)
- `htmlFor` instead of `for`
- Self-close tags: `<img />`, `<input />`
- Must return a **single root element** (or use `<Fragment>` / `<>`)
- JS expressions inside `{}` — but not statements (`if`, `for`)

### Q: What are Fragments?

**Answer:**

Fragments let you group elements **without adding extra DOM nodes**:
```jsx
// Adds unnecessary <div> to DOM
return (
  <div>
    <Header />
    <Main />
  </div>
);

// Fragment — no extra wrapper
return (
  <>
    <Header />
    <Main />
  </>
);

// Long form with key support
return (
  <React.Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.description}</dd>
  </React.Fragment>
);
```

---

## 3 Components — Class vs Function

### Q: What are the differences between class and functional components?

**Answer:**

| Feature | Class Components | Functional Components |
|---|---|---|
| **Syntax** | ES6 class extending `React.Component` | Plain function returning JSX |
| **State** | `this.state` + `this.setState()` | `useState()` hook |
| **Lifecycle** | `componentDidMount`, etc. | `useEffect()` hook |
| **`this` binding** | Required (error-prone) | Not needed |
| **Performance** | Slightly heavier | Slightly lighter |
| **Modern usage** | Legacy codebases | Recommended approach |

```jsx
// Class component
class Counter extends React.Component {
  state = { count: 0 };
  increment = () => this.setState(prev => ({ count: prev.count + 1 }));
  render() {
    return <button onClick={this.increment}>{this.state.count}</button>;
  }
}

// Functional component (modern)
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

**Why functional components won:**
- Simpler, less boilerplate
- Hooks enable code reuse via custom hooks (vs HOCs/render props)
- No `this` confusion
- Easier to test and reason about

---

## 4 State and Props

### Q: What is the difference between state and props?

**Answer:**

| Feature | Props | State |
|---|---|---|
| **Ownership** | Passed by parent | Managed internally |
| **Mutability** | Read-only (immutable) | Mutable via setter |
| **Purpose** | Configure component | Track dynamic data |
| **Re-render trigger** | When parent re-renders with new props | When `setState`/setter is called |

### Q: Why is state immutable in React? What happens if you mutate it directly?

**Answer:**

React compares the **reference** (not deep equality) to determine if state changed. If you mutate the existing object, the reference stays the same, so **React won't re-render**.

```jsx
// BAD — direct mutation, no re-render
const handleClick = () => {
  user.name = 'John';    // Mutating existing object
  setUser(user);          // Same reference! React thinks nothing changed
};

// GOOD — create new object
const handleClick = () => {
  setUser({ ...user, name: 'John' });  // New reference → re-render
};

// GOOD — arrays
setItems([...items, newItem]);           // Add
setItems(items.filter(i => i.id !== id)); // Remove
setItems(items.map(i => i.id === id ? { ...i, updated: true } : i)); // Update
```

### Q: How does `setState` batching work?

**Answer:**

**React 18+:** All state updates are batched automatically — even in `setTimeout`, promises, and event handlers.

```jsx
// React 18 — these are batched into ONE re-render
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  setName('John');
  // Only ONE re-render happens
}

// Even in async code (React 18+)
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // Still ONE re-render in React 18
  // In React 17, this would be TWO re-renders
}, 1000);

// To opt out of batching (rare):
import { flushSync } from 'react-dom';
flushSync(() => setCount(c => c + 1)); // Re-renders immediately
flushSync(() => setFlag(f => !f));      // Re-renders again
```

---

## 5 React Hooks (Core)

### Q: Explain `useState` — what are the rules and gotchas?

**Answer:**

```jsx
const [state, setState] = useState(initialValue);

// Lazy initialization — expensive computation runs only on first render
const [data, setData] = useState(() => expensiveComputation());

// Functional updates — when new state depends on previous state
setCount(prevCount => prevCount + 1);
```

**Gotcha — Stale Closure:**
```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // BUG: count is always 0 (stale closure)
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []); // Empty deps — closure captures initial count

  // FIX: Use functional update
  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1); // Always gets latest value
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

### Q: Explain `useEffect` — lifecycle mapping and cleanup.

**Answer:**

```jsx
// componentDidMount — runs once after first render
useEffect(() => {
  fetchData();
}, []); // Empty dependency array

// componentDidUpdate — runs when `userId` changes
useEffect(() => {
  fetchUser(userId);
}, [userId]);

// componentDidMount + componentDidUpdate — runs every render
useEffect(() => {
  document.title = `Count: ${count}`;
}); // No dependency array

// componentWillUnmount — cleanup function
useEffect(() => {
  const subscription = dataSource.subscribe();
  return () => {
    subscription.unsubscribe(); // Cleanup on unmount or before re-running
  };
}, [dataSource]);
```

**Common Mistakes:**
```jsx
// Infinite loop — object/array in deps creates new reference each render
useEffect(() => {
  fetchData(options);
}, [options]); // If options = { limit: 10 }, new object each render!

// FIX: Use useMemo or specific primitive deps
const options = useMemo(() => ({ limit: 10 }), []);
useEffect(() => {
  fetchData(options);
}, [options]); // Stable reference now
```

### Q: What is `useRef`? When should you use it?

**Answer:**

`useRef` creates a **mutable container** (`.current`) that persists across renders **without causing re-renders** when changed.

```jsx
// 1. DOM access
function TextInput() {
  const inputRef = useRef(null);
  const focusInput = () => inputRef.current.focus();
  return <input ref={inputRef} />;
}

// 2. Store mutable value that doesn't trigger re-render
function Timer() {
  const intervalRef = useRef(null);

  const start = () => {
    intervalRef.current = setInterval(() => console.log('tick'), 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
  };
}

// 3. Track previous value
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

### Q: What is `useMemo` vs `useCallback`? When to use each?

**Answer:**

| Hook | Caches | Use when |
|---|---|---|
| `useMemo` | **Computed value** | Expensive calculations you don't want to repeat |
| `useCallback` | **Function reference** | Passing callbacks to optimized child components |

```jsx
// useMemo — cache expensive computation
const sortedList = useMemo(() => {
  return items.sort((a, b) => a.price - b.price);
}, [items]); // Re-compute only when items changes

// useCallback — cache function reference
const handleClick = useCallback((id) => {
  deleteItem(id);
}, [deleteItem]); // Same reference unless deleteItem changes

// useCallback is equivalent to:
const handleClick = useMemo(() => (id) => deleteItem(id), [deleteItem]);
```

**When NOT to use them:**
- Simple computations (overhead of memoization > computation cost)
- Functions not passed to memoized children
- Premature optimization without profiling

### Q: What is `useReducer`? When to use it over `useState`?

**Answer:**

`useReducer` is preferred when state logic is **complex**, involves **multiple sub-values**, or when the next state depends on the previous state.

```jsx
const initialState = { count: 0, error: null, loading: false };

function reducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'DECREMENT':
      return { ...state, count: state.count - 1 };
    case 'SET_ERROR':
      return { ...state, error: action.payload, loading: false };
    case 'SET_LOADING':
      return { ...state, loading: true, error: null };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
    </div>
  );
}
```

**Use `useReducer` when:**
- State has **multiple related fields** (loading, error, data)
- State transitions are **complex** (form with validation)
- You want to **centralize** state logic and make it **testable**
- Multiple actions need to update state in different ways

---

## 6 Custom Hooks

### Q: What are custom hooks and how do you build them?

**Answer:**

Custom hooks let you **extract and reuse stateful logic** between components. They must start with `use` and can call other hooks.

```jsx
// useFetch — reusable data fetching
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    const fetchData = async () => {
      try {
        setLoading(true);
        const res = await fetch(url, { signal: controller.signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const json = await res.json();
        setData(json);
        setError(null);
      } catch (err) {
        if (err.name !== 'AbortError') {
          setError(err.message);
        }
      } finally {
        setLoading(false);
      }
    };

    fetchData();
    return () => controller.abort(); // Cleanup: cancel on unmount/url change
  }, [url]);

  return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <Spinner />;
  if (error) return <Error message={error} />;
  return <Profile user={user} />;
}
```

```jsx
// useLocalStorage — persist state to localStorage
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// Usage
const [theme, setTheme] = useLocalStorage('theme', 'dark');
```

```jsx
// useDebounce — debounce rapidly changing values
function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

// Usage
function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) searchAPI(debouncedQuery);
  }, [debouncedQuery]);
}
```

---

## 7 Component Lifecycle

### Q: Map class lifecycle methods to hooks.

**Answer:**

| Class Method | Hook Equivalent |
|---|---|
| `constructor` | `useState(initialValue)` or `useRef(initialValue)` |
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate` | `useEffect(() => { ... }, [deps])` |
| `componentWillUnmount` | `useEffect(() => { return () => cleanup() }, [])` |
| `shouldComponentUpdate` | `React.memo(Component)` |
| `getDerivedStateFromProps` | Update state during rendering |
| `getSnapshotBeforeUpdate` | No direct equivalent |
| `componentDidCatch` | No hook (use class Error Boundaries) |

**Derived state from props (without `getDerivedStateFromProps`):**
```jsx
function Component({ items }) {
  // Adjust state during rendering — NO useEffect needed
  const [prevItems, setPrevItems] = useState(items);
  const [selection, setSelection] = useState(null);

  if (items !== prevItems) {
    setPrevItems(items);
    setSelection(null); // Reset selection when items change
  }
}
```

---

## 8 Context API

### Q: How does Context API work and what are its limitations?

**Answer:**

Context provides a way to pass data through the component tree **without prop drilling**.

```jsx
// 1. Create context
const ThemeContext = createContext('light');

// 2. Provider
function App() {
  const [theme, setTheme] = useState('dark');
  const value = useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      <Toolbar />
    </ThemeContext.Provider>
  );
}

// 3. Consume
function ThemedButton() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <button
      style={{ background: theme === 'dark' ? '#333' : '#fff' }}
      onClick={() => setTheme(t => t === 'dark' ? 'light' : 'dark')}
    >
      Toggle Theme
    </button>
  );
}
```

**Limitations:**
1. **All consumers re-render** when the context value changes — even if they only use a subset of the value.
2. Not suitable for **high-frequency updates** (e.g., mouse position, animations).
3. No built-in **selectors** (unlike Redux).

**Mitigation — Split contexts:**
```jsx
// Instead of one big context, split into separate contexts
const ThemeContext = createContext();
const UserContext = createContext();
const LocaleContext = createContext();

// Components only re-render when THEIR context changes
```

---

## 9 State Management — Redux

### Q: Explain Redux architecture and data flow.

**Answer:**

**Core Principles:**
1. **Single source of truth** — one store for entire app state.
2. **State is read-only** — only way to change is dispatching actions.
3. **Changes via pure functions** — reducers are pure functions.

```
Action → Dispatch → Reducer → New State → UI Re-render
```

**Redux Toolkit (Modern Redux):**
```jsx
import { createSlice, configureStore } from '@reduxjs/toolkit';

// 1. Create slice (reducer + actions)
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0, status: 'idle' },
  reducers: {
    increment: (state) => { state.value += 1; },        // Immer allows "mutation"
    decrement: (state) => { state.value -= 1; },
    incrementByAmount: (state, action) => { state.value += action.payload; },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchCount.pending, (state) => { state.status = 'loading'; })
      .addCase(fetchCount.fulfilled, (state, action) => {
        state.status = 'idle';
        state.value = action.payload;
      });
  },
});

// 2. Async thunk
const fetchCount = createAsyncThunk('counter/fetch', async (amount) => {
  const response = await fetch(`/api/counter?amount=${amount}`);
  return response.json();
});

// 3. Store
const store = configureStore({
  reducer: { counter: counterSlice.reducer },
});

// 4. Usage in component
function Counter() {
  const count = useSelector((state) => state.counter.value);
  const status = useSelector((state) => state.counter.status);
  const dispatch = useDispatch();

  return (
    <div>
      <span>{count}</span>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(fetchCount(5))}>Fetch</button>
    </div>
  );
}
```

### Q: Redux vs Context API — when to use which?

**Answer:**

| Feature | Context API | Redux (RTK) |
|---|---|---|
| **Complexity** | Simple, built-in | Additional dependency |
| **Boilerplate** | Minimal | Moderate (less with RTK) |
| **DevTools** | None | Excellent (time-travel debugging) |
| **Performance** | All consumers re-render | Selective re-renders via `useSelector` |
| **Middleware** | None | Thunk, Saga, custom middleware |
| **Best for** | Theme, locale, auth | Complex app state, frequent updates |
| **Server state** | Not ideal | Use RTK Query or React Query instead |

**Modern Recommendation:** For server state (API data), use **React Query / TanStack Query** or **RTK Query** instead of managing it in Redux.

---

## 10 Performance Optimization

### Q: How do you optimize React application performance?

**Answer:**

**1. Prevent unnecessary re-renders:**

```jsx
// React.memo — skip re-render if props haven't changed
const ExpensiveList = React.memo(function ExpensiveList({ items, onItemClick }) {
  return items.map(item => (
    <div key={item.id} onClick={() => onItemClick(item.id)}>
      {item.name}
    </div>
  ));
});

// Custom comparison function
const MemoizedComponent = React.memo(Component, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id; // Only re-render if id changes
});
```

**2. Code Splitting & Lazy Loading:**
```jsx
import { lazy, Suspense } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart data={data} />
    </Suspense>
  );
}

// Route-based splitting
const routes = [
  { path: '/dashboard', element: lazy(() => import('./pages/Dashboard')) },
  { path: '/settings', element: lazy(() => import('./pages/Settings')) },
];
```

**3. Virtualization for large lists:**
```jsx
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

**4. Debounce expensive operations:**
```jsx
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedSearch = useMemo(
    () => debounce((q) => fetchResults(q), 300),
    []
  );

  const handleChange = (e) => {
    setQuery(e.target.value);
    debouncedSearch(e.target.value);
  };

  return <input value={query} onChange={handleChange} />;
}
```

**5. `useMemo` and `useCallback`:**
```jsx
// Avoid recalculating on every render
const filteredItems = useMemo(
  () => items.filter(item => item.category === selectedCategory),
  [items, selectedCategory]
);

// Stable reference for child components
const handleDelete = useCallback(
  (id) => dispatch(deleteItem(id)),
  [dispatch]
);
```

### Q: What causes unnecessary re-renders?

**Answer:**

1. **Parent re-renders** → all children re-render (fix: `React.memo`)
2. **New object/array references** in props (fix: `useMemo`)
3. **New function references** in props (fix: `useCallback`)
4. **Context value changes** → all consumers re-render (fix: split contexts)
5. **Inline objects in JSX** (fix: extract to constants or `useMemo`)

```jsx
// BAD — new object every render
<UserCard style={{ marginTop: 20 }} user={user} />

// GOOD — stable reference
const cardStyle = useMemo(() => ({ marginTop: 20 }), []);
<UserCard style={cardStyle} user={user} />
```

---

## 11 React Fiber Architecture

### Q: What is React Fiber?

**Answer:**

Fiber is React's **reconciliation engine** (introduced in React 16). It replaced the old "stack" reconciler with an **incremental, interruptible rendering** engine.

**Key Features:**
1. **Incremental rendering:** Break rendering work into chunks, spread over multiple frames.
2. **Priority-based updates:** Assign priority to different types of work (user interactions > data fetch > offscreen).
3. **Pause, abort, resume:** Can stop rendering work and come back to it later.
4. **Concurrency:** Enables features like `Suspense`, `useTransition`, `useDeferredValue`.

**How it works:**
- Each React element gets a corresponding **Fiber node** (a JavaScript object).
- Fiber nodes form a **linked list tree** (child, sibling, return pointers).
- Rendering happens in two phases:
  - **Render phase (interruptible):** Compute changes by walking the Fiber tree. No side effects.
  - **Commit phase (synchronous):** Apply changes to the DOM. Cannot be interrupted.

---

## 12 Concurrent Features

### Q: What is `useTransition`?

**Answer:**

`useTransition` lets you mark state updates as **non-urgent**, so React can keep the UI responsive during expensive re-renders.

```jsx
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value);  // Urgent — update input immediately

    startTransition(() => {
      setResults(filterLargeDataset(e.target.value)); // Non-urgent — can be interrupted
    });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultList results={results} />
    </div>
  );
}
```

### Q: What is `useDeferredValue`?

**Answer:**

`useDeferredValue` creates a **deferred version** of a value that "lags behind" the actual value during urgent updates.

```jsx
function SearchPage({ query }) {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <ExpensiveList query={deferredQuery} />
    </div>
  );
}
```

**`useTransition` vs `useDeferredValue`:**
- `useTransition`: You **control** which state update is low-priority.
- `useDeferredValue`: You **don't control** the state update — you receive a prop and defer its effect.

---

## 13 Error Boundaries

### Q: What are Error Boundaries and how do they work?

**Answer:**

Error Boundaries are **class components** that catch JavaScript errors in their **child component tree**, log the error, and display a fallback UI. **There is no hook equivalent.**

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log to error reporting service
    logErrorToService(error, errorInfo.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <pre>{this.state.error?.message}</pre>
          <button onClick={() => this.setState({ hasError: false })}>
            Try Again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage — wrap around risky components
<ErrorBoundary>
  <UserProfile />
</ErrorBoundary>
```

**What they DON'T catch:**
- Event handlers (use try/catch)
- Async code (promises)
- Server-side rendering
- Errors in the error boundary itself

**Best Practice:** Use multiple error boundaries at different levels:
```jsx
<ErrorBoundary>                    {/* App-level fallback */}
  <Header />
  <ErrorBoundary>                  {/* Feature-level fallback */}
    <Dashboard />
  </ErrorBoundary>
  <ErrorBoundary>
    <Sidebar />
  </ErrorBoundary>
</ErrorBoundary>
```

---

## 14 Controlled vs Uncontrolled Components

### Q: What is the difference?

**Answer:**

| Feature | Controlled | Uncontrolled |
|---|---|---|
| **Data source** | React state | DOM (ref) |
| **Update mechanism** | `onChange` → `setState` | DOM handles it |
| **Read value** | From state | From `ref.current.value` |
| **Validation** | On every change (real-time) | On submit |
| **Use case** | Most forms | Simple forms, file inputs |

```jsx
// Controlled — React owns the data
function ControlledForm() {
  const [email, setEmail] = useState('');

  return (
    <input
      value={email}
      onChange={(e) => setEmail(e.target.value)}
    />
  );
}

// Uncontrolled — DOM owns the data
function UncontrolledForm() {
  const emailRef = useRef();

  const handleSubmit = () => {
    console.log(emailRef.current.value);
  };

  return <input ref={emailRef} defaultValue="initial@email.com" />;
}
```

**When to use uncontrolled:**
- File inputs (`<input type="file" />` — must be uncontrolled)
- Integration with non-React libraries
- When you only need the value on submit

---

## 15 Higher-Order Components, Render Props & Hooks

### Q: Compare HOC, Render Props, and Custom Hooks patterns.

**Answer:**

**1. Higher-Order Component (HOC):**
```jsx
function withAuth(WrappedComponent) {
  return function AuthComponent(props) {
    const user = useAuth();
    if (!user) return <Navigate to="/login" />;
    return <WrappedComponent {...props} user={user} />;
  };
}

const ProtectedDashboard = withAuth(Dashboard);
```

**2. Render Props:**
```jsx
function MouseTracker({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  // ... mouse tracking logic
  return render(position);
}

<MouseTracker render={({ x, y }) => <Cursor x={x} y={y} />} />
```

**3. Custom Hook (Modern — Preferred):**
```jsx
function useMousePosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);

  return position;
}

// Usage — clean and composable
function Cursor() {
  const { x, y } = useMousePosition();
  return <div style={{ left: x, top: y }} />;
}
```

**Why hooks won:**
- No "wrapper hell" (HOCs create deeply nested component trees)
- No naming collisions (HOCs can override props)
- TypeScript-friendly
- Composable — combine multiple hooks easily

---

## 16 Portals

### Q: What are React Portals?

**Answer:**

Portals let you render children into a **DOM node outside the parent component's hierarchy**, while maintaining the React component tree (events still bubble up).

```jsx
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.getElementById('modal-root') // Renders here in DOM
  );
}

// In index.html:
// <div id="root"></div>
// <div id="modal-root"></div>
```

**Use Cases:**
- Modals/Dialogs
- Tooltips
- Dropdown menus
- Toast notifications

**Key behavior:** Even though the modal renders in `#modal-root`, events **bubble up through the React tree** (not the DOM tree). A click inside the modal can be caught by a React parent in `#root`.

---

## 17 Server-Side Rendering (SSR) & Next.js

### Q: What is SSR and why use it?

**Answer:**

| Rendering Strategy | Description | When to Use |
|---|---|---|
| **CSR** (Client-Side) | Browser downloads empty HTML + JS bundle, renders on client | Dashboards, internal tools |
| **SSR** (Server-Side) | Server renders HTML on each request | Dynamic pages, personalized content |
| **SSG** (Static Generation) | HTML generated at build time | Blogs, docs, marketing pages |
| **ISR** (Incremental Static) | SSG + regeneration at intervals | E-commerce product pages |

**Next.js Data Fetching (App Router):**

```jsx
// Server Component (default in App Router) — runs on server
async function ProductPage({ params }) {
  const product = await fetch(`/api/products/${params.id}`);
  return <Product data={product} />;
}

// Client Component — runs on client
'use client';
function AddToCartButton({ productId }) {
  const [added, setAdded] = useState(false);
  return <button onClick={() => setAdded(true)}>Add to Cart</button>;
}
```

**Next.js App Router key concepts:**
- **Server Components** (default): No client-side JS, direct DB access, streaming
- **Client Components** (`'use client'`): Interactive, use hooks, browser APIs
- **Layouts**: Shared UI that persists across navigations
- **Loading/Error**: Built-in loading and error states per route segment

---

## 18 React Router

### Q: How does React Router work? Explain key concepts.

**Answer:**

```jsx
import { BrowserRouter, Routes, Route, Link, useNavigate, useParams, Outlet } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/users">Users</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/users" element={<UsersLayout />}>
          <Route index element={<UsersList />} />       {/* /users */}
          <Route path=":userId" element={<UserDetail />} /> {/* /users/123 */}
        </Route>
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}

// Layout with nested routes
function UsersLayout() {
  return (
    <div>
      <h1>Users</h1>
      <Outlet />  {/* Child routes render here */}
    </div>
  );
}

// Access params
function UserDetail() {
  const { userId } = useParams();
  const navigate = useNavigate();

  return (
    <div>
      <h2>User {userId}</h2>
      <button onClick={() => navigate('/users')}>Back</button>
    </div>
  );
}
```

**Protected Routes:**
```jsx
function ProtectedRoute({ children }) {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" replace />;
  return children;
}

<Route path="/dashboard" element={
  <ProtectedRoute>
    <Dashboard />
  </ProtectedRoute>
} />
```

---

## 19 Testing

### Q: How do you test React components?

**Answer:**

**Philosophy:** Test **behavior**, not implementation details. "Does the user see the right thing?" not "Did `setState` get called?"

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// 1. Rendering and querying
test('renders welcome message', () => {
  render(<Greeting name="John" />);
  expect(screen.getByText('Hello, John!')).toBeInTheDocument();
});

// 2. User interactions
test('increments counter on click', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  const button = screen.getByRole('button', { name: /increment/i });
  await user.click(button);

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});

// 3. Async operations
test('loads and displays user data', async () => {
  // Mock the fetch
  jest.spyOn(global, 'fetch').mockResolvedValue({
    json: () => Promise.resolve({ name: 'John' }),
  });

  render(<UserProfile userId="123" />);

  expect(screen.getByText('Loading...')).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText('John')).toBeInTheDocument();
  });
});

// 4. Form validation
test('shows error when submitting empty form', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  await user.click(screen.getByRole('button', { name: /submit/i }));

  expect(screen.getByText('Email is required')).toBeInTheDocument();
});
```

**Query Priority (in order of preference):**
1. `getByRole` — accessible role (button, heading, textbox)
2. `getByLabelText` — form element by label
3. `getByPlaceholderText`
4. `getByText` — visible text
5. `getByTestId` — last resort

---

## 20 TypeScript with React

### Q: What are common TypeScript patterns in React?

**Answer:**

```tsx
// Component props
interface UserCardProps {
  user: User;
  onEdit: (id: string) => void;
  variant?: 'compact' | 'full';   // Optional with union type
  children: React.ReactNode;       // Any renderable content
}

function UserCard({ user, onEdit, variant = 'full', children }: UserCardProps) {
  return <div>{/* ... */}</div>;
}

// Event handlers
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setQuery(e.target.value);
};

const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
};

// useRef with types
const inputRef = useRef<HTMLInputElement>(null);  // DOM ref
const countRef = useRef<number>(0);                // Mutable value

// useState with types
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);

// Generic components
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map(renderItem)}</ul>;
}

// Discriminated unions for component states
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function UserProfile() {
  const [state, setState] = useState<AsyncState<User>>({ status: 'idle' });

  if (state.status === 'success') {
    return <div>{state.data.name}</div>; // TypeScript knows `data` exists here
  }
}
```

---

## 21 Common Coding Patterns

### Q: Implement a debounced search input.

```jsx
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query.trim()) {
      setResults([]);
      return;
    }

    const controller = new AbortController();
    const timer = setTimeout(async () => {
      try {
        setLoading(true);
        const res = await fetch(`/api/search?q=${query}`, {
          signal: controller.signal,
        });
        const data = await res.json();
        setResults(data);
      } catch (err) {
        if (err.name !== 'AbortError') console.error(err);
      } finally {
        setLoading(false);
      }
    }, 300);

    return () => {
      clearTimeout(timer);
      controller.abort();
    };
  }, [query]);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      {loading && <Spinner />}
      <ul>
        {results.map(item => <li key={item.id}>{item.name}</li>)}
      </ul>
    </div>
  );
}
```

### Q: Implement infinite scroll.

```jsx
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const observerRef = useRef();
  const lastItemRef = useCallback(
    (node) => {
      if (observerRef.current) observerRef.current.disconnect();

      observerRef.current = new IntersectionObserver((entries) => {
        if (entries[0].isIntersecting && hasMore) {
          setPage((p) => p + 1);
        }
      });

      if (node) observerRef.current.observe(node);
    },
    [hasMore]
  );

  useEffect(() => {
    fetchItems(page).then((newItems) => {
      setItems((prev) => [...prev, ...newItems]);
      setHasMore(newItems.length > 0);
    });
  }, [page]);

  return (
    <ul>
      {items.map((item, i) => (
        <li key={item.id} ref={i === items.length - 1 ? lastItemRef : null}>
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

---

> **Tip:** For experienced-level interviews, focus on **hooks deep-dive (closures, stale state), performance optimization, reconciliation, concurrent features, and system design patterns** — these are where interviewers assess depth of understanding.
