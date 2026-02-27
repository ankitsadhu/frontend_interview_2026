# Custom Hooks — MANG Level Interview (25 Questions)

> Deep-dive questions with code, cross-questions, gotchas, and real-world scenarios

---

## Q1: What is a custom hook and what are the rules?

**A:** A custom hook is a **JavaScript function whose name starts with `use`** that calls other hooks. It extracts **reusable stateful logic** (not UI) from components.

**Rules of Hooks (enforced by `react-hooks/rules-of-hooks` ESLint rule):**
1. Only call hooks **at the top level** — never inside conditions, loops, or nested functions
2. Only call hooks from **React function components** or **other custom hooks**

```jsx
// ✅ Valid custom hook
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  useEffect(() => {
    const handler = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  return width;
}

// ❌ Breaks Rule 1 — hook inside a condition
function useBadHook(flag) {
  if (flag) {
    const [val, setVal] = useState(0); // React can't track hook call order!
  }
}
```

### Cross-question: *"Why can't hooks be called conditionally?"*

React stores hook state in a **linked list** on the fiber node, indexed by **call order**. If you skip a hook call conditionally, all subsequent hooks shift index → they return the wrong state.

```
Render 1:  useState(0)  → slot 0    useEffect(...) → slot 1
Render 2:  (skipped)                 useEffect(...) → slot 0  ← WRONG state!
```

---

## Q2: Build `useToggle`

**A:** The simplest example — but interviewers test if you handle edge cases.

```ts
function useToggle(initialValue = false): [boolean, () => void] {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// Usage
const [isOpen, toggleOpen] = useToggle();
<button onClick={toggleOpen}>{isOpen ? 'Close' : 'Open'}</button>
```

### Cross-question: *"Why `useCallback` on toggle?"*

Without `useCallback`, `toggle` is a **new function reference** every render. If passed to a `React.memo` child, the child re-renders anyway — defeating memoization. `useCallback` with `[]` deps ensures **same reference** across renders.

---

## Q3: Build `usePrevious`

**A:** Stores the previous value of a prop or state using `useRef`:

```ts
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);

  useEffect(() => {
    ref.current = value;
  }); // no dep array — runs AFTER every render

  return ref.current; // returns the value from the PREVIOUS render
}

// Usage
const [count, setCount] = useState(0);
const prevCount = usePrevious(count);
// Render 1: count=0, prevCount=undefined
// Render 2: count=5, prevCount=0
```

### Cross-question: *"Why useRef and not useState?"*

`useState` would trigger a re-render when setting the previous value → **infinite loop**. `useRef` updates silently (no re-render), which is exactly what we need for storing previous values.

### Cross-question: *"Why no dependency array on useEffect?"*

We want to capture the value **after every render** — not just when `value` changes. This guarantees `ref.current` always holds what `value` was on the previous render.

---

## Q4: Build `useDebounce`

