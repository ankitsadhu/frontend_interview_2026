# Higher-Order Components (HOC) — MANG Level Interview Guide

> Patterns · Real-world code · HOC vs Hooks · Cross-questions · Scenarios

---

## Table of Contents

1. [What is a HOC?](#1-what-is-a-hoc)
2. [Basic HOC Anatomy](#2-basic-hoc-anatomy)
3. [Classic HOC Patterns](#3-classic-hoc-patterns)
4. [HOC vs Custom Hooks vs Render Props](#4-hoc-vs-custom-hooks-vs-render-props)
5. [HOC Pitfalls & How to Fix Them](#5-hoc-pitfalls--how-to-fix-them)
6. [Real-world Production HOCs](#6-real-world-production-hocs)
7. [HOC + TypeScript](#7-hoc--typescript)
8. [Testing HOCs](#8-testing-hocs)
9. [MANG Cross-Questions & Scenarios](#9-mang-cross-questions--scenarios)

---

## 1. What is a HOC?

### Q: Define a Higher-Order Component.

**A:** A HOC is a **function that takes a component and returns a new enhanced component**. It's a pattern for **reusing component logic** — not a React API.

```
const EnhancedComponent = withSomething(WrappedComponent);
```

It comes from the concept of **higher-order functions** in functional programming:
```js
// Higher-order function: takes a function, returns a function
const double = (fn) => (...args) => fn(...args) * 2;

// Higher-order component: takes a component, returns a component
const withLogger = (Component) => (props) => {
  console.log('Rendering:', Component.name, props);
  return <Component {...props} />;
};
```

**Key principle:** A HOC doesn't modify the original component. It **wraps** it, composing new behaviour via a container component.

### Cross-question: *"Name HOCs you've used from popular libraries."*

| Library | HOC | Purpose |
|---|---|---|
| React Redux | `connect(mapState, mapDispatch)(Component)` | Inject Redux state + dispatch as props |
| React Router (v5) | `withRouter(Component)` | Inject `history`, `location`, `match` |
| Material UI | `withStyles(styles)(Component)` | Inject CSS classes |
| React Intl | `injectIntl(Component)` | Inject `intl` object for i18n |
| Relay | `createFragmentContainer(Component, query)` | Inject GraphQL data |

> Most of these have been replaced by hooks (`useSelector`, `useNavigate`, `useIntl`), but HOCs still appear in legacy codebases and design systems.

---

## 2. Basic HOC Anatomy

### Q: Write a HOC from scratch. Explain each part.

```tsx
function withLoading(WrappedComponent) {
  // Return a NEW component (the "enhanced" or "container" component)
  function WithLoadingComponent({ isLoading, ...restProps }) {
    if (isLoading) return <Spinner />;
    return <WrappedComponent {...restProps} />;

    // Key points:
    // 1. Separate HOC-specific props (isLoading) from pass-through props (...restProps)
    // 2. Spread restProps into WrappedComponent — props transparency
    // 3. Conditionally render based on HOC logic
  }

  // Set display name for DevTools debugging
  WithLoadingComponent.displayName = `withLoading(${getDisplayName(WrappedComponent)})`;

  return WithLoadingComponent;
}

// Helper for readable DevTools names
function getDisplayName(Component) {
  return Component.displayName || Component.name || 'Component';
}

// Usage
const UserListWithLoading = withLoading(UserList);
<UserListWithLoading isLoading={loading} users={users} />
```

### Cross-question: *"Why set `displayName`?"*

Without it, React DevTools shows `<WithLoadingComponent>` for every HOC-wrapped component. With it, you see `<withLoading(UserList)>` — making debugging possible when you have 20+ HOCs in a tree.

---

## 3. Classic HOC Patterns

### Pattern 1: Props Injection

Inject data the wrapped component needs without it having to fetch it:

```tsx
function withUser(WrappedComponent) {
  return function WithUserComponent(props) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      api.getCurrentUser()
        .then(setUser)
        .finally(() => setLoading(false));
    }, []);

    if (loading) return <Spinner />;
    if (!user) return <LoginRedirect />;

    // Inject `user` as a prop — WrappedComponent doesn't need to know where it comes from
    return <WrappedComponent user={user} {...props} />;
  };
}

// Usage
const ProfilePage = ({ user, ...props }) => <h1>Hello, {user.name}</h1>;
export default withUser(ProfilePage);
```

---

### Pattern 2: Props Transformation

Modify or derive props before passing them:

```tsx
function withFormattedDate(WrappedComponent, dateKey = 'date') {
  return function WithFormattedDate(props) {
    const formatted = new Intl.DateTimeFormat('en-US', {
      year: 'numeric', month: 'long', day: 'numeric',
    }).format(new Date(props[dateKey]));

    return <WrappedComponent {...props} formattedDate={formatted} />;
  };
}

// Usage
const EventCard = ({ title, formattedDate }) => (
  <div><h3>{title}</h3><p>{formattedDate}</p></div>
);
export default withFormattedDate(EventCard, 'eventDate');
```

---

### Pattern 3: Conditional Rendering (Authorization)

```tsx
function withAuth(WrappedComponent, requiredRole = 'user') {
  return function WithAuthComponent(props) {
    const { user, isAuthenticated } = useAuth();

    if (!isAuthenticated) return <Navigate to="/login" />;
    if (user.role !== requiredRole && requiredRole !== 'user') {
      return <ForbiddenPage />;
    }

    return <WrappedComponent {...props} user={user} />;
  };
}

// Usage
const AdminDashboard = withAuth(Dashboard, 'admin');
const UserProfile = withAuth(Profile);
```

---

### Pattern 4: Lifecycle Interception (Analytics / Logging)

```tsx
function withAnalytics(WrappedComponent, pageName) {
  return function WithAnalyticsComponent(props) {
    useEffect(() => {
      analytics.trackPageView(pageName);
      const startTime = Date.now();

      return () => {
        analytics.trackTimeSpent(pageName, Date.now() - startTime);
      };
    }, []);

    return <WrappedComponent {...props} />;
  };
}

// Usage
export default withAnalytics(CheckoutPage, 'checkout');
```

---

### Pattern 5: State Abstraction

```tsx
function withToggle(WrappedComponent) {
  return function WithToggleComponent(props) {
    const [isOn, setIsOn] = useState(false);
    const toggle = useCallback(() => setIsOn(prev => !prev), []);

    return <WrappedComponent {...props} isOn={isOn} toggle={toggle} />;
  };
}

// Usage
const ExpandablePanel = ({ title, children, isOn, toggle }) => (
  <div>
    <button onClick={toggle}>{title} {isOn ? '▼' : '▶'}</button>
    {isOn && <div>{children}</div>}
  </div>
);
export default withToggle(ExpandablePanel);
```

---

### Pattern 6: Composition (Stacking HOCs)

```tsx
// Composing multiple HOCs
const EnhancedDashboard = withAuth(
  withAnalytics(
    withTheme(
      withErrorBoundary(Dashboard)
    ), 'dashboard'
  ), 'admin'
);

// ❌ Problem: deeply nested, hard to read ("wrapper hell")

// ✅ Better: use a compose utility (from lodash, Redux, or custom)
import { compose } from 'redux'; // or write your own

const enhance = compose(
  withAuth('admin'),
  withAnalytics('dashboard'),
  withTheme,
  withErrorBoundary,
);
const EnhancedDashboard = enhance(Dashboard);
```

```ts
// Simple compose implementation
function compose(...fns) {
  return (component) => fns.reduceRight((acc, fn) => fn(acc), component);
}
```

### Cross-question: *"What's the order of execution in compose?"*

`compose(f, g, h)(x)` = `f(g(h(x)))` — right to left. So in the example above:
1. `withErrorBoundary(Dashboard)` runs first (innermost wrapper)
2. `withTheme(...)` wraps that
3. `withAnalytics('dashboard')(...)` wraps that
4. `withAuth('admin')(...)` is the outermost wrapper

The **outermost HOC's logic runs first** during rendering, but the **innermost HOC wraps closest** to the actual component.

---

## 4. HOC vs Custom Hooks vs Render Props

### Q: When would you use a HOC over a custom hook? Are HOCs dead?

| | HOC | Custom Hook | Render Props |
|---|---|---|---|
| **Reuses** | Structure + logic + conditional rendering | Logic only | Logic + flexible rendering |
| **Returns** | A new component | Values / functions | JSX via render function |
| **Props** | Injected automatically | Destructured manually | Passed via callback |
| **Composition** | `compose(hoc1, hoc2)(Component)` | Call multiple hooks | Nesting render callbacks |
| **TypeScript** | Complex prop type inference | Easy to type | Easy to type |
| **DevTools** | Extra wrapper nodes | No extra nodes | Extra wrapper nodes |
| **Still used?** | Yes, for cross-cutting concerns | Primary pattern | Rare (replaced by hooks) |

### When HOCs still win over hooks:

1. **You need to wrap the entire JSX output** (error boundaries, Suspense wrappers, providers)
2. **3rd-party integration** — some libraries only support HOC pattern
3. **Class components** — hooks don't work in classes, but HOCs do
4. **Cross-cutting concerns** that should be invisible to the component (logging, analytics, auth guards)
5. **Conditional rendering** — preventing the wrapped component from rendering at all (auth, feature flags)

```tsx
// Hook can't prevent rendering — the component must render first, then decide
function Dashboard() {
  const { isAuthed } = useAuth(); // component already rendered
  if (!isAuthed) return <Navigate to="/login" />;
  return <div>...</div>;
}

// HOC can prevent rendering before the wrapped component mounts
const Dashboard = withAuth(DashboardContent); // DashboardContent never mounts if not authed
```

### Cross-question: *"Can you convert any HOC to a custom hook?"*

**Logic-only HOCs → yes.** If the HOC only injects data/state, a hook does the same thing better:

```tsx
// HOC
const withUser = (C) => (props) => { const user = useAuth(); return <C {...props} user={user} />; };
export default withUser(Profile);

// Equivalent hook (simpler)
function Profile() { const user = useAuth(); return <h1>{user.name}</h1>; }
```

**Structural HOCs → no.** If the HOC wraps with providers, error boundaries, or conditionally renders, hooks can't replicate that because hooks run **inside** the component — they can't prevent it from mounting.

---

## 5. HOC Pitfalls & How to Fix Them

### Pitfall 1: Eating refs

```tsx
// ❌ ref points to the HOC wrapper, NOT the inner component
function withTooltip(WrappedComponent) {
  return function WithTooltip(props) {
    return (
      <div className="tooltip-wrapper">
        <WrappedComponent {...props} />
      </div>
    );
  };
}
const EnhancedInput = withTooltip(TextInput);
const ref = useRef();
<EnhancedInput ref={ref} /> // ref → div.tooltip-wrapper, not TextInput!
```

**Fix: `React.forwardRef`**

```tsx
function withTooltip(WrappedComponent) {
  const WithTooltip = React.forwardRef((props, ref) => {
    return (
      <div className="tooltip-wrapper">
        <WrappedComponent {...props} ref={ref} />
      </div>
    );
  });
  WithTooltip.displayName = `withTooltip(${getDisplayName(WrappedComponent)})`;
  return WithTooltip;
}
```

> **React 19 note:** `forwardRef` is deprecated. `ref` becomes a regular prop — so this pitfall disappears.

---

### Pitfall 2: Static methods are lost

```tsx
// Original component has a static method
UserList.fetchData = () => api.getUsers();

// After wrapping, static method is gone
const EnhancedUserList = withLoading(UserList);
EnhancedUserList.fetchData; // undefined!
```

**Fix: `hoist-non-react-statics`**

```tsx
import hoistNonReactStatics from 'hoist-non-react-statics';

function withLoading(WrappedComponent) {
  function WithLoading(props) { /* ... */ }
  hoistNonReactStatics(WithLoading, WrappedComponent); // copies static methods
  return WithLoading;
}
```

**Or manually copy:**
```tsx
WithLoading.fetchData = WrappedComponent.fetchData;
```

---

### Pitfall 3: Props collision

```tsx
// HOC injects a prop called "data"
function withData(WrappedComponent) {
  return (props) => <WrappedComponent {...props} data={fetchedData} />;
}

// Another HOC also injects "data"
function withCache(WrappedComponent) {
  return (props) => <WrappedComponent {...props} data={cachedData} />;
}

// ❌ Second "data" overwrites the first!
const Enhanced = withCache(withData(MyComponent));
// MyComponent receives cachedData — fetchedData is lost
```

**Fix:** Use unique/namespaced prop names:

```tsx
function withData(WrappedComponent) {
  return (props) => <WrappedComponent {...props} apiData={fetchedData} />;
}

function withCache(WrappedComponent) {
  return (props) => <WrappedComponent {...props} cacheData={cachedData} />;
}
```

---

### Pitfall 4: Re-creating HOC inside render

```tsx
// ❌ Creates a NEW component type on every render — destroys and remounts
function App() {
  const EnhancedList = withLoading(UserList); // new component every render!
  return <EnhancedList isLoading={loading} />;
}

// ✅ Create HOC OUTSIDE the component (module level)
const EnhancedList = withLoading(UserList);

function App() {
  return <EnhancedList isLoading={loading} />;
}
```

### Cross-question: *"Why does creating HOC inside render cause problems?"*

React uses the **component type** (function reference) to identify component instances. If the type changes every render, React sees it as a **completely different component** → unmounts the old, mounts a new one → all state is lost, effects re-run, and performance tanks.

---

### Pitfall 5: Wrapper hell in DevTools

```
<withAuth(withAnalytics(withTheme(withErrorBoundary(Dashboard))))>
  <withAnalytics(withTheme(withErrorBoundary(Dashboard)))>
    <withTheme(withErrorBoundary(Dashboard))>
      <withErrorBoundary(Dashboard)>
        <Dashboard />
```

**Fix:** Set `displayName` on every HOC + use React DevTools "Components" tab filter to hide HOC wrappers.

---

## 6. Real-world Production HOCs

### HOC 1: `withErrorBoundary`

Error boundaries **must** be class components — so this is a case where HOCs can't be replaced by hooks:

```tsx
function withErrorBoundary(WrappedComponent, FallbackComponent = DefaultErrorFallback) {
  class ErrorBoundaryWrapper extends React.Component {
    state = { hasError: false, error: null };

    static getDerivedStateFromError(error) {
      return { hasError: true, error };
    }

    componentDidCatch(error, errorInfo) {
      errorReporting.captureException(error, { extra: errorInfo });
    }

    render() {
      if (this.state.hasError) {
        return <FallbackComponent error={this.state.error} reset={() => this.setState({ hasError: false })} />;
      }
      return <WrappedComponent {...this.props} />;
    }
  }

  ErrorBoundaryWrapper.displayName = `withErrorBoundary(${getDisplayName(WrappedComponent)})`;
  return ErrorBoundaryWrapper;
}

// Usage
const SafeDashboard = withErrorBoundary(Dashboard, DashboardError);
```

### Cross-question: *"Can you use a hook for error boundaries?"*

**No.** `getDerivedStateFromError` and `componentDidCatch` have no hook equivalents. Error boundaries are the **one remaining class-only feature** in React. A HOC is the cleanest way to add them to function components.

---

### HOC 2: `withFeatureFlag`

```tsx
function withFeatureFlag(WrappedComponent, flagName, FallbackComponent = null) {
  return function WithFeatureFlag(props) {
    const isEnabled = useFeatureFlag(flagName);

    if (!isEnabled) {
      return FallbackComponent ? <FallbackComponent {...props} /> : null;
    }

    return <WrappedComponent {...props} />;
  };
}

// Usage
const NewCheckout = withFeatureFlag(NewCheckoutFlow, 'new_checkout', LegacyCheckout);
```

---

### HOC 3: `withSuspense`

```tsx
function withSuspense(WrappedComponent, fallback = <Spinner />) {
  return function WithSuspense(props) {
    return (
      <Suspense fallback={fallback}>
        <WrappedComponent {...props} />
      </Suspense>
    );
  };
}

// Usage with lazy loading
const LazyEditor = React.lazy(() => import('./Editor'));
const Editor = withSuspense(LazyEditor, <EditorSkeleton />);
```

---

### HOC 4: `withPerformanceTracking`

```tsx
function withPerformanceTracking(WrappedComponent, componentName) {
  return function WithPerformanceTracking(props) {
    const renderCount = useRef(0);
    const mountTime = useRef(performance.now());

    useEffect(() => {
      const duration = performance.now() - mountTime.current;
      metrics.recordMountTime(componentName, duration);
    }, []);

    useEffect(() => {
      renderCount.current += 1;
      if (renderCount.current > 50) {
        console.warn(`${componentName} rendered ${renderCount.current} times — possible perf issue`);
      }
    });

    return <WrappedComponent {...props} />;
  };
}

// Usage
export default withPerformanceTracking(ProductList, 'ProductList');
```

---

### HOC 5: `withThemeVariant` (Design System)

```tsx
function withThemeVariant(WrappedComponent, defaultVariant = 'light') {
  return function WithThemeVariant({ variant = defaultVariant, ...props }) {
    const theme = variant === 'dark' ? darkTheme : lightTheme;

    return (
      <ThemeContext.Provider value={theme}>
        <WrappedComponent {...props} />
      </ThemeContext.Provider>
    );
  };
}

// Usage — self-contained themed widget
const DarkModal = withThemeVariant(Modal, 'dark');
const LightCard = withThemeVariant(Card, 'light');
```

---

## 7. HOC + TypeScript

### Q: How do you type a HOC properly?

This is a common MANG TypeScript question — HOCs have notoriously tricky types.

```tsx
// Generic HOC with proper type inference
function withLoading<P extends object>(WrappedComponent: React.ComponentType<P>) {
  // The HOC adds its own props (isLoading)
  type HOCProps = P & { isLoading: boolean };

  function WithLoadingComponent({ isLoading, ...restProps }: HOCProps) {
    if (isLoading) return <Spinner />;
    return <WrappedComponent {...(restProps as P)} />;
  }

  WithLoadingComponent.displayName = `withLoading(${getDisplayName(WrappedComponent)})`;
  return WithLoadingComponent;
}

// Usage — TypeScript infers P from UserList's props
const UserListWithLoading = withLoading(UserList);
// <UserListWithLoading isLoading={true} users={users} />
//                      ^^^^^^^^^^^^^ from HOC   ^^^^^^^ from UserList props
```

### Injected props (HOC provides, component receives, consumer doesn't need to pass):

```tsx
interface InjectedProps {
  user: User;
}

function withUser<P extends InjectedProps>(WrappedComponent: React.ComponentType<P>) {
  // Consumer sees P without InjectedProps
  type ExternalProps = Omit<P, keyof InjectedProps>;

  function WithUserComponent(props: ExternalProps) {
    const user = useAuth();
    return <WrappedComponent {...(props as P)} user={user} />;
  }

  WithUserComponent.displayName = `withUser(${getDisplayName(WrappedComponent)})`;
  return WithUserComponent;
}

// Component declares it receives user
function Profile({ user, bio }: { user: User; bio: string }) {
  return <h1>{user.name}: {bio}</h1>;
}

// Consumer only needs to pass `bio` — `user` is injected by HOC
const ProfileWithUser = withUser(Profile);
<ProfileWithUser bio="Engineer" /> // ✅ user is auto-injected
```

---

## 8. Testing HOCs

### Testing strategy: test the enhanced component, not the HOC in isolation

```tsx
// withAuth.test.tsx
import { render, screen } from '@testing-library/react';
import { withAuth } from './withAuth';

// Mock useAuth
vi.mock('../hooks/useAuth', () => ({
  useAuth: vi.fn(),
}));
import { useAuth } from '../hooks/useAuth';

const MockComponent = ({ user }) => <div>Hello, {user.name}</div>;
const ProtectedComponent = withAuth(MockComponent);

describe('withAuth HOC', () => {
  it('renders wrapped component when authenticated', () => {
    vi.mocked(useAuth).mockReturnValue({
      isAuthenticated: true,
      user: { name: 'Alice', role: 'admin' },
    });

    render(<ProtectedComponent />);
    expect(screen.getByText('Hello, Alice')).toBeInTheDocument();
  });

  it('redirects to login when not authenticated', () => {
    vi.mocked(useAuth).mockReturnValue({
      isAuthenticated: false,
      user: null,
    });

    render(<ProtectedComponent />, { wrapper: MemoryRouter });
    // WrappedComponent should NOT render
    expect(screen.queryByText(/Hello/)).not.toBeInTheDocument();
  });

  it('passes through additional props', () => {
    vi.mocked(useAuth).mockReturnValue({
      isAuthenticated: true,
      user: { name: 'Alice', role: 'user' },
    });

    const TestComponent = ({ title, user }) => <h1>{title} - {user.name}</h1>;
    const Protected = withAuth(TestComponent);
    render(<Protected title="Dashboard" />);
    expect(screen.getByText('Dashboard - Alice')).toBeInTheDocument();
  });

  it('sets displayName for debugging', () => {
    expect(ProtectedComponent.displayName).toBe('withAuth(MockComponent)');
  });
});
```

---

## 9. MANG Cross-Questions & Scenarios

---

### Scenario 1: Design a `withRetry` HOC

> **"A component shows user data fetched via a passed-in `onLoad` prop. Design a HOC that automatically retries the load on failure with exponential backoff."**

```tsx
function withRetry(WrappedComponent, maxRetries = 3) {
  return function WithRetry({ onLoad, ...props }) {
    const [error, setError] = useState(null);
    const [retryCount, setRetryCount] = useState(0);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      let cancelled = false;
      const attempt = async (attempt) => {
        try {
          setLoading(true);
          setError(null);
          await onLoad();
          if (!cancelled) setLoading(false);
        } catch (err) {
          if (cancelled) return;
          if (attempt < maxRetries) {
            const delay = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
            setTimeout(() => {
              if (!cancelled) {
                setRetryCount(attempt + 1);
                attempt(attempt + 1);
              }
            }, delay);
          } else {
            setError(err);
            setLoading(false);
          }
        }
      };
      attempt(0);
      return () => { cancelled = true; };
    }, [onLoad]);

    if (error) return <ErrorFallback error={error} onRetry={() => setRetryCount(0)} />;
    if (loading) return <Spinner retryCount={retryCount} />;
    return <WrappedComponent {...props} />;
  };
}

export default withRetry(UserProfile, 3);
```

### Cross-question: *"What if the user navigates away during a retry delay?"*

The `cancelled` flag prevents state updates after unmount. But the timeout is still pending (wasted resource). Better: use `AbortController` + `clearTimeout` in the cleanup.

---

### Scenario 2: HOC Ordering Matters

> **"You have `withAuth`, `withAnalytics`, and `withErrorBoundary`. What order should they be applied?"**

```tsx
// ❌ Wrong order
const Enhanced = withAuth(withErrorBoundary(withAnalytics(Dashboard)));
// Problem: if withAuth redirects to login, analytics still tracks a page view
// Problem: error boundary is INSIDE auth — if auth crashes, nothing catches it

// ✅ Correct order (outermost runs FIRST)
const Enhanced = withErrorBoundary(withAnalytics(withAuth(Dashboard)));
// 1. ErrorBoundary catches ANY crash (including auth/analytics)
// 2. Analytics tracks page view
// 3. Auth checks permissions → renders Dashboard or redirects
```

**Rule of thumb (inside → out):**
1. Feature logic (closest to component)
2. Auth / permissions
3. Analytics / logging
4. Error boundary (outermost — catch everything)

---

### Scenario 3: Migrate `connect()` to Hooks

> **"You have a legacy codebase with 200 uses of `connect()`. How would you migrate?"**

```tsx
// Before (Redux connect HOC)
const mapStateToProps = (state) => ({
  user: state.user.data,
  isLoading: state.user.status === 'loading',
});
const mapDispatchToProps = { fetchUser };
export default connect(mapStateToProps, mapDispatchToProps)(UserProfile);

// After (hooks — no HOC needed)
function UserProfile() {
  const user = useAppSelector(state => state.user.data);
  const isLoading = useAppSelector(state => state.user.status === 'loading');
  const dispatch = useAppDispatch();

  useEffect(() => { dispatch(fetchUser()); }, [dispatch]);

  return /* same JSX */;
}
```

**Migration strategy:**
1. **Don't do a big bang** — migrate one component at a time
2. Start with **leaf components** (no children using the connected props)
3. Keep `connect()` working alongside hooks — they coexist
4. Add a lint rule to flag new uses of `connect()`
5. Track migration progress: `grep -r "connect(" src/ | wc -l`

### Cross-question: *"Are there cases where keeping `connect()` is better?"*

Yes:
- **`connect()` implements `React.memo` by default** — `useSelector` doesn't (component must be wrapped manually)
- **Shallow equality** — `connect()` does shallow comparison of all mapped props automatically
- **Class components** — can't use hooks, must keep `connect()`

---

### Scenario 4: HOC for A/B Testing

> **"Design a HOC that renders different component variants based on an A/B test assignment."**

```tsx
function withABTest(variants, testName) {
  // variants = { control: ControlComponent, variantA: VariantAComponent }
  return function WithABTest(props) {
    const [variant, setVariant] = useState('control');

    useEffect(() => {
      const assignment = abTestService.getAssignment(testName); // 'control' | 'variantA'
      setVariant(assignment);
      analytics.track('ab_test_exposure', { test: testName, variant: assignment });
    }, []);

    const Component = variants[variant] || variants.control;
    return <Component {...props} abVariant={variant} />;
  };
}

// Usage
const CheckoutPage = withABTest({
  control: ClassicCheckout,
  variantA: StreamlinedCheckout,
  variantB: OneClickCheckout,
}, 'checkout_flow_2026');
```

### Cross-question: *"Why HOC over a hook here?"*

Because the variants are **entirely different components** — not just different data. A hook can return which variant is assigned, but the HOC cleanly handles the **component switching** + analytics tracking in one place.

---

### Scenario 5: When NOT to Use a HOC

> **"Your team is debating using a HOC vs a hook for sharing form validation logic. Take a position."**

**Answer: Use a hook.**

```tsx
// ❌ HOC — overkill for pure logic
const ValidatedForm = withValidation(SignupForm, signupSchema);
// Problems:
// - SignupForm can't control WHEN validation runs
// - Validation errors are injected as props — unclear naming
// - Can't validate individual fields

// ✅ Custom hook — component controls everything
function SignupForm() {
  const { validate, errors, isValid } = useValidation(signupSchema);

  const handleSubmit = (data) => {
    if (validate(data)) submit(data); // component decides when to validate
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" />
      {errors.email && <span>{errors.email}</span>}
    </form>
  );
}
```

**Use HOC when:** The concern is **structural** (wrapping, conditional rendering, providers)
**Use hook when:** The concern is **logical** (data, state, callbacks the component needs to control)

---

### Decision Framework (Interview Cheat Sheet)

```
Should I use a HOC?
│
├─ Does it WRAP the component (error boundary, Suspense, Provider)?
│  └─ YES → HOC ✅
│
├─ Does it PREVENT rendering (auth, feature flag)?
│  └─ YES → HOC ✅ (hooks can't block rendering)
│
├─ Does it need to work with CLASS components?
│  └─ YES → HOC ✅ (hooks don't work in classes)
│
├─ Is it INVISIBLE to the component (analytics, logging)?
│  └─ YES → HOC ✅ (component shouldn't know about it)
│
├─ Does it provide LOGIC the component controls?
│  └─ YES → Custom Hook ✅
│
└─ Does the component need FLEXIBLE rendering?
   └─ YES → Render Props or Custom Hook ✅
```

---

*Last updated: February 2026 | React 18 + React 19*
