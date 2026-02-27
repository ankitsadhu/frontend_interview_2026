# React Performance Optimization — MANG Level Interview Guide

> Rendering · Memoization · Virtualization · Code Splitting · Profiling · Architecture · Cross-questions · Scenarios

---

## Table of Contents

1. [How React Rendering Works](#1-how-react-rendering-works)
2. [Unnecessary Re-renders — Root Cause](#2-unnecessary-re-renders)
3. [Memoization — `memo`, `useMemo`, `useCallback`](#3-memoization)
4. [State Design for Performance](#4-state-design-for-performance)
5. [Virtualization — Rendering Large Lists](#5-virtualization)
6. [Code Splitting & Lazy Loading](#6-code-splitting--lazy-loading)
7. [Image & Asset Optimization](#7-image--asset-optimization)
8. [Concurrent Features — `useTransition`, `useDeferredValue`](#8-concurrent-features)
9. [Profiling & Measurement](#9-profiling--measurement)
10. [Network & Data Fetching](#10-network--data-fetching)
11. [Bundle Size Optimization](#11-bundle-size-optimization)
12. [Architecture Patterns for Performance](#12-architecture-patterns)
13. [MANG Cross-Questions & Scenarios](#13-mang-cross-questions--scenarios)

---

## 1. How React Rendering Works

### Q: What triggers a React re-render?

A re-render happens when:

1. **`setState` / `useState` setter** is called (even with the same value — unless bailout kicks in)
2. **Parent re-renders** → all children re-render by default (even if their props haven't changed)
3. **Context value changes** → all consumers re-render
4. **`forceUpdate()`** (class components)

```
Trigger → Render Phase (VDOM diff) → Commit Phase (DOM update)
                                         ↑ only if diff found
```

### Q: What is the difference between "render" and "commit"?

| Render Phase | Commit Phase |
|---|---|
| Calls component functions | Applies DOM changes |
| Builds new VDOM tree | Runs `useLayoutEffect` |
| Diffs (reconciliation) | Then runs `useEffect` |
| **Pure, no side effects** | **Side effects go here** |
| Can be interrupted (Concurrent) | Synchronous, can't be interrupted |
| May run multiple times | Runs once per update |

### Cross-question: *"If a component re-renders but the VDOM diff finds no changes, does the DOM update?"*

**No.** React only touches the real DOM when the diff finds actual changes. But the component function still runs (CPU cost), VDOM is still diffed (CPU cost) — just no DOM mutation. This is why preventing unnecessary renders still matters.

---

## 2. Unnecessary Re-renders

### Q: Why do children re-render when the parent re-renders?

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <ExpensiveChild /> {/* re-renders every click! */}
    </div>
  );
}
```

React re-renders the entire subtree by default. `ExpensiveChild` re-renders not because its props changed, but because **React doesn't know they didn't** — it would need to compare all props, which is equally expensive.

### The 3 Ways to Prevent Unnecessary Re-renders

**1. `React.memo` — skip re-render if props are the same:**
```tsx
const ExpensiveChild = React.memo(function ExpensiveChild({ data }) {
  return <Chart data={data} />;
});
```

**2. Move state closer to where it's used (state colocation):**
```tsx
// ❌ State in parent forces child re-render
function Page() {
  const [query, setQuery] = useState('');
  return (
    <>
      <SearchInput query={query} setQuery={setQuery} />
      <ExpensiveProductGrid /> {/* re-renders on every keystroke! */}
    </>
  );
}

// ✅ State colocated — only SearchInput re-renders
function Page() {
  return (
    <>
      <SearchInput /> {/* query state lives INSIDE here */}
      <ExpensiveProductGrid />
    </>
  );
}
```

**3. Children as props pattern (composition):**
```tsx
// ✅ children are created by the GRANDPARENT, not re-created on Parent re-render
function ScrollTracker({ children }) {
  const [scrollY, setScrollY] = useState(0);
  useEffect(() => {
    const handler = () => setScrollY(window.scrollY);
    window.addEventListener('scroll', handler);
    return () => window.removeEventListener('scroll', handler);
  }, []);

  return <div data-scroll={scrollY}>{children}</div>;
  // children reference doesn't change → no re-render
}

// Usage
<ScrollTracker>
  <ExpensiveContent /> {/* doesn't re-render on scroll! */}
</ScrollTracker>
```

### Cross-question: *"Why does the `children` pattern prevent re-renders?"*

When `<ExpensiveContent />` is written in the parent's JSX, it becomes a `React.createElement` call at the parent level. That call returns a **stable reference** (same element object) — it doesn't change when `ScrollTracker` re-renders. React sees the same element reference → skips re-rendering the child.

---

## 3. Memoization

### Q: When to use `React.memo`?

```tsx
const MemoizedComponent = React.memo(MyComponent);
// Only re-renders if props change (shallow comparison)
```

| ✅ Use `React.memo` when | ❌ Don't use when |
|---|---|
| Component renders often with same props | Props change every render anyway |
| Component is expensive to render | Component is cheap (a `<div>` with text) |
| Parent re-renders frequently | Adds complexity with no measurable benefit |
| Used in a list with many items | Props include unstable objects/functions |

### Q: `useMemo` vs `useCallback` — what's the difference?

```tsx
// useMemo — memoize a COMPUTED VALUE
const sortedItems = useMemo(
  () => items.sort((a, b) => a.price - b.price), // expensive computation
  [items]
);

// useCallback — memoize a FUNCTION REFERENCE
const handleClick = useCallback(
  (id) => dispatch(removeItem(id)),
  [dispatch]
);
```

| `useMemo` | `useCallback` |
|---|---|
| Returns **computed value** | Returns **function reference** |
| For expensive calculations | For stable function references |
| Runs the function, caches result | Doesn't run the function, caches the function itself |
| `useMemo(() => fn(a, b), [a, b])` | `useCallback((x) => fn(x), [deps])` |
| `useCallback(fn, deps)` ≡ `useMemo(() => fn, deps)` | — |

### The Memoization Chain (all 3 must work together)

```tsx
// If ANY link breaks, memoization fails

// Link 1: Memoize the component
const Chart = React.memo(function Chart({ data, onHover }) {
  return /* expensive render */;
});

// Link 2: Memoize the data (useMemo)
const chartData = useMemo(() => processRawData(rawData), [rawData]);

// Link 3: Memoize the callback (useCallback)
const handleHover = useCallback((point) => {
  setTooltip(point);
}, []);

// All links intact → Chart only re-renders when rawData changes
<Chart data={chartData} onHover={handleHover} />
```

```tsx
// ❌ Broken chain — useMemo on data but inline callback
<Chart data={chartData} onHover={(point) => setTooltip(point)} />
// onHover is a NEW function every render → memo is useless!
```

### Cross-question: *"Does React.memo do deep comparison?"*

**No — shallow only.** It compares each prop with `Object.is()`.

```tsx
// ❌ Shallow fails on objects/arrays (new reference every render)
<MemoizedList items={items.filter(i => i.active)} /> // new array every render!

// ✅ Fix 1: useMemo
const activeItems = useMemo(() => items.filter(i => i.active), [items]);
<MemoizedList items={activeItems} />

// ✅ Fix 2: custom comparator
const MemoizedList = React.memo(List, (prevProps, nextProps) => {
  return prevProps.items.length === nextProps.items.length &&
    prevProps.items.every((item, i) => item.id === nextProps.items[i].id);
});
```

### Cross-question: *"Can overusing memoization hurt performance?"*

**Yes.** Every `useMemo`/`useCallback`:
1. Has a **memory cost** — stores previous deps + result
2. Has a **comparison cost** — checks deps on every render
3. Makes code **harder to read**

For cheap components, the comparison cost can exceed the re-render cost. **Profile first, memoize second.**

---

## 4. State Design for Performance

### Q: How does state structure affect performance?

**Flat vs nested state:**

```tsx
// ❌ Deeply nested — updating one field causes massive spread
const [state, setState] = useState({
  user: { profile: { address: { city: 'Delhi' } } }
});
setState(prev => ({
  ...prev,
  user: { ...prev.user, profile: { ...prev.user.profile, address: { ...prev.user.profile.address, city: 'Mumbai' } } }
}));

// ✅ Flat — each piece updates independently
const [city, setCity] = useState('Delhi');
const [name, setName] = useState('Alice');
// Only components using city re-render when city changes
```

### State colocation vs lifting

```
Where should state live?

Does only ONE component use it?
  └─ YES → useState in THAT component (colocation)

Do SIBLINGS share it?
  └─ YES → Lift to nearest common parent

Do DISTANT components share it?
  └─ YES → Context or state manager (Redux, Zustand)

Is it SERVER data (API cache)?
  └─ YES → React Query / SWR (not local state)
```

### Q: Why does Context cause performance problems?

```tsx
const AppContext = React.createContext();

function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('dark');
  const [cart, setCart] = useState([]);

  // ❌ Every value change re-renders ALL consumers
  return (
    <AppContext.Provider value={{ user, theme, cart, setUser, setTheme, setCart }}>
      <Header />    {/* only needs user */}
      <Sidebar />   {/* only needs theme */}
      <CartIcon />  {/* only needs cart */}
    </AppContext.Provider>
  );
}
```

**Fix — split contexts by change frequency:**

```tsx
const UserContext = React.createContext();   // rarely changes
const ThemeContext = React.createContext();  // rarely changes
const CartContext = React.createContext();   // frequently changes

// Now: adding to cart only re-renders CartIcon, not Header or Sidebar
```

**Fix — memoize the context value:**

```tsx
function CartProvider({ children }) {
  const [items, setItems] = useState([]);
  const addItem = useCallback((item) => setItems(prev => [...prev, item]), []);

  // ❌ New object every render → all consumers re-render
  // return <CartContext.Provider value={{ items, addItem }}>

  // ✅ Memoize value object
  const value = useMemo(() => ({ items, addItem }), [items, addItem]);
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
```

---

## 5. Virtualization

### Q: How do you render 10,000 items without lag?

**Virtualization**: only render items **visible in the viewport** + a small buffer. Off-screen items are unmounted.

```
┌────────────────────┐
│  Buffer (2 items)  │  ← not visible, pre-rendered
├────────────────────┤
│  Visible viewport  │  ← user sees these (~15 items)
│  ···               │
│  ···               │
├────────────────────┤
│  Buffer (2 items)  │  ← not visible, pre-rendered
└────────────────────┘
│  9,970+ items      │  ← NOT rendered (no DOM nodes)
```

### Libraries

| Library | Best for |
|---|---|
| `@tanstack/react-virtual` | Modern, headless, flexible (recommended) |
| `react-window` | Lightweight, fixed/variable-size lists and grids |
| `react-virtuoso` | Auto-height, grouping, chat-like reverse scroll |

```tsx
// @tanstack/react-virtual example
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50, // estimated row height in px
    overscan: 5,            // buffer items above/below viewport
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {items[virtualRow.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Cross-question: *"What about variable-height items?"*

Use `measureElement` to calculate actual height after render:

```tsx
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 80, // initial estimate
  measureElement: (el) => el.getBoundingClientRect().height, // measure actual
});
```

### Cross-question: *"What breaks with virtualization?"*

1. **Ctrl+F** browser search doesn't find off-screen content (not in DOM)
2. **Accessibility**: screen readers may not announce total count
3. **Scroll position restoration** on navigation requires saving/restoring scroll offset
4. **CSS transitions** on items entering/leaving viewport can be janky

---

## 6. Code Splitting & Lazy Loading

### Q: What is code splitting and why does it matter?

Without code splitting, the entire app ships as **one JS bundle**. Users download code for pages they may never visit.

```
Before: bundle.js (2.5 MB) — user waits for full download
After:  main.js (400 KB) + lazy chunks loaded on demand
```

### Route-based splitting with `React.lazy`

```tsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Analytics = lazy(() => import('./pages/Analytics'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}
```

### Component-based splitting (heavy components)

```tsx
// Only load the rich text editor when user opens it
const RichEditor = lazy(() => import('./components/RichEditor'));

function PostForm() {
  const [showEditor, setShowEditor] = useState(false);

  return (
    <div>
      <button onClick={() => setShowEditor(true)}>Write Post</button>
      {showEditor && (
        <Suspense fallback={<Spinner />}>
          <RichEditor />
        </Suspense>
      )}
    </div>
  );
}
```

### Prefetching — load before user needs it

```tsx
// Prefetch on hover (anticipate navigation)
const SettingsPage = lazy(() => import('./pages/Settings'));

function NavLink() {
  const prefetch = () => import('./pages/Settings'); // triggers download

  return (
    <Link
      to="/settings"
      onMouseEnter={prefetch}  // start loading on hover
      onFocus={prefetch}       // also on keyboard focus
    >
      Settings
    </Link>
  );
}
```

### `React.lazy` + named exports

```tsx
// React.lazy only supports default exports
// ❌ Won't work
const Chart = lazy(() => import('./Chart').then(m => m.Chart));

// ✅ Workaround for named exports
const Chart = lazy(() =>
  import('./Chart').then(module => ({ default: module.Chart }))
);
```

---

## 7. Image & Asset Optimization

### Q: How do you optimize images in React?

```tsx
// 1. Lazy loading — browser-native
<img src="photo.jpg" loading="lazy" alt="Product" />

// 2. Responsive images with srcSet
<img
  src="photo-400.jpg"
  srcSet="photo-400.jpg 400w, photo-800.jpg 800w, photo-1200.jpg 1200w"
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  alt="Product"
/>

// 3. Modern formats (WebP/AVIF) with fallback
<picture>
  <source srcSet="photo.avif" type="image/avif" />
  <source srcSet="photo.webp" type="image/webp" />
  <img src="photo.jpg" alt="Product" />
</picture>

// 4. Intersection Observer for below-fold images
function LazyImage({ src, alt }) {
  const ref = useRef();
  const isVisible = useIntersectionObserver(ref, { threshold: 0.1 });
  return (
    <div ref={ref}>
      {isVisible ? <img src={src} alt={alt} /> : <Placeholder />}
    </div>
  );
}

// 5. Next.js Image component (auto-optimizes)
import Image from 'next/image';
<Image src="/photo.jpg" width={800} height={600} alt="Product" priority={false} />
```

### Other asset optimizations

```
✅ SVGs as React components (no HTTP request)
✅ Icon fonts → inline SVG sprite
✅ Compress with tools: Squoosh, Sharp, imagemin
✅ CDN with edge caching for static assets
✅ Preload critical images: <link rel="preload" as="image" href="hero.webp">
```

---

## 8. Concurrent Features

### Q: What is `useTransition` and when do you use it?

`useTransition` marks a state update as **non-urgent**. React keeps the UI responsive by rendering the urgent update immediately and deferring the heavy one.

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value);           // URGENT — update input immediately

    startTransition(() => {
      setResults(filterProducts(e.target.value)); // NON-URGENT — can be deferred
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ProductList results={results} />
    </>
  );
}
```

### Q: What is `useDeferredValue`?

Defers a **value** instead of wrapping an **update**:

```tsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  // deferredQuery updates AFTER React finishes urgent work
  const results = useMemo(() => filterProducts(deferredQuery), [deferredQuery]);

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <ProductList results={results} />
    </div>
  );
}
```

### `useTransition` vs `useDeferredValue`

| `useTransition` | `useDeferredValue` |
|---|---|
| Wraps the **setState call** | Wraps the **value** |
| You control what's deferred | React defers the value automatically |
| Use when you **own** the state update | Use when you **receive** a prop you can't control |
| Returns `[isPending, startTransition]` | Returns deferred copy of the value |

### Cross-question: *"How is this different from debouncing?"*

| Debounce | `useTransition` |
|---|---|
| Delays the **update** by fixed time | Updates immediately, but renders in **background** |
| User sees nothing until delay ends | User sees stale UI → fresh UI (no blank gap) |
| Fixed delay regardless of device speed | Adapts to device — fast devices show result faster |
| Drops intermediate values | Processes all values (may skip renders for old ones) |

---

## 9. Profiling & Measurement

### Q: How do you identify performance problems in React?

**Tool 1: React DevTools Profiler**

```
1. Open DevTools → Profiler tab → ⏺ Record
2. Interact with the app
3. Stop recording
4. Review flamegraph — see which components rendered and why
```

Key metrics:
- **Render duration**: time the component took to render
- **Why did this render?** (Enable in Profiler settings): "props changed", "parent rendered", "hooks changed"
- **Commit frequency**: how many times the component rendered during the interaction

**Tool 2: React `<Profiler>` component (programmatic)**

```tsx
import { Profiler } from 'react';

function onRenderCallback(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  // Send to monitoring
  metrics.recordComponentRender({
    component: id,
    phase,           // "mount" or "update"
    actualDuration,  // time spent rendering this update (ms)
    baseDuration,    // estimated time to render without memoization (ms)
  });
}

<Profiler id="ProductList" onRender={onRenderCallback}>
  <ProductList items={items} />
</Profiler>
```

**Tool 3: Browser Performance tab**

```
1. Chrome DevTools → Performance → Record
2. Look for "Long Tasks" (>50ms) blocking the main thread
3. Check for Layout Thrashing (forced reflows)
4. Analyze JS execution time vs idle time
```

**Tool 4: Web Vitals**

```tsx
import { onCLS, onFID, onLCP, onINP } from 'web-vitals';

onLCP(metric => analytics.send('LCP', metric.value));   // Largest Contentful Paint
onINP(metric => analytics.send('INP', metric.value));   // Interaction to Next Paint
onCLS(metric => analytics.send('CLS', metric.value));   // Cumulative Layout Shift
```

| Metric | Good | Needs Work | Poor |
|---|---|---|---|
| LCP (load speed) | < 2.5s | 2.5–4s | > 4s |
| INP (responsiveness) | < 200ms | 200–500ms | > 500ms |
| CLS (visual stability) | < 0.1 | 0.1–0.25 | > 0.25 |

---

## 10. Network & Data Fetching

### Q: How do you optimize API calls in React?

**1. Caching with React Query / SWR**

```tsx
const { data } = useQuery({
  queryKey: ['products', category],
  queryFn: () => fetchProducts(category),
  staleTime: 5 * 60 * 1000,        // data is "fresh" for 5 minutes
  gcTime: 30 * 60 * 1000,          // keep in cache for 30 minutes
  refetchOnWindowFocus: false,      // no refetch when tab regains focus
});
```

**2. Request deduplication** (React Query handles this automatically):
```
Component A: useQuery(['user', 1]) ─┐
Component B: useQuery(['user', 1]) ─┤ → ONE network request
Component C: useQuery(['user', 1]) ─┘
```

**3. Optimistic updates**
```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries(['todos']);
    const previous = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], old => [...old, newTodo]); // optimistic
    return { previous };
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previous); // rollback
  },
});
```

**4. Prefetching** (load data before navigation):
```tsx
// On hover — prefetch the next page's data
<Link
  to={`/product/${id}`}
  onMouseEnter={() => queryClient.prefetchQuery({
    queryKey: ['product', id],
    queryFn: () => fetchProduct(id),
  })}
>
```

**5. Pagination strategies**

| Strategy | UX | Data |
|---|---|---|
| **Offset pagination** | Page 1, 2, 3… | `?page=2&limit=20` |
| **Cursor pagination** | Load more | `?cursor=abc123&limit=20` |
| **Infinite scroll** | Seamless | useInfiniteQuery + intersection observer |

---

## 11. Bundle Size Optimization

### Q: How do you reduce bundle size?

**1. Analyze the bundle**
```bash
# Vite
npx vite-bundle-visualizer

# Webpack
npx webpack-bundle-analyzer stats.json
```

**2. Tree shaking — import only what you need**
```tsx
// ❌ Imports entire library (400 KB)
import _ from 'lodash';
_.debounce(...);

// ✅ Import only the function (4 KB)
import debounce from 'lodash/debounce';

// ✅ Or use lodash-es (tree-shakeable)
import { debounce } from 'lodash-es';
```

**3. Replace heavy libraries with lighter alternatives**

| Heavy | Light Alternative | Saving |
|---|---|---|
| `moment.js` (300 KB) | `date-fns` (tree-shake) or `dayjs` (2 KB) | ~295 KB |
| `lodash` (72 KB) | `lodash-es` (tree-shake) or native | ~60 KB |
| `axios` (14 KB) | Native `fetch` | 14 KB |
| `uuid` (5 KB) | `crypto.randomUUID()` | 5 KB |
| `classnames` | Template literals | 1 KB |

**4. Dynamic imports for optional features**
```tsx
const handleExport = async () => {
  const { exportToPDF } = await import('./utils/pdfExport'); // loaded only when needed
  exportToPDF(data);
};
```

**5. Externalize large dependencies** (load from CDN):
```js
// vite.config.js
build: {
  rollupOptions: {
    external: ['react', 'react-dom'],
    output: { globals: { react: 'React', 'react-dom': 'ReactDOM' } },
  },
}
```

---

## 12. Architecture Patterns

### Pattern 1: Render slicing — split expensive renders

```tsx
// ❌ One giant render — blocks UI
function Dashboard() {
  return (
    <div>
      <Header />
      <HeavyChart data={data} />       {/* 200ms render */}
      <HeavyTable data={data} />       {/* 300ms render */}
      <HeavyMap coordinates={coords} /> {/* 250ms render */}
    </div>
  );
}

// ✅ Wrap non-critical sections in Suspense + lazy
const HeavyChart = lazy(() => import('./HeavyChart'));
const HeavyTable = lazy(() => import('./HeavyTable'));
const HeavyMap = lazy(() => import('./HeavyMap'));

function Dashboard() {
  return (
    <div>
      <Header />
      <Suspense fallback={<ChartSkeleton />}><HeavyChart data={data} /></Suspense>
      <Suspense fallback={<TableSkeleton />}><HeavyTable data={data} /></Suspense>
      <Suspense fallback={<MapSkeleton />}><HeavyMap coordinates={coords} /></Suspense>
    </div>
  );
}
```

### Pattern 2: Debounced renders for fast-changing data

```tsx
function StockTicker({ prices }) {
  const deferredPrices = useDeferredValue(prices);
  const isStale = prices !== deferredPrices;

  return (
    <div style={{ opacity: isStale ? 0.7 : 1, transition: 'opacity 0.2s' }}>
      {deferredPrices.map(p => <PriceRow key={p.symbol} {...p} />)}
    </div>
  );
}
```

### Pattern 3: Web Workers for CPU-heavy tasks

```tsx
// worker.js
self.onmessage = (e) => {
  const result = heavyComputation(e.data); // runs off main thread
  self.postMessage(result);
};

// Component
function DataProcessor({ rawData }) {
  const [result, setResult] = useState(null);

  useEffect(() => {
    const worker = new Worker(new URL('./worker.js', import.meta.url));
    worker.postMessage(rawData);
    worker.onmessage = (e) => setResult(e.data);
    return () => worker.terminate();
  }, [rawData]);

  return result ? <Chart data={result} /> : <Spinner />;
}
```

### Pattern 4: Event delegation instead of per-item handlers

```tsx
// ❌ 1000 event handlers (1000 function allocations)
{items.map(item => (
  <div key={item.id} onClick={() => handleClick(item.id)}>
    {item.name}
  </div>
))}

// ✅ One delegated handler on parent
function ItemList({ items, onSelectItem }) {
  const handleClick = useCallback((e) => {
    const id = e.target.closest('[data-id]')?.dataset.id;
    if (id) onSelectItem(id);
  }, [onSelectItem]);

  return (
    <div onClick={handleClick}>
      {items.map(item => (
        <div key={item.id} data-id={item.id}>{item.name}</div>
      ))}
    </div>
  );
}
```

---

## 13. MANG Cross-Questions & Scenarios

---

### Scenario 1: Slow Product Listing Page

> **"Your e-commerce product page shows 500 products with images, filters, and sorting. Users report it's slow on mobile. Walk through your optimization process."**

**Step-by-step answer:**

1. **Profile first** — React DevTools Profiler + Chrome Performance tab
2. **Identify the bottleneck:**
   - Is it **rendering**? (JS execution time, React components)
   - Is it **paint**? (too many DOM nodes, complex CSS)
   - Is it **network**? (large images, slow API)

3. **Likely fixes (in priority order):**

| Fix | Impact |
|---|---|
| **Virtualize** the product grid (react-virtual) | Huge — renders ~20 items instead of 500 |
| **Lazy load images** (`loading="lazy"` + WebP) | Large — reduce initial bandwidth |
| **Memoize** product cards (`React.memo`) | Medium — prevent re-renders on filter change |
| **Debounce** filter inputs | Medium — fewer filter re-computations |
| **`useTransition`** on sort/filter state | Medium — keep input responsive |
| **Move filter computation** to Web Worker | Low-medium — unblock main thread |
| **Paginate** API (cursor-based) + infinite scroll | Large — fetch only visible page |

### Cross-question: *"How do you prove it's faster?"*

```tsx
// Before/after INP measurement
onINP(metric => console.log('INP:', metric.value));

// React Profiler baseDuration comparison
<Profiler id="ProductGrid" onRender={(id, phase, actualDuration) => {
  console.log(`${id} rendered in ${actualDuration}ms`);
}}>
```

---

### Scenario 2: Context Causing Full-App Re-renders

> **"Every keystroke in a search input causes the entire app to re-render because the search query is in a global AppContext."**

**Answer:**

```tsx
// ❌ One giant context
<AppContext.Provider value={{ user, theme, searchQuery, cart, notifications }}>

// ✅ Split into separate contexts by change frequency
<UserContext.Provider value={user}>           {/* rarely changes */}
  <ThemeContext.Provider value={theme}>        {/* rarely changes */}
    <CartContext.Provider value={cart}>         {/* sometimes changes */}
      <SearchContext.Provider value={query}>    {/* changes rapidly */}
        {children}
      </SearchContext.Provider>
    </CartContext.Provider>
  </ThemeContext.Provider>
</UserContext.Provider>
```

**Additional fixes:**
- `useDeferredValue(searchQuery)` for components that show filtered results
- State colocation — keep `searchQuery` in the SearchBar component, not context
- Use Zustand or Jotai — both have **atomic subscriptions** (only consumers of changed atom re-render)

---

### Scenario 3: Expensive Form with 100 Fields

> **"An insurance form has 100+ fields across tabs. Every keystroke causes all tabs to re-render."**

**Answer:**

1. **Use React Hook Form** (uncontrolled inputs — no re-render per keystroke)
2. **Isolate each tab** into its own component (separate React subtree)
3. **`React.memo` each tab** — only re-render when its own data changes
4. **Lazy load hidden tabs** — `React.lazy` for tabs not yet visited

```tsx
// React Hook Form — no re-render on type
const { register, handleSubmit } = useForm();

<input {...register('firstName')} /> // uncontrolled — React doesn't track value
// vs
<input value={value} onChange={handleChange} /> // controlled — re-render every keystroke
```

### Cross-question: *"When would you still use controlled inputs?"*

When you need **real-time validation**, **dependent field updates** (city changes when state changes), or **input formatting** (credit card number with spaces). But isolate those to individual field components with `React.memo`.

---

### Scenario 4: Memory Leak Debugging

> **"Your app's memory usage grows continuously. Users report the tab crashes after 30 minutes. How do you find the leak?"**

**Debugging steps:**

1. **Chrome DevTools → Memory → Heap Snapshot**
   - Take snapshot 1 → interact → snapshot 2 → compare
   - Look for objects that only grow (detached DOM nodes, event listeners)

2. **Common React memory leaks:**

```tsx
// Leak 1: Missing effect cleanup
useEffect(() => {
  const id = setInterval(fetchData, 1000);
  // ❌ No cleanup — interval runs forever after unmount
  // ✅ return () => clearInterval(id);
}, []);

// Leak 2: Event listeners not removed
useEffect(() => {
  window.addEventListener('resize', handler);
  // ❌ Missing cleanup
  // ✅ return () => window.removeEventListener('resize', handler);
}, []);

// Leak 3: Stale closures holding references
useEffect(() => {
  const socket = new WebSocket(url);
  socket.onmessage = (e) => {
    setMessages(prev => [...prev, e.data]); // ✅ OK — functional update
    // messages.push(e.data); ❌ holds reference to old array
  };
  return () => socket.close(); // ✅ cleanup
}, []);

// Leak 4: AbortController not used — fetch completes after unmount
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal }).then(setData);
  return () => controller.abort(); // ✅ cancel in-flight request
}, [url]);
```

3. **Performance Monitor** (Chrome DevTools → ⋮ → More tools → Performance Monitor)
   - Watch "JS heap size" and "DOM Nodes" over time
   - If either grows linearly → leak

---

### Scenario 5: Optimizing Initial Load (Core Web Vitals)

> **"LCP is 4.2s on your landing page. Bring it under 2.5s."**

**Answer checklist:**

| Technique | Impact |
|---|---|
| **Code split** by route (lazy load non-landing pages) | High — smaller initial JS |
| **Preload** hero image: `<link rel="preload" as="image" href="hero.webp">` | High — browser fetches early |
| **Remove unused CSS** (PurgeCSS or manual audit) | Medium |
| **Compress images** (WebP/AVIF + srcset) | High |
| **CDN** for static assets with edge caching | High — reduce latency |
| **Server-side render** the landing page (Next.js) | High — HTML arrives with content |
| **Preconnect** to API/CDN: `<link rel="preconnect" href="https://api.example.com">` | Medium |
| **Reduce third-party scripts** (analytics, chat widgets) | Medium — they block main thread |
| **Font optimization**: `font-display: swap`, preload critical fonts | Medium — avoid FOIT |
| **HTTP/2** server push or 103 Early Hints | Low — browser fetches critical resources earlier |

### Cross-question: *"How do you handle CLS from images?"*

```tsx
// Always set width/height or aspect-ratio to prevent layout shift
<img src="photo.jpg" width={800} height={600} alt="Product" />

// CSS approach
.image-container {
  aspect-ratio: 16 / 9;
  width: 100%;
}
```

---

### Scenario 6: Real-time Dashboard (60fps)

> **"Build a dashboard that updates 10 charts with live WebSocket data every 100ms. It must stay at 60fps."**

**Architecture:**

```tsx
// 1. Buffer WebSocket updates (don't dispatch per message)
const bufferRef = useRef({});
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (e) => { bufferRef.current[e.data.id] = e.data; };
  // Flush at 10fps (not per WebSocket message)
  const id = setInterval(() => {
    if (Object.keys(bufferRef.current).length) {
      setChartData(prev => ({ ...prev, ...bufferRef.current }));
      bufferRef.current = {};
    }
  }, 100);
  return () => { ws.close(); clearInterval(id); };
}, []);