```ts
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer); // cleanup previous timer on value/delay change
  }, [value, delay]);

  return debouncedValue;
}

// Usage — search input
function SearchPage() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) fetchResults(debouncedQuery);
  }, [debouncedQuery]); // fires 300ms after user stops typing

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

### Cross-question: *"What's the difference between debouncing the value vs debouncing the callback?"*

| `useDebounce(value)` | `useDebouncedCallback(fn)` |
|---|---|
| Returns a debounced **copy** of the value | Returns a debounced **function** |
| Declarative — the value updates after delay | Imperative — you call the function |
| Pairs well with `useEffect` watching the value | Pairs well with event handlers like `onChange` |

```ts
// Debounced callback version
function useDebouncedCallback(fn: (...args: any[]) => void, delay: number) {
  const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const debouncedFn = useCallback((...args: any[]) => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
    timeoutRef.current = setTimeout(() => fn(...args), delay);
  }, [fn, delay]);

  // Cleanup on unmount
  useEffect(() => () => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
  }, []);

  return debouncedFn;
}
```

---

## Q5: Build `useFetch` with loading, error, and cancellation

```ts
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(setData)
      .catch(err => {
        if (err.name !== 'AbortError') setError(err); // ignore abort
      })
      .finally(() => setLoading(false));

    return () => controller.abort(); // cancel on url change or unmount
  }, [url]);

  return { data, loading, error };
}
```

### Cross-question: *"Why AbortController instead of a boolean cancelled flag?"*

| `cancelled` flag | `AbortController` |
|---|---|
| Only prevents **setState** after unmount | Actually **cancels the HTTP request** in flight |
| Network request still completes | Saves bandwidth, faster cleanup |
| Server still processes the request | Server receives TCP RST (for long requests) |

### Cross-question: *"What's missing from this hook for production use?"*

1. **Caching** — same URL fetched twice creates two requests
2. **Deduplication** — multiple components mounting simultaneously fire duplicate requests
3. **Stale-while-revalidate** — show old data while refreshing
4. **Retry logic** — retry on failure with exponential backoff
5. **Refetch on focus** — re-fetch when the tab regains focus

→ This is exactly why libraries like **React Query / SWR** exist. They solve all of the above.

---

## Q6: Build `useLocalStorage`

```ts
function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T | ((prev: T) => T)) => void] {
  // Lazy initializer — only reads localStorage once on mount
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value: T | ((prev: T) => T)) => {
    setStoredValue(prev => {
      const newValue = value instanceof Function ? value(prev) : value;
      localStorage.setItem(key, JSON.stringify(newValue));
      return newValue;
    });
  }, [key]);

  return [storedValue, setValue];
}

// Usage
const [theme, setTheme] = useLocalStorage('theme', 'dark');
```

### Cross-question: *"What if another tab updates the same key?"*

Add a `storage` event listener to sync across tabs:

```ts
useEffect(() => {
  const handleStorage = (e: StorageEvent) => {
    if (e.key === key && e.newValue) {
      setStoredValue(JSON.parse(e.newValue));
    }
  };
  window.addEventListener('storage', handleStorage);
  return () => window.removeEventListener('storage', handleStorage);
}, [key]);
```

### Cross-question: *"Why a lazy initializer in useState?"*

`useState(() => expensiveComputation())` runs the function **only on mount**. Without it, `JSON.parse(localStorage.getItem(key))` runs on every render (even though useState ignores it after the first).

---

## Q7: Build `useOnClickOutside`

```ts
function useOnClickOutside(ref: RefObject<HTMLElement>, handler: (event: MouseEvent | TouchEvent) => void) {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      if (!ref.current || ref.current.contains(event.target as Node)) {
        return; // click is inside the ref element — do nothing
      }
      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}

// Usage — close dropdown when clicking outside
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useOnClickOutside(ref, () => setIsOpen(false));

  return (
    <div ref={ref}>
      <button onClick={() => setIsOpen(v => !v)}>Menu</button>
      {isOpen && <ul className="dropdown">...</ul>}
    </div>
  );
}
```

### Cross-question: *"Why `mousedown` instead of `click`?"*

`mousedown` fires **before** `click`. If you used `click`, the handler would fire after the dropdown's own click handler, potentially causing race conditions (e.g., reopening a dropdown that was just closed).

---

## Q8: Build `useMediaQuery`

```ts
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);

  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mql.addEventListener('change', handler);
    setMatches(mql.matches); // sync if query changed
    return () => mql.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// Usage
function Layout() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const prefersDark = useMediaQuery('(prefers-color-scheme: dark)');
  return isMobile ? <MobileNav /> : <DesktopNav />;
}
```

### Cross-question: *"How would you make this SSR-safe?"*

On the server, `window` doesn't exist. Options:
1. Default to `false` and update on mount (causes flash)
2. Accept a `defaultValue` parameter: `useMediaQuery(query, { defaultValue: false })`
3. Use `useSyncExternalStore` with a `getServerSnapshot` that returns the default

---

## Q9: Build `useIntersectionObserver` (lazy loading / infinite scroll)

```ts
function useIntersectionObserver(
  ref: RefObject<HTMLElement>,
  options: IntersectionObserverInit = {}
): boolean {
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(([entry]) => {
      setIsIntersecting(entry.isIntersecting);
    }, options);

    observer.observe(element);
    return () => observer.disconnect();
  }, [ref, options.threshold, options.root, options.rootMargin]);

  return isIntersecting;
}

