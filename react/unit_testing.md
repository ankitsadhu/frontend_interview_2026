# React Unit Testing — Complete Guide
> **Tools covered:** Jest · Vitest · React Testing Library (RTL)  
> **Target:** MANG-level interviews + day-to-day production testing

---

## Table of Contents

1. [Tool Comparison — Jest vs Vitest vs RTL](#1-tool-comparison)
2. [Setup & Configuration](#2-setup--configuration)
3. [Testing Fundamentals — Jest / Vitest API](#3-testing-fundamentals)
4. [React Testing Library — Core Concepts](#4-react-testing-library--core-concepts)
5. [Querying the DOM](#5-querying-the-dom)
6. [User Interactions — `userEvent` vs `fireEvent`](#6-user-interactions)
7. [Async Testing](#7-async-testing)
8. [Mocking](#8-mocking)
9. [Testing Hooks](#9-testing-hooks)
10. [Testing Context](#10-testing-context)
11. [Testing Forms](#11-testing-forms)
12. [Testing API Calls](#12-testing-api-calls)
13. [Snapshot Testing](#13-snapshot-testing)
14. [Code Coverage](#14-code-coverage)
15. [Common Patterns & Best Practices](#15-common-patterns--best-practices)
16. [MANG Interview Q&A](#16-mang-interview-qa)

---

## 1. Tool Comparison

| Feature | Jest | Vitest | React Testing Library |
|---|---|---|---|
| **Type** | Test runner + assertion + mocking | Test runner + assertion + mocking | DOM testing utilities |
| **Speed** | Moderate (CommonJS transform) | Very fast (native ESM, esbuild) | N/A (used with Jest or Vitest) |
| **Config** | Separate jest.config.js | Inside vite.config.ts | Minimal config |
| **Watch mode** | `jest --watch` | `vitest` (watch by default) | N/A |
| **Used with** | CRA, custom webpack setups | Vite-based projects | Both |
| **Snapshots** | Built-in | Built-in | Can be used |
| **Coverage** | `jest --coverage` (istanbul) | `vitest --coverage` (v8 or istanbul) | N/A |
| **JSDOM** | jest-environment-jsdom | `environment: 'jsdom'` in config | Always DOM-based |

> **Rule of thumb:** New Vite/Next.js project → **Vitest + RTL**. Existing CRA or webpack project → **Jest + RTL**. RTL is framework-agnostic and works with both.

---

## 2. Setup & Configuration

### 2a. Vitest + RTL Setup (Vite project)

```bash
npm install -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',           // simulate browser DOM
    globals: true,                  // no need to import describe/it/expect
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
    },
  },
});
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom'; // adds matchers: toBeInTheDocument, toHaveValue, etc.
```

```json
// package.json scripts
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",        // single run (CI)
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"         // browser-based test UI
  }
}
```

---

### 2b. Jest + RTL Setup (CRA / Webpack)

```bash
npm install -D jest @testing-library/react @testing-library/user-event @testing-library/jest-dom
npm install -D jest-environment-jsdom babel-jest @babel/preset-env @babel/preset-react
```

```js
// jest.config.js
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterFramework: ['<rootDir>/src/test/setup.js'],
  moduleNameMapper: {
    '\\.(css|scss)$': 'identity-obj-proxy',  // mock CSS imports
    '\\.(png|jpg|svg)$': '<rootDir>/__mocks__/fileMock.js',
  },
  transform: {
    '^.+\\.[jt]sx?$': 'babel-jest',
  },
  collectCoverageFrom: ['src/**/*.{ts,tsx}', '!src/**/*.d.ts'],
};
```

```js
// src/test/setup.js
import '@testing-library/jest-dom';
```

---

### 2c. TypeScript Support

```bash
npm install -D @types/jest  # for Jest
# Vitest has built-in TypeScript support
```

```ts
// tsconfig.json — include jest-dom types
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  }
}
```

---

## 3. Testing Fundamentals

### Test File Structure

```ts
// Component.test.tsx  OR  Component.spec.tsx
import { render, screen } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {            // group related tests
  it('renders the label', () => {     // individual test
    render(<Button label="Save" />);
    expect(screen.getByText('Save')).toBeInTheDocument();
  });

  it('calls onClick when clicked', async () => {
    const handleClick = vi.fn();      // jest.fn() in Jest
    render(<Button label="Save" onClick={handleClick} />);
    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledOnce();
  });
});
```

### Core Vitest / Jest Globals

```ts
// Test organisation
describe('GroupName', () => { ... })
it('test name', () => { ... })         // alias: test()
it.only('run only this', () => { ... }) // alias: fit()
it.skip('skip this test', () => { ... })
it.todo('implement me later')

// Lifecycle hooks
beforeAll(() => { /* run once before all tests in describe */ })
afterAll(() => { /* run once after all tests */ })
beforeEach(() => { /* run before EACH test */ })
afterEach(() => { /* run after EACH test */ })

// Assertions (Jest/Vitest-compatible)
expect(value).toBe(42)                  // strict equality (Object.is)
expect(value).toEqual({ a: 1 })        // deep equality
expect(value).toBeTruthy()
expect(value).toBeFalsy()
expect(value).toBeNull()
expect(value).toBeUndefined()
expect(value).toContain('substring')
expect(array).toHaveLength(3)
expect(fn).toThrow('error message')
expect(fn).toHaveBeenCalled()
expect(fn).toHaveBeenCalledWith('arg1', 'arg2')
expect(fn).toHaveBeenCalledTimes(2)
expect(promise).resolves.toBe('value')
expect(promise).rejects.toThrow('error')

// @testing-library/jest-dom matchers
expect(el).toBeInTheDocument()
expect(el).toBeVisible()
expect(el).toBeDisabled()
expect(el).toHaveValue('hello')
expect(el).toHaveTextContent('Save')
expect(el).toHaveClass('btn-primary')
expect(el).toHaveFocus()
expect(el).toHaveAttribute('href', '/home')
expect(el).toBeChecked()
```

---

## 4. React Testing Library — Core Concepts

### The RTL Philosophy

> **"The more your tests resemble the way your software is used, the more confidence they can give you."** — Kent C. Dodds

RTL renders components into a real (JSDOM) DOM and encourages querying elements the way **users** see them — by visible text, role, label — not by CSS classes or component internals.

```
❌ Avoid (implementation details):
- Finding by CSS class: .btn-primary
- Finding by component state
- Testing internal methods
- Shallow rendering (Enzyme-style)

✅ Prefer (user-visible behaviour):
- Finding by role, label, text
- Testing what users see and do
- Testing outcomes, not how they happen
```

### Basic Render

```tsx
import { render, screen } from '@testing-library/react';

// render() returns: { container, getByText, queryByRole, ... }
// but prefer screen.* — always queries the full document
render(<MyComponent prop="value" />);

// screen has all query methods
screen.getByText('Hello');
screen.getByRole('button', { name: 'Submit' });
screen.queryByText('Error'); // returns null if not found (won't throw)
```

### `render` return value and `rerender`

```tsx
const { rerender, unmount } = render(<Counter count={0} />);

expect(screen.getByText('Count: 0')).toBeInTheDocument();

// Simulate prop change
rerender(<Counter count={5} />);
expect(screen.getByText('Count: 5')).toBeInTheDocument();

// Simulate unmount (test cleanup effects)
unmount();
```

---

## 5. Querying the DOM

### Query Priority (use in this order)

| Priority | Query | When to use |
|---|---|---|
| **1** | `getByRole` | Most elements — buttons, inputs, headings, links |
| **2** | `getByLabelText` | Form inputs with `<label>` |
| **3** | `getByPlaceholderText` | Input with placeholder |
| **4** | `getByText` | Non-interactive text elements |
| **5** | `getByDisplayValue` | Current value of input/select |
| **6** | `getByAltText` | Images |
| **7** | `getByTitle` | Element with `title` attribute |
| **8** | `getByTestId` | Last resort — add `data-testid` to element |

### `getBy` vs `queryBy` vs `findBy`

| Prefix | Returns | Throws if not found | Async |
|---|---|---|---|
| `getBy` | Single element | Yes (synchronously) | No |
| `getAllBy` | Array | Yes (if empty) | No |
| `queryBy` | Element or `null` | No | No |
| `queryAllBy` | Array (empty OK) | No | No |
| `findBy` | Promise → element | Yes (after timeout) | **Yes** |
| `findAllBy` | Promise → array | Yes | **Yes** |

```tsx
// getBy — asserts element EXISTS, throws if not
const btn = screen.getByRole('button', { name: 'Submit' });

// queryBy — for asserting element does NOT exist
expect(screen.queryByText('Error message')).not.toBeInTheDocument();

// findBy — wait for async appearance
const toast = await screen.findByText('Saved!'); // waits up to 1000ms
```

### Role-based querying (most powerful)

```tsx
// Common ARIA roles automatically inferred from HTML
screen.getByRole('button', { name: 'Save' })       // <button>Save</button>
screen.getByRole('link', { name: 'Home' })         // <a href="/">Home</a>
screen.getByRole('textbox', { name: 'Email' })     // <input> with label "Email"
screen.getByRole('checkbox', { name: 'Remember' }) // <input type="checkbox">
screen.getByRole('combobox', { name: 'Country' })  // <select>
screen.getByRole('heading', { name: 'Dashboard', level: 1 }) // <h1>
screen.getByRole('dialog', { name: 'Confirm' })    // <div role="dialog" aria-label="Confirm">
screen.getByRole('alert')                          // <div role="alert"> — error messages
screen.getByRole('progressbar')                    // loading spinners
screen.getByRole('img', { name: 'Profile photo' }) // <img alt="Profile photo">
```

---

## 6. User Interactions

### `userEvent` vs `fireEvent`

| | `userEvent` | `fireEvent` |
|---|---|---|
| Simulates | Real user behaviour (full event chain) | Single DOM event |
| Typing | Characters one by one, triggers keyboard events | Just sets value |
| Click | pointerover → pointerenter → mouseover → click → focus | Just `click` event |
| Async | Returns Promise — must `await` | Synchronous |
| Recommended | ✅ Always prefer | Use only for events userEvent doesn't support |

```tsx
import userEvent from '@testing-library/user-event';

// SETUP — call setup() once per test
const user = userEvent.setup();

it('types into a text field', async () => {
  render(<SearchInput />);
  const input = screen.getByRole('textbox', { name: 'Search' });
  
  await user.type(input, 'react hooks');
  expect(input).toHaveValue('react hooks');
});

it('clears and retyoes', async () => {
  const user = userEvent.setup();
  render(<Input defaultValue="old" />);
  const input = screen.getByRole('textbox');
  
  await user.clear(input);
  await user.type(input, 'new value');
  expect(input).toHaveValue('new value');
});

it('selects from a dropdown', async () => {
  render(<CountrySelect />);
  await user.selectOptions(
    screen.getByRole('combobox', { name: 'Country' }),
    'India'
  );
  expect(screen.getByRole('option', { name: 'India' })).toBeSelected();
});

it('checks a checkbox', async () => {
  render(<TermsCheckbox />);
  const checkbox = screen.getByRole('checkbox', { name: 'Accept terms' });
  expect(checkbox).not.toBeChecked();
  
  await user.click(checkbox);
  expect(checkbox).toBeChecked();
});

it('uploads a file', async () => {
  render(<FileUpload />);
  const file = new File(['hello'], 'hello.png', { type: 'image/png' });
  const input = screen.getByLabelText('Upload file');
  
  await user.upload(input, file);
  expect(input.files[0]).toBe(file);
  expect(screen.getByText('hello.png')).toBeInTheDocument();
});

it('uses keyboard navigation', async () => {
  render(<Menu />);
  await user.tab(); // focus first focusable element
  await user.keyboard('{Enter}'); // press Enter
  await user.keyboard('{Escape}'); // press Escape
});
```

---

## 7. Async Testing

### Pattern 1 — `findBy` (wait for element to appear)

```tsx
it('shows data after API call', async () => {
  // Mock the fetch
  global.fetch = vi.fn().mockResolvedValue({
    json: () => Promise.resolve([{ id: 1, name: 'Alice' }]),
  });

  render(<UserList />);

  // Loading state
  expect(screen.getByText('Loading...')).toBeInTheDocument();

  // Wait for data to appear — findBy waits up to 1000ms by default
  const alice = await screen.findByText('Alice');
  expect(alice).toBeInTheDocument();

  // Loading spinner gone
  expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
});
```

### Pattern 2 — `waitFor` (wait for assertion to pass)

```tsx
import { waitFor } from '@testing-library/react';

it('shows error message on failed submit', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  await user.click(screen.getByRole('button', { name: 'Login' }));

  // waitFor retries the assertion until it passes (or times out)
  await waitFor(() => {
    expect(screen.getByRole('alert')).toHaveTextContent('Email is required');
  });
});
```

### Pattern 3 — `waitForElementToBeRemoved`

```tsx
import { waitForElementToBeRemoved } from '@testing-library/react';

it('hides skeleton after data loads', async () => {
  render(<Dashboard />);

  // Wait for the skeleton to disappear
  await waitForElementToBeRemoved(() => screen.queryByTestId('skeleton'));

  expect(screen.getByText('Welcome back!')).toBeInTheDocument();
});
```

### Configuring timeouts

```tsx
// Default timeout is 1000ms
await screen.findByText('Done', {}, { timeout: 3000 });

await waitFor(
  () => expect(screen.getByText('Done')).toBeInTheDocument(),
  { timeout: 3000, interval: 100 }
);
```

---

## 8. Mocking

### 8a. Mocking Functions

```ts
// Vitest
const mockFn = vi.fn();
const mockFn = vi.fn(() => 'return value');
const mockFn = vi.fn().mockReturnValue('value');
const mockFn = vi.fn().mockResolvedValue({ data: 'ok' }); // async
const mockFn = vi.fn().mockRejectedValue(new Error('fail')); // async error
const mockFn = vi.fn().mockImplementation((x) => x * 2);

// Jest equivalents — same API but vi → jest
const mockFn = jest.fn();

// Assertions on mock calls
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(3);
expect(mockFn).toHaveBeenCalledWith('arg1', { key: 'value' });
expect(mockFn).toHaveBeenLastCalledWith('last arg');
expect(mockFn).toHaveReturnedWith('result');

// Reset between tests
mockFn.mockClear();     // clears call history
mockFn.mockReset();     // clears + removes implementation
mockFn.mockRestore();   // restores original (only for vi.spyOn)
```

### 8b. Mocking Modules

```ts
// Vitest — vi.mock() is hoisted to top of file automatically
vi.mock('./api/users', () => ({
  fetchUsers: vi.fn().mockResolvedValue([{ id: 1, name: 'Alice' }]),
  createUser: vi.fn().mockResolvedValue({ id: 2, name: 'Bob' }),
}));

// Jest equivalent
jest.mock('./api/users', () => ({
  fetchUsers: jest.fn().mockResolvedValue([{ id: 1, name: 'Alice' }]),
}));

// Change mock implementation per test
import { fetchUsers } from './api/users';

it('handles empty list', async () => {
  vi.mocked(fetchUsers).mockResolvedValueOnce([]);
  render(<UserList />);
  await screen.findByText('No users found');
});

it('handles error', async () => {
  vi.mocked(fetchUsers).mockRejectedValueOnce(new Error('Network error'));
  render(<UserList />);
  await screen.findByRole('alert');
});
```

### 8c. Mocking React Router

```tsx
import { MemoryRouter, Route, Routes } from 'react-router-dom';

// Wrap component in a router for testing
function renderWithRouter(ui: React.ReactElement, { route = '/' } = {}) {
  return render(
    <MemoryRouter initialEntries={[route]}>
      <Routes>
        <Route path="*" element={ui} />
      </Routes>
    </MemoryRouter>
  );
}

it('renders profile page for /profile/:id', () => {
  renderWithRouter(<ProfilePage />, { route: '/profile/42' });
  expect(screen.getByText('User #42')).toBeInTheDocument();
});
```

### 8d. Mocking `useNavigate`

```ts
// Vitest
const mockNavigate = vi.fn();
vi.mock('react-router-dom', async () => {
  const actual = await vi.importActual('react-router-dom');
  return {
    ...actual,
    useNavigate: () => mockNavigate,
  };
});

it('redirects to login after logout', async () => {
  const user = userEvent.setup();
  render(<Header />);
  await user.click(screen.getByRole('button', { name: 'Logout' }));
  expect(mockNavigate).toHaveBeenCalledWith('/login');
});
```

### 8e. Mocking `window` / Global APIs

```ts
// Mock localStorage
const localStorageMock = {
  getItem: vi.fn(),
  setItem: vi.fn(),
  clear: vi.fn(),
  removeItem: vi.fn(),
};
Object.defineProperty(window, 'localStorage', { value: localStorageMock });

// Mock window.matchMedia (e.g., dark mode hook)
Object.defineProperty(window, 'matchMedia', {
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
  })),
});

// Mock IntersectionObserver
global.IntersectionObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}));

// Mock ResizeObserver
global.ResizeObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}));

// Mock fetch
global.fetch = vi.fn();

// Mock timers
vi.useFakeTimers();
vi.advanceTimersByTime(1000); // fast-forward 1 second
vi.runAllTimers();
vi.useRealTimers(); // restore after test
```

### 8f. `vi.spyOn` — spy on real implementation

```ts
import * as userApi from './api/users';

it('calls API with correct params', async () => {
  const spy = vi.spyOn(userApi, 'fetchUsers').mockResolvedValue([]);
  render(<UserList />);
  await screen.findByText('No users');
  expect(spy).toHaveBeenCalledWith({ page: 1, limit: 20 });
  spy.mockRestore();
});
```

---

## 9. Testing Hooks

Use `renderHook` from RTL to test custom hooks in isolation:

```tsx
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('initialises with default value', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('increments the count', () => {
    const { result } = renderHook(() => useCounter(0));
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  it('resets to initial value', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => { result.current.increment(); });
    act(() => { result.current.reset(); });
    
    expect(result.current.count).toBe(5);
  });
});
```

**Testing async hooks:**

```tsx
import { renderHook, waitFor } from '@testing-library/react';
import { useFetch } from './useFetch';

it('fetches and returns data', async () => {
  global.fetch = vi.fn().mockResolvedValue({
    json: () => Promise.resolve({ name: 'Alice' }),
  });

  const { result } = renderHook(() => useFetch('/api/user/1'));

  // Initially loading
  expect(result.current.loading).toBe(true);

  // Wait for data
  await waitFor(() => expect(result.current.loading).toBe(false));

  expect(result.current.data).toEqual({ name: 'Alice' });
  expect(result.current.error).toBeNull();
});
```

**Testing hooks that use Context:**

```tsx
const wrapper = ({ children }) => (
  <AuthProvider><ThemeProvider>{children}</ThemeProvider></AuthProvider>
);

const { result } = renderHook(() => useAuth(), { wrapper });
```

---

## 10. Testing Context

### Approach 1 — Provide real Context in test

```tsx
import { render, screen } from '@testing-library/react';
import { ThemeContext } from './ThemeContext';
import { ThemedButton } from './ThemedButton';

it('renders with dark theme from context', () => {
  render(
    <ThemeContext.Provider value={{ theme: 'dark', toggleTheme: vi.fn() }}>
      <ThemedButton>Click me</ThemedButton>
    </ThemeContext.Provider>
  );
  expect(screen.getByRole('button')).toHaveClass('btn-dark');
});
```

### Approach 2 — Custom render with all providers (recommended for large apps)

```tsx
// test-utils.tsx — shared across all tests
import { render, RenderOptions } from '@testing-library/react';
import { AuthProvider } from '../context/AuthContext';
import { ThemeProvider } from '../context/ThemeContext';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function AllProviders({ children }) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } }, // no retries in tests
  });

  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </AuthProvider>
    </QueryClientProvider>
  );
}

// Override RTL render with providers
const customRender = (ui: React.ReactElement, options?: Omit<RenderOptions, 'wrapper'>) =>
  render(ui, { wrapper: AllProviders, ...options });

export * from '@testing-library/react';
export { customRender as render }; // export our custom render

// Usage in test files — import from test-utils, not RTL directly
import { render, screen } from '../test/test-utils';

it('shows user name from auth context', () => {
  render(<Header />);
  expect(screen.getByText('Alice')).toBeInTheDocument();
});
```

---

## 11. Testing Forms

### Controlled form

```tsx
// LoginForm.tsx
function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    if (!email) { setError('Email is required'); return; }
    onSubmit({ email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input id="email" value={email} onChange={e => setEmail(e.target.value)} />
      <label htmlFor="password">Password</label>
      <input id="password" type="password" value={password} onChange={e => setPassword(e.target.value)} />
      {error && <span role="alert">{error}</span>}
      <button type="submit">Login</button>
    </form>
  );
}

// LoginForm.test.tsx
describe('LoginForm', () => {
  it('shows validation error when email is empty', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.click(screen.getByRole('button', { name: 'Login' }));

    expect(screen.getByRole('alert')).toHaveTextContent('Email is required');
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it('calls onSubmit with credentials on valid submit', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'alice@example.com');
    await user.type(screen.getByLabelText('Password'), 'secret123');
    await user.click(screen.getByRole('button', { name: 'Login' }));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'alice@example.com',
      password: 'secret123',
    });
    expect(screen.queryByRole('alert')).not.toBeInTheDocument();
  });
});
```

### React Hook Form (RHF)

```tsx
// Since RHF is uncontrolled — type into inputs and submit normally
it('submits RHF form with correct values', async () => {
  const user = userEvent.setup();
  const handleSubmit = vi.fn();
  render(<RHFLoginForm onSubmit={handleSubmit} />);

  await user.type(screen.getByLabelText('Email'), 'bob@example.com');
  await user.type(screen.getByLabelText('Password'), 'pass1234');
  await user.click(screen.getByRole('button', { name: 'Sign In' }));

  // RHF may be async — wait for submit handler
  await waitFor(() => {
    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'bob@example.com',
      password: 'pass1234',
    }, expect.anything());
  });
});
```

---

## 12. Testing API Calls

### With `msw` (Mock Service Worker) — recommended for integration tests

```bash
npm install -D msw
```

```ts
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' },
    ]);
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 3, ...body }, { status: 201 });
  }),

  http.get('/api/users/:id', ({ params }) => {
    if (params.id === '999') {
      return HttpResponse.json({ message: 'Not found' }, { status: 404 });
    }
    return HttpResponse.json({ id: params.id, name: 'Alice' });
  }),
];
```

```ts
// src/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```ts
// src/test/setup.ts
import { server } from '../mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers()); // reset overrides between tests
afterAll(() => server.close());
```

```tsx
// UserList.test.tsx
import { server } from '../mocks/server';
import { http, HttpResponse } from 'msw';

it('renders user list', async () => {
  render(<UserList />);
  expect(await screen.findByText('Alice')).toBeInTheDocument();
  expect(screen.getByText('Bob')).toBeInTheDocument();
});

it('shows error state on API failure', async () => {
  // Override handler for this specific test
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json({ message: 'Server Error' }, { status: 500 });
    })
  );

  render(<UserList />);
  expect(await screen.findByRole('alert')).toHaveTextContent('Something went wrong');
});
```

### With direct `vi/jest.fn()` mock (unit tests)

```tsx
// Mock the entire module
vi.mock('../api/users', () => ({
  getUsers: vi.fn(),
}));

import { getUsers } from '../api/users';

beforeEach(() => {
  vi.mocked(getUsers).mockResolvedValue([{ id: 1, name: 'Alice' }]);
});

it('fetches and renders users', async () => {
  render(<UserList />);
  await screen.findByText('Alice');
  expect(getUsers).toHaveBeenCalledOnce();
});
```

---

## 13. Snapshot Testing

### When to use snapshots

| ✅ Good for | ❌ Bad for |
|---|---|
| Preventing unexpected UI regressions in static/presentational components | Components with heavy logic — snapshots hide intent |
| Design system atoms (Badge, Tag, Icon) | Large components — diffs become unreadable |
| CSS-in-JS class name stability | Frequently changing UI |

```tsx
// Badge.test.tsx
import { render } from '@testing-library/react';

it('matches snapshot', () => {
  const { container } = render(<Badge variant="success" count={5} />);
  expect(container).toMatchSnapshot();
});

// First run: creates __snapshots__/Badge.test.tsx.snap
// Subsequent runs: fails if output changes
// Update snapshots: vitest --update-snapshots  OR  jest --updateSnapshot
```

### Inline snapshots (preferred — snapshot lives in the test file)

```tsx
it('renders success badge', () => {
  const { container } = render(<Badge variant="success">New</Badge>);
  expect(container).toMatchInlineSnapshot(`
    <div>
      <span class="badge badge-success">
        New
      </span>
    </div>
  `);
});
```

---

## 14. Code Coverage

```bash
# Vitest
vitest run --coverage

# Jest  
jest --coverage
```

```
# Coverage output categories:
Statements   —  % of JS statements executed
Branches     —  % of if/else/ternary branches taken
Functions    —  % of functions called
Lines        —  % of lines executed
```

### Coverage thresholds (enforce in CI)

```ts
// vite.config.ts
test: {
  coverage: {
    provider: 'v8',
    thresholds: {
      statements: 80,
      branches: 75,
      functions: 80,
      lines: 80,
    },
    exclude: [
      'src/main.tsx',
      'src/test/**',
      '**/*.d.ts',
      '**/*.config.*',
      '**/types/**',
    ],
  },
}
```

> **MANG tip:** 80%+ coverage is a good target but coverage is not a guarantee of quality. 100% coverage with bad assertions gives false confidence. **Focus on testing user-visible behaviour, not every line.**

---

## 15. Common Patterns & Best Practices

### Pattern 1 — Arrange / Act / Assert (AAA)

```tsx
it('shows cart count after adding item', async () => {
  // Arrange — set up the test
  const user = userEvent.setup();
  render(<ProductPage productId="1" />);
  await screen.findByText('Add to Cart'); // wait for page to load

  // Act — do the thing
  await user.click(screen.getByRole('button', { name: 'Add to Cart' }));

  // Assert — verify outcome
  expect(screen.getByRole('status')).toHaveTextContent('1 item in cart');
});
```

### Pattern 2 — Test IDs as last resort

```tsx
// Only use data-testid when no semantic query works
// e.g., a loading skeleton with no accessible text
<div data-testid="skeleton-loader" aria-hidden="true" />

// In test:
expect(screen.getByTestId('skeleton-loader')).toBeInTheDocument();
```

### Pattern 3 — Cleaning up timers and subscriptions

```tsx
afterEach(() => {
  vi.clearAllMocks();   // clear mock call history
  vi.clearAllTimers();  // clear fake timers
  cleanup();            // RTL auto-cleans via afterEach by default
});
```

### Pattern 4 — Testing accessibility

```tsx
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

it('has no accessibility violations', async () => {
  const { container } = render(<LoginForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### Pattern 5 — Avoid `act()` warnings

```tsx
// RTL's userEvent and findBy automatically wrap in act()
// Only use act() manually for:
// 1. renderHook state changes
// 2. Directly calling component methods that update state

act(() => {
  result.current.increment();
});
```

### Pattern 6 — Testing error boundaries

```tsx
// Suppress console.error for expected errors
beforeEach(() => {
  vi.spyOn(console, 'error').mockImplementation(() => {});
});
afterEach(() => {
  vi.mocked(console.error).mockRestore();
});

it('renders fallback on child error', () => {
  const ThrowingComponent = () => { throw new Error('Boom!'); };

  render(
    <ErrorBoundary fallback={<div>Something went wrong</div>}>
      <ThrowingComponent />
    </ErrorBoundary>
  );

  expect(screen.getByText('Something went wrong')).toBeInTheDocument();
});
```

### Pattern 7 — Parameterised tests (`it.each`)

```tsx
it.each([
  { input: '',          valid: false, error: 'Required' },
  { input: 'bad',       valid: false, error: 'Invalid email' },
  { input: 'a@b.com',   valid: true,  error: null },
])('validates email "$input"', async ({ input, valid, error }) => {
  const user = userEvent.setup();
  render(<EmailInput />);
  if (input) await user.type(screen.getByRole('textbox'), input);
  await user.tab(); // trigger blur validation

  if (error) {
    expect(screen.getByText(error)).toBeInTheDocument();
  } else {
    expect(screen.queryByRole('alert')).not.toBeInTheDocument();
  }
});
```

---

## 16. MANG Interview Q&A

---

### Q1: What is the difference between unit, integration, and e2e tests?

| | Unit Tests | Integration Tests | E2E Tests |
|---|---|---|---|
| **What** | Single function/component in isolation | Multiple components/modules working together | Full user journey in real browser |
| **Tools** | Jest/Vitest + RTL | Jest/Vitest + RTL + MSW | Cypress, Playwright |
| **Speed** | Very fast (ms) | Fast (seconds) | Slow (minutes) |
| **Confidence** | Low (doesn't test integration) | Medium | High |
| **Maintenance** | Low | Medium | High |

> **Google/Meta testing philosophy (The Testing Trophy / Test Pyramid):** Invest most in integration tests — they give the best confidence-to-cost ratio.

---

### Q2: Why does RTL discourage testing implementation details?

**A:** If you test implementation details (component state, internal method calls, CSS classes tied to implementation), your tests break every time you refactor — even when the user experience doesn't change. This creates **false negatives** (test fails when behaviour is correct) or **false positives** (test passes when UI is broken).

```tsx
// ❌ Implementation detail test — breaks on refactor
expect(wrapper.state('isOpen')).toBe(true);    // Enzyme-style
expect(component.find('.menu-open')).toExist(); // CSS class

// ✅ Behaviour test — survives refactoring internals
expect(screen.getByRole('menu')).toBeVisible();
expect(screen.getByRole('button', { name: 'Menu' })).toHaveAttribute('aria-expanded', 'true');
```

---

### Q3: How do you test a component with `useEffect` that fetches data?

```tsx
// Component
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser)
      .finally(() => setLoading(false));
  }, [userId]);

  if (loading) return <Skeleton />;
  if (!user) return <p>Not found</p>;
  return <h1>{user.name}</h1>;
}

// Test
it('fetches and displays user', async () => {
  // Option 1: Mock fetch directly
  global.fetch = vi.fn().mockResolvedValue({
    json: () => Promise.resolve({ id: 1, name: 'Alice' }),
  });

  render(<UserProfile userId="1" />);
  
  // Loading state
  expect(screen.getByTestId('skeleton')).toBeInTheDocument();
  
  // Loaded state
  expect(await screen.findByRole('heading', { name: 'Alice' })).toBeInTheDocument();
  
  // Verify fetch was called correctly
  expect(fetch).toHaveBeenCalledWith('/api/users/1');
});
```

---

### Q4: How would you test a debounced search input?

```tsx
it('debounces search — only calls API after user stops typing', async () => {
  vi.useFakeTimers();
  const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
  const onSearch = vi.fn();
  
  render(<SearchInput onSearch={onSearch} debounceMs={300} />);
  
  await user.type(screen.getByRole('textbox'), 'react');
  
  // Hasn't fired yet — still within debounce window
  expect(onSearch).not.toHaveBeenCalled();
  
  // Fast-forward 300ms
  act(() => vi.advanceTimersByTime(300));
  
  expect(onSearch).toHaveBeenCalledTimes(1);
  expect(onSearch).toHaveBeenCalledWith('react');
  
  vi.useRealTimers();
});
```

---

### Q5: What is `act()` and why does RTL handle it automatically?

**A:** `act()` is a React test utility that ensures all state updates, effects, and re-renders triggered by a code block are processed and the DOM is updated before you make assertions.

RTL wraps its utilities (`render`, `fireEvent`, `userEvent`, `findBy`, `waitFor`) in `act()` automatically. You only need manual `act()` when:
- Testing with `renderHook` and calling state-updating methods
- Advancing fake timers that trigger React state updates

```tsx
// Manual act() needed here:
const { result } = renderHook(() => useTimer());
act(() => {
  vi.advanceTimersByTime(1000); // triggers setState inside the hook
});
expect(result.current.seconds).toBe(1);
```

---

### Q6: How do you test React portals?

```tsx
// createPortal renders into document.body — RTL queries the whole document by default
// So screen.getByRole() WILL find elements inside portals!

it('renders modal content', async () => {
  const user = userEvent.setup();
  render(<App />);
  
  await user.click(screen.getByRole('button', { name: 'Open Modal' }));
  
  // Even though modal renders in document.body portal, screen finds it
  expect(screen.getByRole('dialog')).toBeInTheDocument();
  expect(screen.getByText('Confirm your action')).toBeInTheDocument();
  
  await user.click(screen.getByRole('button', { name: 'Cancel' }));
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
});
```

---

### Q7: What is the difference between `vi.mock` and `vi.spyOn`?

```ts
// vi.mock — replaces the ENTIRE module with a mock
// Useful when you want to fully control module behaviour
vi.mock('./api');
// Now all exports from ./api are vi.fn()

// vi.spyOn — wraps a SPECIFIC method of an object/module
// Preserves the original implementation unless you mock it
const spy = vi.spyOn(console, 'error');
// Can still call the original, but you can track/override calls
spy.mockImplementation(() => {}); // suppress console.error in test
```

---

### Q8: Scenario — Your RTL test intermittently fails in CI. How do you debug it?

**Answer framework (MANG-level):**

1. **Identify the flakiness type:**
   - Timing issue → use `findBy` instead of `getBy`, or increase `waitFor` timeout
   - Order dependency → tests share state (missing `cleanup`, shared mock not reset)
   - Non-deterministic data → mock unstable data (Date, Math.random, UUIDs)

2. **Run in isolation:** `vitest run --reporter=verbose --reporter=hanging-process`

3. **Add explicit waits:** Replace synchronous assertions after async actions with `waitFor`

4. **Check timer leaks:** Ensure `vi.useRealTimers()` in `afterEach` if using fake timers

5. **Stub unstable APIs:**
   ```ts
   vi.setSystemTime(new Date('2026-01-01')); // deterministic date
   vi.stubGlobal('crypto', { randomUUID: () => 'test-id-123' });
   ```

6. **Isolate with `--sequence.shuffle=false`** to detect order-dependent failures

---

### Q9: How do you ensure tests don't affect each other?

```ts
// 1. RTL auto-calls cleanup() after each test (unmounts rendered components)
// 2. Reset mocks after each test
afterEach(() => {
  vi.clearAllMocks();         // clear call history
  server.resetHandlers();     // reset MSW handler overrides
  localStorage.clear();       // clear any modified storage
});

// 3. Use factory functions for test data — never share mutable objects
const createUser = (overrides = {}) => ({
  id: '1', name: 'Alice', role: 'user', ...overrides
});

// 4. Each test renders fresh — never store render results outside a test
// 5. Use beforeEach to reset module-level state
```

---

### Q10: Scenario — How would you test a shopping cart feature?

```tsx
// Test the full user journey:
it('full cart flow: add, update quantity, remove, checkout', async () => {
  const user = userEvent.setup();
  
  // Mock API
  server.use(
    http.get('/api/products', () => HttpResponse.json([
      { id: 'p1', name: 'T-Shirt', price: 29.99 },
    ])),
    http.post('/api/orders', () => HttpResponse.json({ orderId: 'ord-123' }, { status: 201 })),
  );
  
  render(<App />, { wrapper: AllProviders });

  // Add to cart
  const addButton = await screen.findByRole('button', { name: 'Add to Cart' });
  await user.click(addButton);
  expect(screen.getByRole('status')).toHaveTextContent('1 item');

  // Update quantity
  const qtyInput = screen.getByRole('spinbutton', { name: 'Quantity' });
  await user.clear(qtyInput);
  await user.type(qtyInput, '3');
  expect(screen.getByText('$89.97')).toBeInTheDocument(); // 3 × 29.99

  // Remove item
  await user.click(screen.getByRole('button', { name: 'Remove T-Shirt' }));
  expect(screen.getByText('Your cart is empty')).toBeInTheDocument();

  // Add again and checkout
  await user.click(screen.getByRole('button', { name: 'Add to Cart' }));
  await user.click(screen.getByRole('button', { name: 'Checkout' }));
  
  expect(await screen.findByText('Order #ord-123 confirmed!')).toBeInTheDocument();
});
```

---

*Last updated: February 2026 | Covers Vitest 1.x · Jest 29.x · RTL 14.x · MSW 2.x · React 18*