// 2. Each chart is memoized + only receives its own data
const ChartWidget = React.memo(({ data }) => <LineChart data={data} />);

// 3. Use individual selectors per chart
{chartIds.map(id => (
  <ChartWidget key={id} data={chartData[id]} />
  // Only the chart whose data changed re-renders
))}

// 4. Heavy chart rendering → requestAnimationFrame or canvas instead of SVG
// 5. Move data transformations to Web Worker
```

---

### Quick Reference: Optimization Decision Matrix

```
Performance Problem?
│
├─ Too many RE-RENDERS?
│  ├─ State too high → colocate state
│  ├─ Context too broad → split contexts
│  ├─ Unstable props → useMemo / useCallback
│  └─ Expensive children → React.memo
│
├─ Too many DOM NODES?
│  └─ Virtualization (react-virtual)
│
├─ SLOW INITIAL LOAD?
│  ├─ Code splitting (React.lazy)
│  ├─ Image optimization
│  ├─ Bundle analysis + tree shaking
│  └─ SSR / streaming
│
├─ JANKY INTERACTIONS?
│  ├─ useTransition for non-urgent updates
│  ├─ useDeferredValue for heavy renders
│  ├─ Web Workers for CPU-heavy computation
│  └─ requestAnimationFrame for animations
│
└─ MEMORY LEAK?
   ├─ Missing useEffect cleanup
   ├─ Event listeners not removed
   └─ AbortController for fetch
```

---

*Last updated: February 2026 | React 18 + React 19 · Vite · Next.js*