// Usage — infinite scroll sentinel
function Feed() {
  const sentinelRef = useRef<HTMLDivElement>(null);
  const isVisible = useIntersectionObserver(sentinelRef, { threshold: 1.0 });

  useEffect(() => {
    if (isVisible) loadMore();
  }, [isVisible]);

  return (
    <div>
      {posts.map(p => <Post key={p.id} {...p} />)}
      <div ref={sentinelRef} /> {/* invisible sentinel at bottom */}
    </div>
  );
}
```

### Cross-question: *"Why do you put options fields in the dep array instead of the options object?"*

If you put `options` (object) in deps, it triggers re-creation of the observer on **every render** because `{}` !== `{}` (new reference each time). By extracting primitives (`threshold`, `rootMargin`), deps are stable.

---

## Q10: Build `useEventListener`

```ts
function useEventListener<K extends keyof WindowEventMap>(
  eventName: K,
  handler: (event: WindowEventMap[K]) => void,
  element: HTMLElement | Window = window
) {
  const handlerRef = useRef(handler);

  // Keep handler ref up-to-date (avoids stale closures)
  useEffect(() => {
    handlerRef.current = handler;
  }, [handler]);

  useEffect(() => {
    const listener = (event: Event) => handlerRef.current(event as WindowEventMap[K]);
    element.addEventListener(eventName, listener);
    return () => element.removeEventListener(eventName, listener);
  }, [eventName, element]); // handler NOT in deps — ref handles it
}

// Usage
useEventListener('keydown', (e) => {
  if (e.key === 'Escape') closeModal();
});
```

### Cross-question: *"Why store handler in a ref?"*

The ref pattern prevents two problems:
1. **Stale closures** — ref always points to the latest handler (latest props/state)
2. **Unnecessary listener re-attachment** — without ref, changing the handler function re-runs the effect → removes old listener + adds new one on every render

---

## Q11: Build `useInterval`

```ts
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);

  // Update ref to latest callback (no stale closure)
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return; // null = paused

    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]); // only re-create interval when delay changes
}

// Usage — stopwatch
function Timer() {
  const [count, setCount] = useState(0);
  const [running, setRunning] = useState(true);

  useInterval(() => setCount(c => c + 1), running ? 1000 : null);

  return <p>{count}s <button onClick={() => setRunning(r => !r)}>
    {running ? 'Pause' : 'Resume'}
  </button></p>;
}
```

### Cross-question: *"Why not just use `setInterval` in a `useEffect`?"*

```ts
// ❌ Naive approach — stale closure bug
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // captures count=0 forever (stale!)
  }, 1000);
  return () => clearInterval(id);
}, []); // count is not in deps

// If you add [count] to deps, the interval clears/resets every second — not a true interval
```

The `useInterval` hook solves both problems by separating the **timer lifecycle** (deps: `delay`) from the **callback** (ref: always fresh).

---

## Q12: Build `useThrottle`

```ts
function useThrottle<T>(value: T, interval: number): T {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastRan = useRef(Date.now());

  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= interval) {
        setThrottledValue(value);
        lastRan.current = Date.now();
      }
    }, interval - (Date.now() - lastRan.current));

    return () => clearTimeout(handler);
  }, [value, interval]);

  return throttledValue;
}

// Usage — throttle scroll position updates
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);
  useEventListener('scroll', () => setScrollY(window.scrollY));

  const throttledY = useThrottle(scrollY, 200); // max 5 updates/sec
  // Use throttledY for analytics, API calls, etc.
}
```

### Cross-question: *"Debounce vs throttle — when to use which?"*

| Debounce | Throttle |
|---|---|
| Waits until user **stops** | Fires at **regular intervals** |
| Search input, form validation | Scroll tracking, resize, mousemove |
| Last value wins | Guaranteed max frequency |

---

## Q13: Build `useAsync` — generic async state machine

```ts
type AsyncState<T> =
  | { status: 'idle'; data: null; error: null }
  | { status: 'loading'; data: null; error: null }
  | { status: 'success'; data: T; error: null }
  | { status: 'error'; data: null; error: Error };

