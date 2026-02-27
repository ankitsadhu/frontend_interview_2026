# React Interview Q&A — MANG Level

> **Target**: Meta / Amazon / Netflix / Google senior & staff engineer interviews  
> **Covers**: All topics in `qna.md` with deep-dive answers + scenario-based questions

---

## Table of Contents

1. [History & Virtual DOM](#1-history--virtual-dom)
2. [Reconciliation & Fiber Architecture](#2-reconciliation--fiber-architecture)
3. [Diffing Algorithm](#3-diffing-algorithm)
4. [Render & Commit Phases](#4-render--commit-phases)
5. [JSX & Babel](#5-jsx--babel)
6. [Props](#6-props)
7. [Events](#7-events)
8. [Component Patterns (Smart/Dumb, Controlled/Uncontrolled)](#8-component-patterns)
9. [Rendering & Re-renders](#9-rendering--re-renders)
10. [Built-in Hooks](#10-built-in-hooks)
11. [Custom Hooks](#11-custom-hooks)
12. [React APIs](#12-react-apis)
13. [Forms](#13-forms)
14. [React Portals](#14-react-portals)
15. [Rendering Patterns (HOC, Render Props, Compound, etc.)](#15-rendering-patterns)
16. [Advanced Rendering (Hydration, SSR, CSR, SSG, ISR)](#16-advanced-rendering)
17. [Concurrency](#17-concurrency)
18. [Error Handling](#18-error-handling)
19. [Suspense](#19-suspense)
20. [Performance](#20-performance)
21. [Security](#21-security)
22. [React Router](#22-react-router)
23. [Bundling: Webpack / Vite](#23-bundling-webpack--vite)
24. [Code Splitting](#24-code-splitting)
25. [Folder Structure](#25-folder-structure)
26. [Scenario-Based MANG Questions](#26-scenario-based-mang-questions)

---

## 1. History & Virtual DOM

### Q: When was React released and what was notable about it?
**A:** React was open-sourced on **May 29, 2013** at JSConf US by Jordan Walke (Facebook). At that time, there were **no class components** — React originally used `createClass()`. ES6 classes were introduced in React v0.13 (March 2015).

### Q: What is the Virtual DOM (VDOM)?
**A:** The Virtual DOM is a **lightweight, in-memory JavaScript representation** of the actual DOM tree. It is a plain object tree that mirrors the real DOM node structure.

```
Real DOM Node:        Virtual DOM Object:
<div class="box">    { type: 'div', props: { className: 'box' },
  <p>Hello</p>  →      children: [{ type: 'p', props: {}, children: ['Hello'] }]
</div>               }
```

**Why it exists:** Direct DOM manipulation is expensive. React batches and minimises real DOM changes by:
1. Maintaining two VDOM trees (current & work-in-progress)
2. Diffing them (reconciliation)
3. Flushing only the diff to the real DOM (commit phase)

### Q: Does VDOM make React faster than vanilla JS?
**A:** Not always. Vanilla JS with targeted DOM operations can outperform React. VDOM's value is **developer ergonomics at scale** — it makes UI predictable as a function of state (`UI = f(state)`) without the developer having to manually track which DOM nodes changed.

---

## 2. Reconciliation & Fiber Architecture

### Q: What is Reconciliation?
**A:** Reconciliation is React's algorithm that determines **what changed in the VDOM** and produces the minimal set of real DOM mutations. It compares the new element tree (after a state/prop change) against the previous tree.

#### Old Stack Reconciler (pre-React 16) Problems:
- **Synchronous & blocking**: processed the entire tree in one go on the call stack
- **Non-interruptible**: "stack-based" — browser could not paint or handle user input mid-reconcile
- Long lists or deep trees caused **jank** (dropped frames, unresponsive UI)

### Q: What is Fiber and why was it introduced? (React 16.0)
**A:** **Fiber** is a complete rewrite of React's reconciliation engine, replacing the Stack reconciler. Key goals:

| Feature | Stack Reconciler | Fiber Reconciler |
|---------|-----------------|-----------------|
| Execution | Synchronous, uninterruptible | Asynchronous, interruptible |
| Unit of work | Full VDOM tree | Individual fiber nodes |
| Scheduling | None | Priority-based scheduling |
| Concurrency support | No | Yes (concurrent mode) |

**Fiber Node**: Each React element (component, DOM node) gets a corresponding fiber node — a plain JS object tracking:
- `type`, `key`, `ref`
- `stateNode` (class instance or DOM node)
- `return` (parent fiber), `child` (first child fiber), `sibling` (next sibling fiber) — a **linked list** structure
- `pendingProps`, `memoizedProps`, `memoizedState`
- `effectTag` (what DOM mutation is needed)
- `lanes` (priority)

#### Why linked list?
Unlike a tree traversal on a call stack, a linked list lets React **pause after any fiber node** and resume later. The current position is saved as a pointer.

```
FiberNode {
  type: 'div',
  child: FiberNode { type: 'p', sibling: null, return: <div fiber> },
  sibling: FiberNode { type: 'span', ... },
  return: FiberNode { type: 'App', ... },
  effectTag: UPDATE,
  lanes: 1 (SyncLane)
}
```

### Q: What is Incremental Rendering?
**A:** Fiber splits reconciliation work into **chunks (fiber nodes)**. After completing each unit, React checks if the browser has higher-priority work (user input, animation frame). If yes, it **pauses** and yields to the browser, then resumes reconciliation later. This prevents jank.

### Q: What is Priority Scheduling in Fiber?
**A:** React assigns every update a **lane** (priority level):
- `SyncLane` — e.g., click events (highest)
- `InputContinuousLane` — e.g., drag
- `DefaultLane` — normal state updates
- `TransitionLane` — `startTransition` updates (low priority)
- `IdleLane` — offscreen work (lowest)

Higher-priority updates can **interrupt** lower-priority ongoing reconciliation.

### Q: What is Interruptible Rendering?
**A:** During the **render phase**, Fiber can stop mid-way and restart. This is safe because the render phase is **pure** (no side effects, no DOM mutations). Only the commit phase (which is synchronous and uninterruptible) touches the DOM.

### Q: How does `requestIdleCallback` compare to React's scheduling?
**A:** React does **not use** `requestIdleCallback` in production because:
1. It has inconsistent browser support
2. It fires at low frequency (~20fps on some browsers)
3. It doesn't integrate well with React's lane-based priority system

React uses its own **scheduler** package (`react-scheduler`) that uses `MessageChannel` for asynchronous yielding, giving a ~5ms time-slice per frame and supporting priority levels natively.

---

## 3. Diffing Algorithm

### Q: What is React's diffing algorithm?
**A:** React's diffing (tree comparison) algorithm converts an O(n³) general tree diff problem into **O(n)** using two heuristics:

#### Heuristic 1: Different element types → unmount + remount
If the root element type changes (e.g., `<div>` → `<span>`), React **destroys the old tree** and builds a new one from scratch.

```jsx
// Old            // New
<div>         →   <span>       // React destroys <div> and all children
  <Counter />       <Counter />  // Counter loses all state
</div>            </span>
```

#### Heuristic 2: Same type → only update changed attributes
If the type is the same, React keeps the DOM node and **only updates changed props**.

```jsx
// Old                    // New
<div className="old" />  → <div className="new" />  // only className updated
```

#### Heuristic 3: Keys for list reconciliation
For children arrays, React uses `key` props to match elements between renders.

### Q: Why is React's diff O(n)?
**A:** React only does a **single pass** through the tree (each node visited once). It doesn't try to find the optimal edit distance between trees (which would be O(n³) with dynamic programming). The two heuristics above make it O(n) at the cost of some sub-optimality — but these heuristics are correct ~99% of the time in real apps.

### Q: Why doesn't React do a full tree diff?
**A:** A theoretically optimal tree diff is O(n³) — for 1000 elements that's 1 billion operations. It's computationally infeasible for UI rendering that must happen at 60fps. React's heuristics achieve near-optimal results in practice with a fraction of the cost.

### Q: What heuristics does React use?
**A:** (See above) Two core heuristics:
1. **Different type → full subtree replace**
2. **Same type → reconcile props and recurse into children using keys**

### Q: Why is using array index as key dangerous?
**A:** When items are reordered, added to the beginning, or deleted, index-based keys cause React to **reuse incorrect fiber nodes**:

```jsx
// Items: ['A', 'B', 'C'] → ['X', 'A', 'B', 'C']
// With index keys:
// key=0: A → X (React updates, doesn't reuse) — loses state
// key=1: B → A
// key=2: C → B
// key=3: undefined → C (new mount)

// With stable IDs as keys:
// React matches by ID, moves nodes, preserves state
```

**Safe to use index as key only when**: the list is static, never reordered, and items have no state.

---

## 4. Render & Commit Phases

### Q: What are the render and commit phases?
**A:** React's update cycle has two distinct phases:

#### Render Phase (Reconciliation — async, interruptible)
- React calls your component functions (or `render()` for class components)
- Builds the work-in-progress fiber tree
- Runs the diffing algorithm
- **No side effects, no DOM mutations** — pure computation
- Can be **paused, interrupted, or restarted**

#### Commit Phase (Synchronous — cannot be interrupted)
React commits the fiber tree to the DOM in three sub-phases:
1. **Before mutation** — `getSnapshotBeforeUpdate`, `useLayoutEffect` cleanup
2. **Mutation** — DOM insertions, updates, deletions; `ref` detachment
3. **Layout** — `useLayoutEffect` callbacks, `ref` attachment
4. (async) **Passive effects** — `useEffect` callbacks scheduled after paint

```
Render Phase → Commit Phase
[pure, interruptible]  [sync, side-effects ok, DOM mutations]
```

---

## 5. JSX & Babel

### Q: Why JSX and not plain HTML?
**A:** JSX is a **syntax extension** that allows writing HTML-like code in JavaScript files. It is NOT HTML — it compiles to `React.createElement()` calls.

```jsx
// JSX
const el = <div className="box"><p>{name}</p></div>;

// After Babel compilation (old transform)
const el = React.createElement(
  'div',
  { className: 'box' },
  React.createElement('p', null, name)
);

// New JSX transform (React 17+) — no React import needed
import { jsx as _jsx } from 'react/jsx-runtime';
const el = _jsx('div', { className: 'box', children: _jsx('p', { children: name }) });
```

**Benefits of JSX:**
- Co-locate markup with logic (not templating, but composition)
- Full JavaScript power in templates (`{}` expressions, map, ternary)
- Compile-time optimisations by babel/bundler
- Better error messages than template strings

---

## 6. Props

### Q: What are default props?
**A:** Default values for props when they aren't passed:
```jsx
// Modern (recommended)
function Button({ label = 'Click me', disabled = false }) { ... }

// Legacy
Button.defaultProps = { label: 'Click me' }; // deprecated in React 19
```

### Q: What are children as props?
**A:** `props.children` is a special prop that holds whatever is between the JSX opening and closing tags. It can be a string, element, array, or function.

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}
// Usage:
<Card><p>Hello</p></Card>
```

### Q: What are callback props (functions as props)?
**A:** Passing functions from parent to child for child-to-parent communication (React is one-way data flow):
```jsx
function Parent() {
  const handleClick = (value) => console.log(value);
  return <Child onClick={handleClick} />;
}
function Child({ onClick }) {
  return <button onClick={() => onClick('hi')}>Click</button>;
}
```

### Q: Why do we use `forwardRef`?
**A:** By default, `ref` is not passed through as a prop. `forwardRef` allows a parent to attach a `ref` to a **child's DOM node** (or expose child imperative handles):

```jsx
const Input = React.forwardRef((props, ref) => (
  <input {...props} ref={ref} />
));

// Parent
const inputRef = useRef();
<Input ref={inputRef} />
inputRef.current.focus(); // works!
```

---

## 7. Events

### Q: What are Synthetic Events? Why does React use them?
**A:** React wraps the native browser event in a **SyntheticEvent** — a cross-browser wrapper with the same interface as native events (`.stopPropagation()`, `.preventDefault()`, `.target`, etc.).

**Why:**
- **Cross-browser consistency** — normalise IE vs Chrome vs Safari differences
- **Performance via event delegation** — React attaches a single listener per event type at the root, not per element
- **Event pooling** (React <17) — SyntheticEvent objects were pooled and reused (now removed)

### Q: How does event delegation work in React?
**A:** Instead of attaching `onClick` listeners to each individual DOM node, React attaches **one listener per event type** at the **root container** (`document` in React ≤16, `rootElement` in React 17+). Events bubble up to this listener, React determines which fiber triggered the event, and calls the appropriate handler.

### Q: What changed in React 17's event system?
**A:** React 17 moved event delegation from `document` to the **React root DOM container**:
- `document.getElementById('root')` gets the event listeners instead of `document`
- **Why?** Allows multiple React versions to coexist on the same page (micro-frontends, incremental upgrades)
- Event propagation now stops properly at the root instead of the document level

### Q: Why do events attach to root?
**A:** Single listener at root = O(1) event listeners regardless of element count. Adding/removing components doesn't require adding/removing event listeners — a huge performance win for large trees.

---

## 8. Component Patterns

### Q: What is the difference between Smart and Dumb components?
| Smart (Container) | Dumb (Presentational) |
|---|---|
| Contains business logic | Purely renders UI |
| Manages state, side effects | Receives props, calls callbacks |
| Uses hooks, Context | No hooks (ideally) |
| e.g., `UserListContainer` | e.g., `UserCard` |

> Modern React with hooks blurs this line — hooks extract logic, function components can be smart.

### Q: Controlled vs Uncontrolled Components?
**A:**

**Controlled**: React state is the **single source of truth** for the input value.
```jsx
const [val, setVal] = useState('');
<input value={val} onChange={e => setVal(e.target.value)} />
```

**Uncontrolled**: The DOM manages the value; React reads it via a `ref`.
```jsx
const ref = useRef();
<input ref={ref} defaultValue="hello" />
// Read: ref.current.value
```

| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | DOM |
| Instant validation | Easy | Harder |
| Re-renders on every keystroke | Yes | No |
| Form libraries (RHF) | Can use both | Prefers uncontrolled |

---

## 9. Rendering & Re-renders

### Q: What happens when you call `setState`?
**A:** (In function components, `useState`'s setter):
1. React **enqueues** the state update (not immediate)
2. Schedules a re-render at the appropriate priority
3. During the re-render, the component function re-executes
4. React applies the new state value via the fiber's `memoizedState`
5. If the new state === old state (via `Object.is`), React **bails out** (no re-render)

### Q: What triggers re-rendering in React?
- `setState` / `useState` setter called with a new value
- `useReducer` dispatch
- Parent component re-renders (unless `React.memo` prevents it)
- Context value changes (`useContext`)
- `forceUpdate()` (class components)

### Q: Does updating a parent always re-render children?
**A:** **Yes by default**, unless:
- Child is wrapped in `React.memo` AND props haven't changed (by reference)
- Child uses `shouldComponentUpdate` or `PureComponent` (class components)

This is why `useCallback` and `useMemo` matter — they stabilise prop references.

### Q: What is referential equality?
**A:** JavaScript compares objects and arrays by **reference**, not value:
```js
{} === {}   // false — different references
[] === []   // false

// Common React bug:
function Parent() {
  const config = { theme: 'dark' }; // new reference every render!
  return <Child config={config} />;  // Child re-renders even with React.memo
}
// Fix:
const config = useMemo(() => ({ theme: 'dark' }), []); // stable reference
```

### Q: How to avoid unnecessary re-renders?
1. `React.memo` for components — memoize by props
2. `useMemo` for expensive computed values
3. `useCallback` for function props passed to memoized children
4. **State colocation** — keep state as low in the tree as possible
5. Split context — separate frequently-changing and rarely-changing contexts
6. Use libraries: `zustand`, `jotai` (atom-based, avoids global re-renders)

### Q: What causes tearing?
**A:** **Tearing** occurs in concurrent mode when React reads external state (non-React store) during an interruptible render. Different parts of the UI may read different "snapshots" of the same store during a single render — causing visual inconsistency.

**Solution:** `useSyncExternalStore` — designed specifically to prevent tearing for external stores.

### Q: What is layout shift?
**A:** Cumulative Layout Shift (CLS) — when rendered content moves after initial paint because async content (images, ads) loads and pushes existing content. React-specific causes: SSR mismatches, late-loading components without skeleton placeholders.

---

## 10. Built-in Hooks

### 10a. State Hooks

### Q: Why does `setState` not update immediately?
**A:** React **batches** state updates. The setter enqueues an update; the component re-renders happen asynchronously after the current event handler or effect completes. During the current render, `state` still holds the old value.

```jsx
const [count, setCount] = useState(0);
setCount(count + 1); // count is still 0 here
setCount(count + 1); // still based on stale count = 0
// After render: count = 1 (not 2!)

// Fix — use functional updater:
setCount(prev => prev + 1);
setCount(prev => prev + 1);
// After render: count = 2 ✓
```

### Q: What happens if you call `setState` during render?
**A:** React allows `setState` during render under a narrow rule: if called with a value different from the current state and called in the same component (not in effects/callbacks), React will **re-render that component immediately** before returning to the browser. This is how `getDerivedStateFromProps` is emulated in hooks. However, unguarded use causes infinite loops.

### Q: `useReducer` — when to use over `useState`?
**A:** Use `useReducer` when:
- State transitions are complex or involve multiple sub-values
- Next state depends on multiple old state values simultaneously
- You want to extract state logic for testability
- State updates form a clear "action" vocabulary

```jsx
const initialState = { count: 0, status: 'idle' };
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + 1 };
    case 'reset': return initialState;
    default: throw new Error();
  }
}
const [state, dispatch] = useReducer(reducer, initialState);
```

### Q: Why can Context cause performance issues?
**A:** Every time the Context **value** changes, **all consumers re-render** — even if they only use a small part of the context. Unlike Redux selectors, there is no built-in granular subscription.

```jsx
// Problem: changing `user` re-renders ALL consumers, including those only needing `theme`
const ctx = { user, theme, locale };

// Solutions:
// 1. Split into multiple contexts
// 2. Memoize context value: useMemo(() => ({ user, theme }), [user, theme])
// 3. Use state management libraries with selectors (Zustand, Jotai)
```

### Q: What is State Lifting?
**A:** Moving shared state to the **lowest common ancestor** of components that need it. The ancestor passes state down as props and update functions as callbacks.

---

### 10b. Component Lifecycle Hooks

### Q: Deep dive — `useEffect` dependency array
**A:**

| Dependency Array | Behavior |
|---|---|
| No array | Runs after **every** render |
| `[]` empty | Runs only after **mount** (and cleanup on unmount) |
| `[a, b]` | Runs after mount + whenever `a` or `b` changes |

**React compares deps using `Object.is`** (similar to `===`). Objects/arrays fail this check if recreated each render.

### Q: Why do stale closures happen in `useEffect`?
**A:** A closure **captures** variables at the time it's created. If an effect captures state/props and the dependency array doesn't include them, the effect uses **stale (old) values** from when it was created.

```jsx
// Bug: stale count
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // always uses count=0 from first render
  }, 1000);
  return () => clearInterval(id);
}, []); // Missing count dependency

// Fix 1: add count to deps (but interval clears/resets each time)
// Fix 2: use functional updater
setCount(prev => prev + 1);
```

### Q: `useEffect` cleanup — what are they and when do they run?
**A:** The function returned from `useEffect` is the **cleanup function**. It runs:
1. Before the next time the effect runs (between renders if deps change)
2. When the component unmounts

```jsx
useEffect(() => {
  const sub = eventBus.subscribe(handler);
  return () => sub.unsubscribe(); // cleanup: prevent memory leak
}, []);
```

### Q: How to avoid infinite loops in `useEffect`?
Common causes:
1. **Updating state that is in deps**: `useEffect(() => { setX(x+1); }, [x])` — x changes → effect runs → x changes → loop
2. **Object/array in deps**: new reference each render triggers effect each render
3. **Missing cleanup on subscriptions**

Fix: stabilise dependencies with `useMemo`/`useCallback`, or use functional updaters.

### Q: `useLayoutEffect` vs `useEffect` — internal difference?
**A:**

| | `useEffect` | `useLayoutEffect` |
|---|---|---|
| Timing | After paint (async) | After DOM mutation, before paint (sync) |
| Blocks paint? | No | Yes |
| Use case | Data fetching, subscriptions | Reading DOM measurements, preventing flash |
| Server rendering | Runs (with warning) | Doesn't run on server |

Internally: `useLayoutEffect` callbacks are called in the **Layout sub-phase** of commit. `useEffect` is scheduled as a **passive effect** after the browser has painted.

### Q: When does `useInsertionEffect` run?
**A:** Added in React 18. Runs **before** `useLayoutEffect` and before any DOM mutations. Intended for **CSS-in-JS libraries** that need to inject styles into the DOM before any reads/layout. Should not be used in application code.

```
useInsertionEffect → DOM mutations → useLayoutEffect → paint → useEffect
```

### Q: What is `useEffectEvent`? (React experimental)
**A:** A hook that creates an "effect event" — a function that always reads the **latest** props/state but is **not** a dependency of `useEffect`. Solves the stale closure problem without adding to deps:
```jsx
const onMessage = useEffectEvent((msg) => {
  // sees latest state/props — never stale
  log(roomId, msg); // roomId always up-to-date
});
useEffect(() => {
  const unsub = socket.subscribe(onMessage); // not listed as dep
  return () => unsub();
}, [socket]); // only dep is socket
```

### Q: What is `useSyncExternalStore`?
**A:** A hook for subscribing to external stores (non-React state) in a way that is **concurrent-mode safe** (prevents tearing):
```jsx
const count = useSyncExternalStore(
  store.subscribe,      // subscribe(callback) — call callback on change
  store.getSnapshot,    // getSnapshot() — return current value (sync)
  store.getServerSnapshot // optional — for SSR
);
```

---

### 10c. Performance Hooks

### Q: `useMemo` — when and how?
**A:** Memoizes the **result** of an expensive computation between renders.

```jsx
const sortedList = useMemo(() => {
  return items.sort((a, b) => a.name.localeCompare(b.name));
}, [items]); // only re-computes when `items` changes
```

**Don't over-use**: React's renders are usually fast. Only memoize when:
- Profiling confirms it's slow
- The value is a reference passed to a memoized child

### Q: `useCallback` — how is it different from `useMemo`?
**A:** `useCallback(fn, deps)` is sugar for `useMemo(() => fn, deps)`. It memoizes the **function reference** (not its return value). Useful to give stable callback references to memoized children.

```jsx
// Without useCallback: new function reference each render → Child re-renders
const handleClick = () => doSomething(id);

// With useCallback: same reference if id doesn't change
const handleClick = useCallback(() => doSomething(id), [id]);
```

### Q: `React.memo` — how does it work?
**A:** A HOC that wraps a component and **shallowly compares props** before re-rendering:
```jsx
const MyComp = React.memo(function MyComp({ name, count }) { ... });
// Custom comparator (like shouldComponentUpdate):
const MyComp = React.memo(MyComp, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id; // true = skip re-render
});
```

### Q: `useTransition` vs `useDeferredValue`
**A:**

| `useTransition` | `useDeferredValue` |
|---|---|
| Marks a state **update** as non-urgent | Marks a **value** as deferred |
| You control the setState call | Works with values from props/parent |
| Returns `[isPending, startTransition]` | Returns deferred value |

```jsx
// useTransition
const [isPending, startTransition] = useTransition();
startTransition(() => {
  setSearchQuery(input); // low priority update
});

// useDeferredValue
const deferredQuery = useDeferredValue(searchQuery); // deferred copy
// UI using deferredQuery updates at lower priority
```

---

### 10d. Ref Hooks

### Q: `useRef` — what is it really?
**A:** `useRef(initialValue)` returns a **mutable ref object** `{ current: initialValue }` that persists for the full component lifetime. Changes to `.current` do NOT trigger re-renders.

Two uses:
1. **DOM access**: `<input ref={inputRef} />` → `inputRef.current` is the DOM node
2. **Mutable instance variable**: Store previous values, timer IDs, flags, without triggering re-renders

### Q: `useImperativeHandle` — what does it do?
**A:** Customises what is exposed when a parent uses `ref` on a child wrapped in `forwardRef`:
```jsx
const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    reset: () => { inputRef.current.value = ''; }
    // parent only gets focus() and reset(), not the raw DOM node
  }));
  return <input ref={inputRef} />;
});
```

---

### 10e. Other Hooks

### Q: `useId` — what is it for?
**A:** Generates **stable, unique IDs** for accessibility attributes (linking `<label>` to `<input>`). SSR-safe — generates the same ID on server and client.
```jsx
const id = useId();
return <>
  <label htmlFor={id}>Name</label>
  <input id={id} />
</>;
```

### Q: `useDebugValue` — what is it for?
**A:** Displays a label for custom hooks in **React DevTools**:
```jsx
function useOnlineStatus() {
  const isOnline = useSyncExternalStore(...);
  useDebugValue(isOnline ? 'Online' : 'Offline');
  return isOnline;
}
```

---

### 10f. Server Component Hooks (React 19)

### Q: `useFormStatus`
**A:** A hook that gives the **pending state of a parent `<form>`** submission. Must be used in a component rendered inside the `<form>`.
```jsx
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}
```

### Q: `useOptimistic`
**A:** Lets you show an **optimistic UI update** immediately, before the server confirms the action:
```jsx
const [optimisticMessages, addOptimistic] = useOptimistic(messages,
  (state, newMsg) => [...state, { text: newMsg, sending: true }]
);
```

### Q: `useActionState`
**A:** Manages the state of a **server action** (form submission), returning `[state, dispatch, isPending]`.

---

## 11. Custom Hooks

### Q: What are custom hooks and why use them?
**A:** Custom hooks are **functions starting with `use`** that call other hooks. They extract reusable stateful logic from components (not UI).

```jsx
// useFetch — reusable data fetching
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetch(url)
      .then(r => r.json())
      .then(d => { if (!cancelled) setData(d); })
      .catch(e => { if (!cancelled) setError(e); })
      .finally(() => { if (!cancelled) setLoading(false); });
    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}
```

**Rules of Hooks apply inside custom hooks** — they must be called at the top level, not conditionally.

---

## 12. React APIs

### React APIs — Deep Dive

---

### 1. `React.createElement`

**What it is:** The function that JSX compiles down to. Every `<Tag />` in JSX becomes `React.createElement(Tag, props, ...children)`.

```jsx
// JSX (what you write)
const el = <button className="btn" onClick={handleClick}>Save</button>;

// What Babel compiles it to (old transform)
const el = React.createElement(
  'button',
  { className: 'btn', onClick: handleClick },
  'Save'
);

// New JSX transform (React 17+) — you never call this directly
import { jsx as _jsx } from 'react/jsx-runtime';
const el = _jsx('button', { className: 'btn', onClick: handleClick, children: 'Save' });
```

**Real-world use case:** Rarely called directly — but understanding it matters for:
- Writing **render prop libraries** that produce elements programmatically
- **Dynamic component factories** (e.g., building a form field renderer that takes a `type` config and calls `React.createElement(componentMap[type], props)`)

```jsx
// Dynamic component renderer (no JSX)
const componentMap = { text: TextInput, select: SelectInput, checkbox: CheckboxInput };

function DynamicField({ type, ...props }) {
  return React.createElement(componentMap[type], props);
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Full programmatic control over element creation | Verbose — JSX is almost always cleaner |
| Works in environments where JSX transpilation is unavailable | Hard to read/maintain for complex trees |
| Foundation for understanding how React works internally | Manual children chaining is error-prone |

---

### 2. `React.cloneElement`

**What it is:** Clones a React element and optionally overrides its props or children.

```jsx
// Signature
React.cloneElement(element, [extraProps], [...children])
```

```jsx
// Real-world: Injecting extra props into children in a layout component
function Toolbar({ children }) {
  return (
    <div className="toolbar">
      {React.Children.map(children, child =>
        React.cloneElement(child, { size: 'small', variant: 'ghost' })
      )}
    </div>
  );
}

// Usage
<Toolbar>
  <Button>Save</Button>      {/* receives size="small" variant="ghost" automatically */}
  <Button>Cancel</Button>    {/* same */}
</Toolbar>
```

**Real-world use case:** Design systems that **inject shared props** into child components without requiring each child to accept/pass the prop explicitly. E.g., a `<ButtonGroup>` that injects `size` and `spacing` into all `<Button>` children.

```jsx
// Tooltip wrapper that injects aria attributes
function TooltipWrapper({ children, tooltipText }) {
  return React.cloneElement(children, {
    'aria-label': tooltipText,
    onMouseEnter: (e) => { showTooltip(tooltipText); children.props.onMouseEnter?.(e); },
  });
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Lets parents enhance children without children knowing | Can create "magic" implicit behaviour — hard to debug |
| Useful in design system layout components | Prop collisions: cloned props may overwrite child's own props |
| Preserves `key` and `ref` from original element | **Deprecated pattern** — React team now recommends render props or Context instead |
| No need to modify child components | Children must be a single valid React element |

> ⚠️ **Note:** The React team considers `cloneElement` an "escape hatch" and recommends avoiding it in new code. Use Context or render props instead.

---

### 3. `React.Children`

**What it is:** A set of utilities for safely working with `props.children`, which can be a single element, array, string, null, or fragment.

```jsx
React.Children.map(children, fn)       // map over each child
React.Children.forEach(children, fn)   // iterate without returning array
React.Children.count(children)         // count children (handles null/undefined)
React.Children.only(children)          // assert exactly one child (throws if not)
React.Children.toArray(children)       // flatten children to a stable array with keys
```

**Real-world use case — Tab component:**
```jsx
function Tabs({ children, activeIndex = 0 }) {
  // Count children to validate
  const count = React.Children.count(children);
  if (count === 0) return null;

  return (
    <div>
      <nav>
        {React.Children.map(children, (child, i) => (
          <button
            key={i}
            className={i === activeIndex ? 'active' : ''}
            onClick={() => setActive(i)}
          >
            {child.props.label}
          </button>
        ))}
      </nav>
      <div className="panel">
        {/* Render only active panel */}
        {React.Children.toArray(children)[activeIndex]}
      </div>
    </div>
  );
}

// Usage
<Tabs>
  <Tab label="Profile">Profile content</Tab>
  <Tab label="Settings">Settings content</Tab>
</Tabs>
```

**Real-world use case — `React.Children.only` for strictness:**
```jsx
function Tooltip({ children, text }) {
  // Enforce exactly one child (throws if 0 or 2+)
  const child = React.Children.only(children);
  return (
    <span title={text}>
      {child}
    </span>
  );
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Safely handles any type of children (null, array, fragment, single) | **Deprecated in favour of array methods** — React team discourages `React.Children` |
| `count` and `only` add runtime validation | Doesn't handle nested fragments well |
| `toArray` gives stable keys for animations/sorting | API is verbose compared to normal array ops |
| Widely used in design systems (older pattern) | Encourages tight parent-child coupling |

> ⚠️ React team recommendation: Prefer explicit arrays or Compound Pattern (Context) over `React.Children`.

---

### 4. `React.createContext`

**What it is:** Creates a Context object for sharing values deeply through the component tree without prop drilling.

```jsx
// 1. Create the context
const ThemeContext = React.createContext('light'); // 'light' is the default value

// 2. Provide a value
function App() {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

// 3. Consume anywhere in the tree
function Button({ children }) {
  const { theme } = useContext(ThemeContext);
  return <button className={`btn-${theme}`}>{children}</button>;
}
```

**Real-world use case — Auth context pattern:**
```jsx
// auth-context.js
const AuthContext = React.createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = async (credentials) => {
    const user = await api.login(credentials);
    setUser(user);
  };

  const logout = () => {
    api.logout();
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook — better DX + error guard
export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}

// Usage anywhere
function Header() {
  const { user, logout } = useAuth();
  return <div>{user.name} <button onClick={logout}>Logout</button></div>;
}
```

**Performance fix — split contexts:**
```jsx
// ❌ Bad: changing user re-renders ALL consumers including those only using theme
const AppContext = React.createContext({ user, theme, locale });

// ✅ Good: split into separate stable vs dynamic contexts
const ThemeContext = React.createContext('light');   // rarely changes
const UserContext = React.createContext(null);       // user actions
const LocaleContext = React.createContext('en');
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Eliminates prop drilling for deeply nested data | All consumers re-render when context value changes |
| Works great for infrequently-changing global data (theme, locale, auth) | No built-in selector — you can't subscribe to part of the context |
| Built into React — no external library needed | Can encourage overuse, making component reuse harder |
| Works with SSR naturally | Deep context nesting can make tracing data flow difficult |

---

### 5. `React.lazy`

**What it is:** Dynamically imports a component, enabling **code splitting**. The component's JS bundle is only loaded when it's first rendered.

```jsx
// Syntax
const MyComponent = React.lazy(() => import('./MyComponent'));
// Must be wrapped in <Suspense> to handle the loading state
```

**Real-world use case — Route-based code splitting (most impactful):**
```jsx
import { Suspense, lazy } from 'react';
import { Routes, Route } from 'react-router-dom';

// Each page is a separate JS chunk — only loaded when user navigates there
const HomePage    = lazy(() => import('./pages/Home'));
const Dashboard   = lazy(() => import('./pages/Dashboard'));
const AdminPanel  = lazy(() => import('./pages/AdminPanel'));
const UserProfile = lazy(() => import('./pages/UserProfile'));

function App() {
  return (
    <Routes>
      <Route path="/" element={
        <Suspense fallback={<PageSkeleton />}><HomePage /></Suspense>
      } />
      <Route path="/dashboard" element={
        <Suspense fallback={<PageSkeleton />}><Dashboard /></Suspense>
      } />
      <Route path="/admin" element={
        <Suspense fallback={<PageSkeleton />}><AdminPanel /></Suspense>
      } />
    </Routes>
  );
}
```

**Real-world use case — Heavy modal/dialog:**
```jsx
// Don't load the rich text editor JS until the user opens the modal
const RichTextEditor = lazy(() => import('./RichTextEditor')); // ~150KB

function PostEditor() {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(true)}>Edit Post</button>
      {open && (
        <Suspense fallback={<Spinner />}>
          <RichTextEditor />
        </Suspense>
      )}
    </>
  );
}
```

**Prefetch on hover (proactive loading):**
```jsx
const Dashboard = lazy(() => import('./Dashboard'));

function NavLink() {
  // Start loading the chunk when user hovers, before they click
  const prefetch = () => import('./Dashboard');
  return <a href="/dashboard" onMouseEnter={prefetch}>Dashboard</a>;
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Reduces initial bundle size → faster FCP/TTI | Requires `<Suspense>` boundary — always |
| Only loads code when needed | Named exports not directly supported (must re-export as default) |
| Works with Webpack and Vite naturally | Loading state may cause layout shift if not designed for it |
| Can combine with `webpackPrefetch`/`webpackPreload` hints | Server Components (RSC) don't need lazy — they have a different model |

---

### 6. `React.Suspense`

**What it is:** A boundary component that shows a fallback UI while waiting for something async (lazy component, data fetching via `use()`, or streaming SSR).

```jsx
<Suspense fallback={<LoadingSpinner />}>
  <SlowComponent />  {/* can "suspend" — throw a Promise */}
</Suspense>
```

**Real-world use case — Nested Suspense boundaries for granular loading:**
```jsx
function ProductPage({ productId }) {
  return (
    <div>
      <h1>Product Page</h1>

      {/* Header loads fast — its own boundary */}
      <Suspense fallback={<HeaderSkeleton />}>
        <ProductHeader id={productId} />
      </Suspense>

      <div className="content">
        {/* Reviews can load slower — isolated boundary */}
        <Suspense fallback={<ReviewsSkeleton />}>
          <ProductReviews id={productId} />
        </Suspense>

        {/* Recommendations load last — don't block above */}
        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations id={productId} />
        </Suspense>
      </div>
    </div>
  );
}
```

**Real-world use case — Streaming SSR (Next.js App Router):**
```jsx
// Next.js App Router — server streams HTML progressively
export default function Page() {
  return (
    <main>
      <h1>Dashboard</h1>
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />  {/* async server component, streamed when ready */}
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />  {/* streamed independently */}
      </Suspense>
    </main>
  );
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Declarative loading states — no `isLoading` flags in every component | Requires data-fetching libraries to support Suspense (React Query v5, Relay, Next.js RSC) |
| Enables streaming SSR — sends HTML progressively | Hard to implement correctly in custom data-fetching without throwing Promises |
| Boundary isolation — one slow component doesn't block others | `fallback` renders synchronously — complex fallbacks can be expensive |
| Integrates with Error Boundaries for full error + loading handling | Not yet supported for arbitrary async ops without a library |

---

### 7. `React.memo`

**What it is:** A Higher-Order Component that wraps a component and **skips re-rendering** if props haven't changed (shallow comparison).

```jsx
// Basic usage
const UserCard = React.memo(function UserCard({ name, avatar, role }) {
  console.log('UserCard rendered');
  return (
    <div className="card">
      <img src={avatar} alt={name} />
      <h3>{name}</h3>
      <span>{role}</span>
    </div>
  );
});

// Custom comparator (deep equality or subset check)
const Chart = React.memo(
  function Chart({ data, width, height }) { /* expensive render */ },
  (prevProps, nextProps) => {
    // Return true to SKIP re-render (props considered equal)
    return prevProps.data === nextProps.data &&
           prevProps.width === nextProps.width;
    // Ignored: height — doesn't trigger re-render even if changed
  }
);
```

**Real-world use case — Large list with many items:**
```jsx
// Without memo: all 1000 items re-render when parent state changes
// With memo: only items whose props changed re-render
const ProductCard = React.memo(function ProductCard({ product, onAddToCart }) {
  return (
    <div>
      <h4>{product.name}</h4>
      <p>${product.price}</p>
      <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
    </div>
  );
});

function ProductList({ products, cartCount }) {
  // useCallback is REQUIRED here — otherwise new function reference each render
  // defeats React.memo
  const handleAddToCart = useCallback((id) => {
    dispatch({ type: 'ADD', id });
  }, [dispatch]);

  return products.map(p =>
    <ProductCard key={p.id} product={p} onAddToCart={handleAddToCart} />
  );
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Prevents expensive re-renders for stable components | Shallow comparison — objects/arrays still cause re-renders if reference changes |
| Works with custom comparator for fine-grained control | The comparison itself has a cost — may not be worth it for cheap components |
| Easy to add — just wrap the component | Easy to misuse: forgetting `useCallback` for function props defeats the purpose |
| Effective pattern for list items | Gives false sense of security — profile first, don't premature-optimise |

---

### 8. `React.forwardRef`

**What it is:** Allows a parent component to pass a `ref` to a child's internal DOM node (or class instance).

**Why needed:** `ref` is special in React — it's not passed as a regular prop. Without `forwardRef`, a child component cannot receive a ref from its parent.

```jsx
// Basic pattern
const TextInput = React.forwardRef(function TextInput({ label, ...props }, ref) {
  return (
    <div className="field">
      <label>{label}</label>
      <input ref={ref} {...props} />  {/* ref attached to the actual DOM input */}
    </div>
  );
});

// Parent: gets direct access to the <input> DOM node
function LoginForm() {
  const emailRef = useRef();

  useEffect(() => {
    emailRef.current.focus(); // auto-focus on mount
  }, []);

  return <TextInput ref={emailRef} label="Email" type="email" />;
}
```

**Real-world use case — Design System Input with `useImperativeHandle`:**
```jsx
const OTPInput = React.forwardRef(function OTPInput(props, ref) {
  const inputRef = useRef();

  // Expose a controlled API instead of the raw DOM node
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ''; },
    getValue: () => inputRef.current.value,
  }));

  return <input ref={inputRef} maxLength={6} pattern="[0-9]*" {...props} />;
});

// Parent uses the clean API, not the raw DOM
function VerifyPage() {
  const otpRef = useRef();
  const handleResend = () => {
    otpRef.current.clear();
    otpRef.current.focus();
  };
  return (
    <>
      <OTPInput ref={otpRef} />
      <button onClick={handleResend}>Resend & Clear</button>
    </>
  );
}
```

**React 19 update:** `forwardRef` is no longer needed — `ref` is passed as a regular prop:
```jsx
// React 19 — ref is just a prop
function TextInput({ label, ref, ...props }) {
  return <input ref={ref} {...props} />;
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Allows parent to imperatively control child DOM (focus, scroll, animate) | Breaks the declarative/data-down model of React |
| Required for custom input components in design libraries | Exposes internals — creates tight coupling between parent and child |
| Works seamlessly with `useImperativeHandle` for clean APIs | Overuse is an anti-pattern — prefer state/props + callbacks |
| Transparent to the component tree | `forwardRef` wrapper adds extra component layer in DevTools |

---

### 9. `React.createPortal`

**What it is:** Renders children into a **different DOM node** than the parent's DOM subtree, while keeping them in the React tree (so events and context still work).

```jsx
// from 'react-dom'
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;

  // Renders into document.body, not inside the component's parent <div>
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-box" onClick={e => e.stopPropagation()}>
        <button className="close-btn" onClick={onClose}>×</button>
        {children}
      </div>
    </div>,
    document.body
  );
}

// Usage — even though <Modal> is inside a deeply nested component,
// the DOM renders at document.body level
function CheckoutForm() {
  const [showConfirm, setShowConfirm] = useState(false);
  return (
    <div style={{ overflow: 'hidden' }}> {/* overflow wouldn't clip the modal! */}
      <button onClick={() => setShowConfirm(true)}>Confirm Order</button>
      <Modal isOpen={showConfirm} onClose={() => setShowConfirm(false)}>
        <h2>Are you sure?</h2>
        <button onClick={placeOrder}>Yes, place order</button>
      </Modal>
    </div>
  );
}
```

**Real-world use cases:**
- **Modals/Dialogs** — escape `overflow: hidden` and stacking context
- **Tooltips & Popovers** — render near `document.body` to avoid z-index issues
- **Toast notifications** — always at top level of DOM
- **Drag ghost elements** — render outside their container during drag

**Key behaviour: Events bubble through React tree, not DOM tree:**
```jsx
// Even though Portal renders in <body>, a click inside it bubbles to
// the React parent component — context, handlers, all work normally
function Wrapper() {
  return (
    <div onClick={() => console.log('React parent clicked!')}>
      <Modal>
        <button>Click me</button>  {/* click bubbles to Wrapper's handler */}
      </Modal>
    </div>
  );
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Escapes CSS stacking context (z-index, overflow) | Can make accessibility (focus trapping) harder to implement correctly |
| React tree structure preserved — context & events work | DOM hierarchy mismatch can confuse debugging tools |
| Clean solution for modals, tooltips, toasts | Need to manually manage `document.body` portal mount node |
| No library needed — built into React DOM | Multiple portals can conflict on `document.body` (z-index wars) |

---

### 10. `React.startTransition`

**What it is:** Marks a state update as **non-urgent (low priority)**. React will deprioritise it and let urgent updates (user input, clicks) happen first. It's the module-level version — `useTransition` is the hook that also provides `isPending`.

```jsx
import { startTransition } from 'react';

// Without transition — heavy computation blocks typing
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  function handleChange(e) {
    setQuery(e.target.value);            // urgent: update input immediately
    startTransition(() => {
      setResults(filterData(e.target.value)); // non-urgent: can be interrupted
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </>
  );
}
```

**Real-world use case — Tab switching with heavy content:**
```jsx
function TabContainer({ tabs }) {
  const [activeTab, setActiveTab] = useState(0);
  const [isPending, startTransition] = useTransition();

  function switchTab(index) {
    startTransition(() => {
      setActiveTab(index); // rendering the tab content is non-urgent
    });
  }

  return (
    <div>
      <nav>
        {tabs.map((tab, i) => (
          <button
            key={i}
            onClick={() => switchTab(i)}
            className={activeTab === i ? 'active' : ''}
          >
            {tab.label}
            {isPending && activeTab === i && <Spinner size="xs" />}
          </button>
        ))}
      </nav>
      <div className={isPending ? 'opacity-50' : ''}>
        {tabs[activeTab].content}
      </div>
    </div>
  );
}
```

**`startTransition` (module) vs `useTransition` (hook):**
```jsx
// Use startTransition (module) when you don't need isPending
import { startTransition } from 'react';
startTransition(() => setRoute(newRoute));

// Use useTransition (hook) when you need to show a loading indicator
const [isPending, startTransition] = useTransition();
{isPending && <Spinner />}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Keeps UI responsive during heavy state transitions | Only works with React's state — not for external side effects |
| `isPending` lets you show a subtle loading indicator | Transition updates may be deferred indefinitely if urgent work keeps coming |
| No external library needed — built into React 18+ | Can make debugging harder — deferred updates are invisible in stack traces |
| Works with Suspense — suspending transitions show old content, not fallback | Misuse (marking urgent updates as transitions) hurts responsiveness |

---

### 11. `flushSync` (from `react-dom`)

**What it is:** Forces React to **synchronously flush** all pending state updates and re-render immediately. Opts out of automatic batching for a specific update.

```jsx
import { flushSync } from 'react-dom';
```

**Real-world use case — Scroll to a newly added item:**
```jsx
function ChatWindow({ messages }) {
  const bottomRef = useRef();
  const [msgs, setMsgs] = useState(messages);

  function sendMessage(text) {
    // PROBLEM without flushSync:
    // setMsgs → batched (DOM not updated yet)
    // scrollIntoView → scrolls before new message is in DOM → doesn't scroll to new msg

    // SOLUTION with flushSync:
    flushSync(() => {
      setMsgs(prev => [...prev, { text, sender: 'me', id: Date.now() }]);
    }); // DOM is updated SYNCHRONOUSLY by this point

    bottomRef.current?.scrollIntoView({ behavior: 'smooth' }); // now works ✓
  }

  return (
    <div className="chat">
      {msgs.map(m => <Message key={m.id} msg={m} />)}
      <div ref={bottomRef} />
    </div>
  );
}
```

**Real-world use case — Third-party library integration:**
```jsx
// When a non-React event (e.g., from a canvas library) needs to sync React state
canvas.on('objectSelected', (obj) => {
  flushSync(() => {
    setSelectedObject(obj); // force React to render before canvas continues
  });
  // React has re-rendered — safe to read updated DOM
  updateCanvasOverlay();
});
```

**Real-world use case — Print / Screenshot:**
```jsx
function PrintButton() {
  function handlePrint() {
    flushSync(() => {
      setShowPrintView(true); // force React to render the print layout
    });
    window.print(); // DOM is ready for print
    setShowPrintView(false);
  }
  return <button onClick={handlePrint}>Print</button>;
}
```

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Guarantees DOM is updated before the next line — necessary for DOM measurements or scroll | Defeats automatic batching — causes extra re-renders |
| Essential for integrating with non-React code that reads the DOM | Synchronous = blocks the browser — use sparingly |
| Fixes subtle scroll-after-add bugs | Can hurt performance if used frequently |
| Works as an escape hatch from React's async update model | Almost never needed in pure React code — signals a design issue |

---

## 13. Forms

### Q: How do you optimise large forms?
**A:**
1. **Use uncontrolled components** via `React Hook Form` (reads DOM values on submit, no re-render per keystroke)
2. **Field-level subscriptions** — RHF only re-renders the changed field
3. **Debounce** validation triggers
4. **Virtualize** very long forms (rare)
5. **Split into steps/pages** with local state per step

### Q: How does React Hook Form (RHF) work internally?
**A:** RHF uses an **uncontrolled** paradigm:
- Registers inputs via `ref` (native DOM inputs hold their own values)
- On submit, reads values from refs — only **one render** for the whole form
- Uses a **subscription model** — only components that are subscribed to a field's state (e.g., validation errors) re-render when that field changes
- Uses `useForm()` to return stable `register`, `handleSubmit`, `formState` objects

### Q: Why are controlled inputs expensive?
**A:** Every keystroke:
1. User types → `onChange` fires → `setState` called → component re-renders
2. React reconciles the VDOM → DOM updated
For large forms with many fields, this causes many re-renders per second. With uncontrolled inputs (RHF, native `<form>`), no re-renders occur during typing.

---

## 14. React Portals

### Q: What are React Portals?
**A:** `ReactDOM.createPortal(children, domNode)` renders children into a **different DOM node** than the parent component's hierarchy — while keeping them in the **React component tree**.

```jsx
function Modal({ children }) {
  return ReactDOM.createPortal(
    <div className="modal">{children}</div>,
    document.body // renders directly into body, not in parent's div
  );
}
```

**Why:** Modals, tooltips, dropdowns need to escape parent `overflow: hidden` or `z-index` stacking contexts, but still need React event bubbling, context, and lifecycle.

**Event propagation:** Events still bubble through the React component tree (not the DOM tree), so a click inside a portal bubbles to the portal's React parent.

---

## 15. Rendering Patterns

### Q: Higher-Order Component (HOC)
**A:** A function that takes a component and returns a new, enhanced component.
```jsx
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const { isAuthenticated } = useAuth();
    if (!isAuthenticated) return <Redirect to="/login" />;
    return <WrappedComponent {...props} />;
  };
}
const ProtectedDashboard = withAuth(Dashboard);
```
**Problems**: Wrapper hell, naming collisions, static methods not forwarded.

### Q: Render Props
**A:** A component accepts a **function as a prop** and calls it to determine what to render.
```jsx
function Mouse({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={e => setPos({ x: e.clientX, y: e.clientY })}>
      {render(pos)}
    </div>
  );
}
// Usage:
<Mouse render={({ x, y }) => <p>Mouse at {x},{y}</p>} />
```

### Q: Compound Pattern
**A:** Components that work together by sharing implicit state via Context. Used in design systems.
```jsx
// Usage:
<Accordion>
  <Accordion.Item>
    <Accordion.Header>Title</Accordion.Header>
    <Accordion.Body>Content</Accordion.Body>
  </Accordion.Item>
</Accordion>
```
Internally, `Accordion` provides a Context that `Item`, `Header`, and `Body` consume.

### Q: Controlled State Pattern
**A:** Making a component work in both **controlled** (parent owns state) and **uncontrolled** (component owns state) modes. Also called "state hoisting with fallback".

```jsx
function Toggle({ isOn: controlledIsOn, onChange }) {
  const isControlled = controlledIsOn !== undefined;
  const [internalIsOn, setInternalIsOn] = useState(false);
  const isOn = isControlled ? controlledIsOn : internalIsOn;
  const toggle = () => {
    if (!isControlled) setInternalIsOn(v => !v);
    onChange?.(!isOn);
  };
  return <button onClick={toggle}>{isOn ? 'ON' : 'OFF'}</button>;
}
```

### Q: State Machines in React
**A:** Managing complex UI state with explicit states and transitions (using `XState` or custom reducers):
```jsx
// States: idle → loading → success/error
const [state, send] = useMachine(fetchMachine);
// Prevents impossible states (e.g., success + error simultaneously)
```

### Q: Headless UI
**A:** Components that provide **behaviour and accessibility** with **zero UI** (no styles). Consumers supply their own rendering. Examples: Radix UI, Headless UI by Tailwind. Pattern: hook that returns state + event handlers, component renders nothing visible.

---

## 16. Advanced Rendering

### Q: What is Hydration?
**A:** When an SSR-rendered HTML page reaches the browser, React **attaches event listeners and state** to the existing DOM nodes — without re-creating them. This is called hydration.

**Hydration mismatch**: If client-rendered output differs from server HTML, React discards server HTML and fully re-renders (performance cost + layout shift). Common cause: `typeof window` checks, random values, dates.

**React 18 — Selective Hydration (Concurrent)**: Using `<Suspense>`, React can hydrate parts of the page independently. User interactions can prioritise hydration of specific areas.

### Q: Difference between CSR, SSR, SSG, ISR?

| Rendering | When HTML built | Data freshness | Use case |
|---|---|---|---|
| **CSR** (Client-Side Rendering) | In browser, after JS loads | Real-time | Dashboards, authenticated apps |
| **SSR** (Server-Side Rendering) | On server, per request | Fresh on each request | News, product pages with dynamic SEO |
| **SSG** (Static Site Generation) | At build time | Stale until rebuild | Blogs, docs, marketing pages |
| **ISR** (Incremental Static Regeneration) | At build + revalidated on request | Configurable TTL | E-commerce listings, frequently-updated static content |

---

## 17. Concurrency

### Q: What is Concurrent Rendering?
**A:** React 18's ability to **prepare multiple versions of UI simultaneously** and interrupt, pause, and resume rendering work. It does NOT mean multi-threading (JS is single-threaded). It means React can start rendering, pause to handle a higher-priority event, then resume.

### Q: `startTransition` and `useTransition`
**A:** Mark a state update as **non-urgent** (low priority). Urgent updates (typing, clicking) are not blocked. The transition update is deferred.

```jsx
const [isPending, startTransition] = useTransition();
function onInput(e) {
  setInputValue(e.target.value); // urgent — updates immediately
  startTransition(() => {
    setSearchResults(filter(data, e.target.value)); // non-urgent
  });
}
// isPending is true while transition is in flight — show spinner
```

### Q: `useDeferredValue`
**A:** Returns a **deferred copy** of a value that may lag behind the original. Useful when you can't control the setState call (e.g., value comes from props):
```jsx
const deferredQuery = useDeferredValue(query);
const results = useMemo(() => expensiveFilter(data, deferredQuery), [deferredQuery]);
```

### Q: Automatic Batching (React 18)
**A:** React 18 automatically batches **all** state updates (even inside `setTimeout`, `Promises`, native event handlers) into a single re-render. Previously, only updates inside React event handlers were batched.

```jsx
// React 17 — 2 renders
setTimeout(() => {
  setA(1); // render
  setB(2); // render
}, 1000);

// React 18 — 1 render (automatic batching)
setTimeout(() => {
  setA(1);
  setB(2); // batched!
}, 1000);

// Opt out with flushSync:
import { flushSync } from 'react-dom';
flushSync(() => setA(1)); // forces synchronous render
```

### Q: Why does `StrictMode` render components twice (in development)?
**A:** In React 18 development mode, `StrictMode` **double-invokes** render functions and effect setup/cleanup to:
- Detect **side effects in render** (render should be pure)
- Detect **non-idempotent effects** (missing cleanup)
- Simulate `offscreen` component (future feature) that may mount/unmount components
- Helps catch bugs early in development, not in production

---

## 18. Error Handling

### Q: What are Error Boundaries?
**A:** Class components that catch **JavaScript errors** in their child tree during rendering, lifecycle methods, and constructors. They use `componentDidCatch` and `getDerivedStateFromError`.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  componentDidCatch(error, info) {
    logErrorToService(error, info.componentStack);
  }
  render() {
    if (this.state.hasError) return <h1>Something went wrong.</h1>;
    return this.props.children;
  }
}
```

**Limitations**: Error boundaries do NOT catch:
- Event handler errors (use try/catch)
- Async errors (use try/catch in async functions)
- Server-side rendering errors
- Errors in the error boundary itself

**React 19**: `react-error-boundary` and soon native function component support.

---

## 19. Suspense

### Q: How does Suspense work internally?
**A:** When a component **throws a Promise** (the `use()` pattern or `React.lazy`), React catches it, renders the nearest `<Suspense fallback>`, and **waits for the Promise to resolve**. When resolved, React retries rendering the suspended subtree.

```jsx
// React.lazy example
const UserProfile = React.lazy(() => import('./UserProfile'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile /> {/* may throw a Promise while loading */}
    </Suspense>
  );
}
```

**React 18 streaming SSR + Suspense:**
- Server streams HTML immediately for ready parts
- Suspended parts send a spinner placeholder
- Once data is ready, server streams the real HTML + JS to swap in the content (selectjve hydration)

---

## 20. Performance

### Q: How to profile a React app?
**A:**
1. **React DevTools Profiler** — Record renders, flamegraph, ranked chart. Shows which components rendered and for how long.
2. **Chrome Performance tab** — CPU/memory timeline, flame chart
3. **Lighthouse** — FCP, LCP, CLS scores
4. **`<Profiler>` API** — Programmatic per-component timing:
   ```jsx
   <Profiler id="List" onRender={(id, phase, actualDuration) => console.log(actualDuration)}>
     <MyList />
   </Profiler>
   ```

### Q: Why do memory leaks happen in React?
Common causes:
1. **Subscriptions not cleaned up**: event listeners, WebSocket, RxJS added in `useEffect` without cleanup
2. **Async state updates after unmount** (fixed with cancelled flag or AbortController)
3. **Closures holding large objects**
4. **Refs holding removed DOM nodes**

### Q: How do stale closures cause subtle bugs?
**A:** (See useEffect section above.) Key manifestation in intervals and timeouts:
```jsx
// Bug: logs 0 every second, not incrementing count
useEffect(() => {
  setInterval(() => console.log(count), 1000);
}, []); // captures count=0 at mount
```
Fix: use `useRef` to hold latest value, or functional updaters, or `useEffectEvent`.

### Q: How does React DevTools Profiler work?
**A:** In development mode (or profiling build), React wraps render calls with timing measurements using `performance.now()`. DevTools subscribes to these measurements via `__REACT_DEVTOOLS_GLOBAL_HOOK__`. The Profiler shows a **flamegraph** (time × depth) and **ranked chart** (components by render time).

---

## 21. Security

### Q: What is XSS in React context?
**A:** Cross-Site Scripting — injecting malicious scripts into the page. React **automatically escapes** all string values rendered via JSX, rendering them as text, not HTML.

### Q: `dangerouslySetInnerHTML` — when and risks?
**A:** Directly sets the inner HTML of a DOM node. Bypasses React's escaping — **vulnerable to XSS** if used with user-controlled data.
```jsx
// DANGEROUS if content is user input:
<div dangerouslySetInnerHTML={{ __html: userContent }} />
// SAFE only with sanitized content:
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />
```

### Q: How does React escape values?
**A:** JSX string expressions are rendered using `textContent` or `createTextNode` (not `innerHTML`). React encodes `<`, `>`, `&`, `"` as HTML entities in string context.

### Q: How to prevent injection attacks?
1. Never use `dangerouslySetInnerHTML` with unsanitised input (use DOMPurify)
2. Validate and sanitise on the server
3. Avoid concatenating user input into URLs without encoding
4. Use `Content-Security-Policy` headers
5. Avoid `eval()` with user data

### Q: CSP (Content Security Policy) in React apps?
**A:** CSP is an HTTP header that restricts which scripts, styles, and resources can load. For React:
- Avoid inline `style` attributes or inline scripts (CSP blocks them)
- Use nonce-based inline scripts if needed
- `script-src 'self'` — only load scripts from your domain
- CSS-in-JS libraries (styled-components) need nonce or `unsafe-inline` (weakens CSP)

---

## 22. React Router

### Q: Key React Router v6 concepts

| Concept | Description |
|---|---|
| `<BrowserRouter>` | HTML5 history API router |
| `<Routes>` / `<Route>` | Declarative route matching |
| `useNavigate()` | Programmatic navigation |
| `useParams()` | Access URL parameters |
| `useSearchParams()` | Read/write query string |
| `<Outlet>` | Render child routes (nested routing) |
| `<Link>` | Client-side navigation (no full page reload) |
| Loaders/Actions (v6.4+) | Data fetching & mutations co-located with routes |

### Q: How does client-side routing work?
**A:** React Router intercepts link clicks, updates the browser's URL using `history.pushState()` (no server request), and renders the matching components. The server must serve `index.html` for all routes (catch-all config).

---

## 23. Bundling: Webpack / Vite

### Q: Webpack vs Vite

| | Webpack | Vite |
|---|---|---|
| Dev server | Bundles everything first | ESM native — serves files as-is |
| HMR speed | Slower (re-bundles module graph) | Near-instant (only transforms changed file) |
| Production build | Mature, highly configurable | Rollup under the hood |
| Config complexity | High (loaders, plugins) | Low (sensible defaults) |
| Tree shaking | Yes | Yes (Rollup) |

### Q: How does Webpack build the dependency graph?
**A:** Starts from the **entry point** (`index.js`), recursively follows `import`/`require` statements, builds a dependency graph, applies **loaders** (transform files: TypeScript → JS, CSS → JS module), and outputs **chunks** (bundles).

---

## 24. Code Splitting

### Q: What is code splitting and why?
**A:** Splitting the JS bundle into multiple smaller chunks loaded **on demand** instead of all at once. Reduces initial bundle size → faster First Contentful Paint.

### Q: How to code split in React?
```jsx
// Dynamic import + React.lazy
const Dashboard = React.lazy(() => import('./Dashboard'));

// Wrap in Suspense
<Suspense fallback={<Spinner />}>
  <Dashboard />
</Suspense>
```

**Route-based splitting** (most impactful):
```jsx
const Home = React.lazy(() => import('./pages/Home'));
const About = React.lazy(() => import('./pages/About'));

<Routes>
  <Route path="/" element={<Suspense fallback={<Loader />}><Home /></Suspense>} />
  <Route path="/about" element={<Suspense fallback={<Loader />}><About /></Suspense>} />
</Routes>
```

**Webpack magic comments** (prefetch/preload):
```js
const Dashboard = React.lazy(() => import(/* webpackPrefetch: true */ './Dashboard'));
```

---

## 25. Folder Structure

### Q: Feature-based vs Layer-based folder structure?

**Layer-based (traditional):**
```
src/
  components/
  hooks/
  services/
  store/
  utils/
```
Problem: Files for one feature are scattered across many folders. As app grows, difficult to know what's related.

**Feature-based (recommended for large apps):**
```
src/
  features/
    auth/
      components/
      hooks/
      api.ts
      store.ts
      index.ts  ← public API
    dashboard/
    orders/
  shared/
    components/
    hooks/
    utils/
  app/
    routes.tsx
    store.ts
```

Benefits: High **cohesion** within features, low **coupling** across features. Features can be developed/deleted independently.

### Q: How to prevent tight coupling?
1. **Barrel exports** (`index.ts`) — only expose what should be public, hide internals
2. **Import rules** — features should not import from each other directly (use shared/)
3. **Dependency Inversion** — depend on shared interfaces, not concrete implementations
4. **No cross-feature imports** — enforce with ESLint `no-restricted-imports`
5. **Events / pub-sub** — for cross-feature communication instead of direct calls
6. **State colocation** — keep state inside the feature that owns it

---

## 26. Scenario-Based MANG Questions

These are the types of open-ended, system-design-meets-React questions asked at Meta, Amazon, Netflix, Google.

---

### Scenario 1: Infinite Scroll Feed (Meta/TikTok-style)
> **"Design an infinite scroll feed for a social media app with 100k+ posts. How would you handle performance?"**

**Answer framework:**
- **Virtualisation**: Use `react-window` or `react-virtual` — only render visible rows (20-30 items), not all 100k. DOM node count stays constant.
- **Data fetching**: Cursor-based pagination with `react-query`/`useSWRInfinite`. Fetch next page when user is 80% scrolled.
- **Image lazy loading**: `loading="lazy"` attribute or `IntersectionObserver`.
- **Debounce scroll events** or use `IntersectionObserver` on a sentinel div at the bottom.
- **Optimistic updates**: Like/comment without waiting for server response.
- **Stale-while-revalidate**: Show cached data immediately, update in background.
- **Memory management**: Evict items from VList as user scrolls away to prevent memory bloat.

---

### Scenario 2: Real-time Collaborative Editor (Google Docs-style)
> **"How would you build a collaborative text editor in React where multiple users see changes in real-time?"**

**Answer framework:**
- **WebSocket/SSE** for real-time updates between clients
- **Operational Transformation (OT)** or **CRDT** (Conflict-free Replicated Data Types, e.g., Yjs) to handle concurrent edits without conflicts
- State management: Use `useSyncExternalStore` to subscribe to the CRDT store
- **Optimistic UI**: Apply local change immediately, sync with server
- **Awareness**: Show other users' cursor positions (via Yjs awareness)
- **Undo/redo**: Managed by OT/CRDT stack
- React component: Use a library like `Slate.js` or `Tiptap` (built on ProseMirror) as the editor engine

---

### Scenario 3: Large Form with Conditional Fields
> **"You have a multi-step form with 50+ fields, complex validation, and conditional visibility. How do you architect this?"**

**Answer framework:**
- **React Hook Form** (uncontrolled) — no re-renders per keystroke
- **Schema-driven**: Define fields in a JSON schema, render dynamically — reduces code duplication
- **Step isolation**: Each step is a separate component with its own RHF context. Merge on final submit.
- **Conditional fields**: `watch()` from RHF to observe field values; conditionally register/unregister fields
- **Validation**: `zod` or `yup` schema per step; validate only current step fields on "Next"
- **State persistence**: `sessionStorage` to not lose data on page refresh
- **Accessibility**: `useId` for label associations, proper `aria-*` attributes

```jsx
const { watch, register } = useForm();
const country = watch('country');
{country === 'US' && <input {...register('ssn')} />}
```

---

### Scenario 4: Dashboard with 50 Live Charts
> **"You need to build a dashboard with 50 charts updating every second from a WebSocket feed. How do you prevent the UI from freezing?"**

**Answer framework:**
- **`useDeferredValue` / `useTransition`**: Wrap chart updates in transitions so user interactions (filters, tab switching) remain responsive
- **Throttle updates**: Don't render every WebSocket message; throttle to 10fps for charts
- **`useSyncExternalStore`**: Subscribe to a central WebSocket store, each chart only re-renders when its specific data key changes
- **Canvas-based charts**: Use `<canvas>` (via Chart.js or D3) instead of SVG/DOM for 50+ data-heavy charts — DOM has too many nodes
- **Web Workers**: Compute data transformations off the main thread via `Comlink`
- **Memoization**: `React.memo` + `useMemo` for chart components; only re-render if data reference changed
- **Selective subscription**: Each chart subscribes to only its data slice (Zustand selector pattern)

---

### Scenario 5: Migrating Class Components to Hooks
> **"Legacy codebase has 200 class components. What's your migration strategy?"**

**Answer framework:**
- **Don't big-bang rewrite** — incrementally migrate
- **Identify leaf components first** — no children, easier to convert
- **Colocate custom hooks first** — extract lifecycle logic to hooks, leave class component shell initially
- **Test coverage first** — write tests for existing behaviour before converting
- **forwardRef migration** — check for any `ref` usage before converting
- **`getDerivedStateFromProps`** → `useEffect` with care (avoid derived state anti-pattern)
- **Use Error Boundaries** — still need class components; wrap in a reusable `<ErrorBoundary>` HOC
- **ESLint rules**: `react-hooks/rules-of-hooks`, `react-hooks/exhaustive-deps` to enforce hook correctness

---

### Scenario 6: Performance Debugging — App is Slow
> **"Users report the app is sluggish when interacting with a data table. How do you debug and fix it?"**

**Answer framework:**
1. **Profile with React DevTools Profiler** — identify components with high `actualDuration`
2. **Check for unnecessary re-renders** — install `why-did-you-render` package
3. **Common culprits**:
   - Parent re-renders → all children re-render (add `React.memo`)
   - Unstable callback props (add `useCallback`)
   - Context value changes (split context, memoize value)
   - No virtualisation on large lists (add `react-window`)
4. **Check Chrome Performance tab**: Look for Long Tasks (>50ms)
5. **Memoize expensive calculations** with `useMemo`
6. **Debounce** frequent events (resize, scroll, input)
7. **Move expensive computation to Web Worker**

---

### Scenario 7: Authentication + Route Protection
> **"How do you implement route-level authentication in a SPA with token refresh?"**

**Answer framework:**
- **AuthContext**: `isAuthenticated`, `user`, `login()`, `logout()`, `refreshToken()`
- **Protected Route HOC**:
  ```jsx
  function ProtectedRoute({ children }) {
    const { isAuthenticated, loading } = useAuth();
    if (loading) return <Spinner />;
    if (!isAuthenticated) return <Navigate to="/login" replace />;
    return children;
  }
  ```
- **Token storage**: `httpOnly` cookie (XSS-safe) preferred over localStorage
- **Token refresh**: Axios interceptor auto-refreshes token on 401; queue pending requests during refresh
- **Silent refresh**: Refresh token before expiry using a timer
- **React Query integration**: Global error handler to catch 401 globally

---

### Scenario 8: Micro-Frontend Architecture
> **"How would you split a large React app into independently deployable micro-frontends?"**

**Answer framework:**
- **Module Federation** (Webpack 5): Each micro-frontend exposes React components; host app loads them at runtime
- **React 17 isolation**: Each MFE runs its own React instance (enabled by event delegation change in React 17)
- **Shared dependencies**: Mark `react` and `react-dom` as shared singletons in Module Federation config to avoid duplicate React instances
- **Routing**: Shell app handles top-level routes; each MFE handles its own sub-routes
- **Communication**: Custom events or shared Redux/Zustand store via Module Federation shared state
- **Independent deployments**: Each team deploys their remote URL; host fetches latest at runtime
- **Error boundaries between MFEs**: Isolate crashes to one MFE

---

### Scenario 9: Server Components vs Client Components
> **"A colleague says 'Use Server Components everywhere.' When would you push back?"**

**Answer framework:**
**Use Server Components (RSC) when:**
- Need direct DB/file system access
- Large dependencies that should not ship to client
- Pure data display, no interactivity

**Keep Client Components when:**
- Need event handlers (`onClick`, `onChange`)
- Use browser APIs (`localStorage`, `window`)
- Need `useState`, `useEffect`, `useContext`
- Real-time updates (WebSocket, polling)

**Push back because:**
- RSCs add architectural complexity (cannot import client hooks, cannot pass functions as props from server to client)
- Waterfall risk if not designed carefully
- Limited to Next.js App Router (not universal React)
- Debugging is harder (server logs vs browser DevTools)

---

### Scenario 10: State Management Decision
> **"When would you choose Redux, Zustand, Jotai, React Query, or Context for state management?"**

| Solution | When to use |
|---|---|
| **useState / Context** | Simple, local-ish state; < 5 consumers |
| **Zustand** | Global state that needs selectors, simple API, no boilerplate |
| **Redux Toolkit** | Complex state, time-travel debugging, large team, existing codebase |
| **Jotai / Recoil** | Atomic state, fine-grained subscriptions, derived state graphs |
| **React Query / SWR** | Server state (async data, caching, invalidation) — NOT React Query for UI state |
| **XState** | Complex state machines, multi-step flows, event-driven logic |

**Key insight for interviews**: Split server state (React Query) from client UI state (Zustand/Context). Don't use Redux for server data and don't use React Query for pure UI state.

---

*Document last updated: February 2026 | Covers React 18 + React 19 APIs*