function useAsync<T>(asyncFn: () => Promise<T>, deps: any[] = []) {
  const [state, setState] = useState<AsyncState<T>>({
    status: 'idle', data: null, error: null,
  });

  const execute = useCallback(() => {
    setState({ status: 'loading', data: null, error: null });
    return asyncFn()
      .then(data => { setState({ status: 'success', data, error: null }); return data; })
      .catch(error => { setState({ status: 'error', data: null, error }); throw error; });
  }, deps);

  return { ...state, execute };
}

// Usage
function CreateUserButton() {
  const { status, error, execute } = useAsync(
    () => api.createUser({ name: 'Alice' }),
    []
  );

  return (
    <button onClick={execute} disabled={status === 'loading'}>
      {status === 'loading' ? 'Creating…' : 'Create User'}
      {status === 'error' && <span>{error.message}</span>}
    </button>
  );
}
```

### Cross-question: *"Why is the state typed as a discriminated union?"*

With a union, TypeScript guarantees that if `status === 'success'` then `data` is `T` (not null). If `status === 'error'` then `error` is `Error`. You can't accidentally access `data` when in error state — **impossible states are impossible**.

---

## Q14: Build `useOnMount` and `useOnUnmount`

```ts
function useOnMount(fn: () => void) {
  // eslint-disable-next-line react-hooks/exhaustive-deps
  useEffect(fn, []); // intentionally empty deps — run only on mount
}

function useOnUnmount(fn: () => void) {
  const fnRef = useRef(fn);
  fnRef.current = fn; // always latest (avoids stale closure)
  useEffect(() => () => fnRef.current(), []);
}

// Usage
useOnMount(() => analytics.trackPageView('/dashboard'));
useOnUnmount(() => cleanup());
```

### Cross-question: *"Are these hooks bad practice? The React team discourages `useEffect` for mount-only logic."*

For **setup/cleanup** (subscriptions, event listeners): yes, mount/unmount is fine — it's the pattern `useEffect` was designed for.

For **data fetching**: don't use `useOnMount` — use a data-fetching library (React Query) or the `use()` hook. The React team is moving toward Suspense-based fetching, where `useEffect` for fetching is an anti-pattern.

---

## Q15: Build `useKeyPress`

```ts
function useKeyPress(targetKey: string): boolean {
  const [pressed, setPressed] = useState(false);

  useEffect(() => {
    const down = (e: KeyboardEvent) => { if (e.key === targetKey) setPressed(true); };
    const up = (e: KeyboardEvent) => { if (e.key === targetKey) setPressed(false); };

    window.addEventListener('keydown', down);
    window.addEventListener('keyup', up);
    return () => {
      window.removeEventListener('keydown', down);
      window.removeEventListener('keyup', up);
    };
  }, [targetKey]);

  return pressed;
}

// Usage — game controls
function Game() {
  const left = useKeyPress('ArrowLeft');
  const right = useKeyPress('ArrowRight');
  const space = useKeyPress(' ');
  // Move character based on active keys
}
```

---

## Q16: Build `useForm` — generic form state hook

```ts
function useForm<T extends Record<string, any>>(initialValues: T) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});

  const handleChange = useCallback((e: ChangeEvent<HTMLInputElement>) => {
    const { name, value, type, checked } = e.target;
    setValues(prev => ({ ...prev, [name]: type === 'checkbox' ? checked : value }));
  }, []);

  const handleBlur = useCallback((e: FocusEvent<HTMLInputElement>) => {
    setTouched(prev => ({ ...prev, [e.target.name]: true }));
  }, []);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  }, [initialValues]);

  const setFieldError = useCallback((field: keyof T, message: string) => {
    setErrors(prev => ({ ...prev, [field]: message }));
  }, []);

  return { values, errors, touched, handleChange, handleBlur, reset, setFieldError, setValues };
}

// Usage
const { values, errors, handleChange, handleBlur } = useForm({ email: '', password: '' });
<input name="email" value={values.email} onChange={handleChange} onBlur={handleBlur} />
{errors.email && <span>{errors.email}</span>}
```

### Cross-question: *"Why would you still use React Hook Form over this?"*

1. **Performance** — RHF uses **uncontrolled inputs** (no re-render per keystroke). This custom `useForm` is controlled — every keystroke triggers a render.
2. **Validation integration** — RHF has native Zod/Yup integration
3. **Field array management** — `useFieldArray` for dynamic field lists
4. **Field isolation** — RHF only re-renders the specific field that changed

---

## Q17: Build `useCopyToClipboard`

```ts
function useCopyToClipboard(): [boolean, (text: string) => Promise<void>] {
  const [copied, setCopied] = useState(false);
  const timeoutRef = useRef<ReturnType<typeof setTimeout>>();

  const copy = useCallback(async (text: string) => {
    try {
      await navigator.clipboard.writeText(text);
      setCopied(true);
      clearTimeout(timeoutRef.current);
      timeoutRef.current = setTimeout(() => setCopied(false), 2000); // reset after 2s
    } catch {
      setCopied(false);
    }
  }, []);

  useEffect(() => () => clearTimeout(timeoutRef.current), []);

  return [copied, copy];
}

// Usage
const [copied, copy] = useCopyToClipboard();
<button onClick={() => copy('https://example.com')}>
  {copied ? '✓ Copied!' : 'Copy Link'}
</button>
```

---

## Q18: Build `usePermission` — check browser Permissions API

```ts
function usePermission(permissionName: PermissionName): PermissionState | 'loading' {
  const [state, setState] = useState<PermissionState | 'loading'>('loading');

  useEffect(() => {
    let mounted = true;
    navigator.permissions.query({ name: permissionName }).then(result => {
      if (mounted) setState(result.state);
      result.addEventListener('change', () => {
        if (mounted) setState(result.state);
      });
    });
    return () => { mounted = false; };
  }, [permissionName]);

  return state;
}

// Usage
function LocationButton() {
  const permState = usePermission('geolocation');
  if (permState === 'denied') return <p>Location access denied</p>;
  if (permState === 'prompt') return <button onClick={requestLocation}>Allow Location</button>;
  return <MapComponent />;
}
```

---

## Q19: Build `useSyncExternalStore` from scratch (conceptual)

This is a MANG-level question testing deep React internals knowledge:

```ts
// Simplified version — actual React impl handles concurrent mode, server snapshots, etc.
function useMyExternalStore<T>(subscribe: (cb: () => void) => () => void, getSnapshot: () => T): T {
  const [snapshot, setSnapshot] = useState(getSnapshot);

  useEffect(() => {
    // Check for changes immediately (may have changed between render and effect)
    const check = () => {
      const next = getSnapshot();
      setSnapshot(prev => Object.is(prev, next) ? prev : next);
    };

    check(); // sync check
    const unsubscribe = subscribe(check);
    return unsubscribe;
  }, [subscribe, getSnapshot]);

  return snapshot;
}

// Usage
const width = useMyExternalStore(
  (callback) => {
    window.addEventListener('resize', callback);
    return () => window.removeEventListener('resize', callback);
  },
  () => window.innerWidth
);
```

### Cross-question: *"Why does the real `useSyncExternalStore` exist if we can do this?"*

This simplified version has a **tearing bug** in Concurrent Mode. Between when React reads the snapshot during render and when the effect subscribes, the store could change — causing different components to see different values in the same render pass. The real `useSyncExternalStore` guarantees a **consistent snapshot** across the entire render.

---

## Q20: Build `useReducerWithMiddleware`

```ts
function useReducerWithMiddleware<S, A>(
  reducer: (state: S, action: A) => S,
  initialState: S,
  middlewares: Array<(action: A, state: S) => void>
): [S, (action: A) => void] {
  const [state, dispatch] = useReducer(reducer, initialState);
  const stateRef = useRef(state);
  stateRef.current = state;

  const dispatchWithMiddleware = useCallback((action: A) => {
    // Run all middlewares before dispatch
    middlewares.forEach(mw => mw(action, stateRef.current));
    dispatch(action);
  }, [middlewares]);

  return [state, dispatchWithMiddleware];
}

// Usage — logging middleware
const logger = (action, state) => console.log('Dispatch:', action, 'Current state:', state);
const analytics = (action) => { if (action.type === 'purchase') trackPurchase(); };

const [state, dispatch] = useReducerWithMiddleware(
  reducer, initialState, [logger, analytics]
);
```

---

## Q21: Build `useAbortController`

```ts
function useAbortController(): [AbortSignal, () => void] {
  const controllerRef = useRef(new AbortController());

  const reset = useCallback(() => {
    controllerRef.current.abort();
    controllerRef.current = new AbortController();
  }, []);

  // Abort on unmount
  useEffect(() => () => controllerRef.current.abort(), []);

  return [controllerRef.current.signal, reset];
}

// Usage — cancel in-flight requests on route change
function SearchPage() {
  const [signal, resetAbort] = useAbortController();

  const search = async (query: string) => {
    resetAbort(); // cancel previous search
    const res = await fetch(`/api/search?q=${query}`, { signal });
    return res.json();
  };
}
```

---

## Q22: Build `useDeepCompareEffect`

```ts
import { isEqual } from 'lodash-es';

function useDeepCompareEffect(effect: EffectCallback, deps: any[]) {
  const ref = useRef<any[]>(deps);

  if (!isEqual(ref.current, deps)) {
    ref.current = deps; // only update ref when deeply different
  }

  // eslint-disable-next-line react-hooks/exhaustive-deps
  useEffect(effect, ref.current);
}

// Usage — when deps are objects with same value but new reference each render
useDeepCompareEffect(() => {
  fetchData(filters); // filters is { page: 1, sort: 'name' } — new object each render
}, [filters]);
```

### Cross-question: *"When should you NOT use this?"*

**Almost always.** Deep comparison is O(n) per render. The right fix is usually to **stabilize the reference** with `useMemo`:

```ts
// ✅ Better approach — fix the root cause
const filters = useMemo(() => ({ page, sort }), [page, sort]);
useEffect(() => fetchData(filters), [filters]); // stable reference, no deep compare needed
```

Use `useDeepCompareEffect` only when you can't control the reference (e.g., props from a parent you don't own).

---

## Q23: Build `useLockBodyScroll`

```ts
function useLockBodyScroll(locked: boolean = true) {
  useEffect(() => {
    if (!locked) return;

    const originalOverflow = document.body.style.overflow;
    const scrollbarWidth = window.innerWidth - document.documentElement.clientWidth;

    document.body.style.overflow = 'hidden';
    document.body.style.paddingRight = `${scrollbarWidth}px`; // prevent layout shift

    return () => {
      document.body.style.overflow = originalOverflow;
      document.body.style.paddingRight = '';
    };
  }, [locked]);
}

// Usage — lock scroll when modal is open
function Modal({ isOpen, children }) {
  useLockBodyScroll(isOpen);
  if (!isOpen) return null;
  return createPortal(<div className="modal">{children}</div>, document.body);
}
```

### Cross-question: *"Why add paddingRight?"*

When `overflow: hidden` removes the scrollbar, the page widens by the scrollbar width (~15-17px), causing a visible **layout shift**. Adding `paddingRight` compensates, keeping the layout stable.

---

## Q24: Build `useLatest` — always-fresh ref to avoid stale closures

```ts
function useLatest<T>(value: T): MutableRefObject<T> {
  const ref = useRef(value);
  ref.current = value; // sync on every render (before effects run)
  return ref;
}

// Usage — solving stale closure in intervals
function Chat({ roomId }) {
  const latestRoomId = useLatest(roomId);

  useEffect(() => {
    const id = setInterval(() => {
      // latestRoomId.current is ALWAYS the current roomId — never stale
      sendHeartbeat(latestRoomId.current);
    }, 5000);
    return () => clearInterval(id);
  }, []); // safe to have empty deps!
}
```

### Cross-question: *"Is this the same as `useEffectEvent`?"*

Conceptually similar — both solve stale closures. But `useEffectEvent` (experimental) is a React primitive that **also tells React this function is not a dependency**. `useLatest` is a userland workaround that requires you to manually exclude it from deps (which the exhaustive-deps lint will warn about).

---

## Q25: Scenario — Design a `useNotifications` hook for a real-time system

> **"Design a custom hook that manages real-time notifications via WebSocket. It should support: read/unread state, dismissal, grouping by type, and a max count."**

```ts
interface Notification {
  id: string;
  type: 'info' | 'warning' | 'error';
  message: string;
  read: boolean;
  timestamp: number;
}

function useNotifications(wsUrl: string, maxCount = 50) {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const wsRef = useRef<WebSocket | null>(null);

  // Connect to WebSocket
  useEffect(() => {
    const ws = new WebSocket(wsUrl);
    wsRef.current = ws;

    ws.onmessage = (event) => {
      const notification: Notification = {
        ...JSON.parse(event.data),
        read: false,
        timestamp: Date.now(),
      };
      setNotifications(prev => [notification, ...prev].slice(0, maxCount));
    };

    ws.onclose = () => {
      // Reconnect after 3s
      setTimeout(() => { wsRef.current = new WebSocket(wsUrl); }, 3000);
    };

    return () => ws.close();
  }, [wsUrl, maxCount]);

  // Actions
  const markAsRead = useCallback((id: string) => {
    setNotifications(prev => prev.map(n => n.id === id ? { ...n, read: true } : n));
  }, []);

  const markAllRead = useCallback(() => {
    setNotifications(prev => prev.map(n => ({ ...n, read: true })));
  }, []);

  const dismiss = useCallback((id: string) => {
    setNotifications(prev => prev.filter(n => n.id !== id));
  }, []);

  // Derived state
  const unreadCount = useMemo(
    () => notifications.filter(n => !n.read).length,
    [notifications]
  );

  const groupedByType = useMemo(
    () => Object.groupBy(notifications, n => n.type),
    [notifications]
  );

  return {
    notifications,
    unreadCount,
    groupedByType,
    markAsRead,
    markAllRead,
    dismiss,
  };
}
```

### Cross-questions the interviewer would follow up with:

1. *"How do you handle reconnection with exponential backoff?"*
   → Track retry count in a ref, double the delay each time (max 30s)

2. *"What if the component unmounts and remounts — do you lose notifications?"*
   → Persist unread notifications in `localStorage` or lift the hook to a provider

3. *"How do you test this hook?"*
   → Mock WebSocket with a class, use `renderHook`, simulate `onmessage` events via `act()`

4. *"What about memory leaks if maxCount is removed?"*
   → Without `slice(0, maxCount)`, the array grows unbounded; answer: always cap + offer a "load older" feature via REST

---

## Summary — Hook Patterns at a Glance

| Pattern | Hooks | Purpose |
|---|---|---|
| **State wrapper** | `useToggle`, `useLocalStorage`, `useForm` | Enhance `useState` with extra logic |
| **Previous value** | `usePrevious`, `useLatest` | Track prior render state or avoid stale closures |
| **Time-based** | `useDebounce`, `useThrottle`, `useInterval` | Control update frequency |
| **DOM interaction** | `useOnClickOutside`, `useKeyPress`, `useLockBodyScroll` | Respond to browser events |
| **Browser APIs** | `useMediaQuery`, `useIntersectionObserver`, `usePermission`, `useCopyToClipboard` | Abstract browser APIs |
| **Async** | `useFetch`, `useAsync`, `useAbortController` | Manage loading/error/data lifecycle |
| **External stores** | `useSyncExternalStore`, `useEventListener` | Subscribe to non-React state |
| **Deep equality** | `useDeepCompareEffect` | Handle reference instability |
| **Middleware** | `useReducerWithMiddleware` | Add cross-cutting concerns to `useReducer` |

---

*Last updated: February 2026 | React 18 + React 19*
