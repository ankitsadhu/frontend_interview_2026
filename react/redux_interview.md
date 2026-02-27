# Redux & Redux Toolkit — MANG Level Interview Guide

> **Covers:** Redux Core · Redux Toolkit (RTK) · RTK Query · Async Patterns · Middleware · Testing · Performance · MANG Deep-Dive Q&A + Scenarios

---

## Table of Contents

1. [Core Redux Concepts](#1-core-redux-concepts)
2. [Redux Data Flow — The Big Picture](#2-redux-data-flow)
3. [Redux Toolkit (RTK)](#3-redux-toolkit)
4. [Slices — `createSlice`](#4-slices--createslice)
5. [Async Actions — `createAsyncThunk`](#5-async-actions--createasyncthunk)
6. [RTK Query](#6-rtk-query)
7. [Selectors & Reselect](#7-selectors--reselect)
8. [Middleware](#8-middleware)
9. [Redux DevTools](#9-redux-devtools)
10. [Redux vs Other State Solutions](#10-redux-vs-other-state-solutions)
11. [Testing Redux](#11-testing-redux)
12. [Performance Patterns](#12-performance-patterns)
13. [MANG Cross-Questions & Scenarios](#13-mang-cross-questions--scenarios)

---

## 1. Core Redux Concepts

### Q: What is Redux? What problem does it solve?

**A:** Redux is a **predictable state container** for JavaScript apps. It solves:

- **Prop drilling**: deeply nested components need state from far-away ancestors
- **Shared mutable state**: multiple components reading/writing the same data without coordination
- **Debugging complexity**: in large apps, it's hard to trace which action caused which state change

**Core principle — three laws:**

| Law | Meaning |
|---|---|
| **Single source of truth** | The entire app state lives in one **store** |
| **State is read-only** | State can only change by **dispatching actions** — never mutate directly |
| **Changes via pure functions** | **Reducers** are pure functions: `(state, action) => newState` |

---

### Q: What are the core building blocks of Redux?

```
Action → Reducer → Store → View → Action (cycle)
```

#### **Action**
A plain object describing *what happened*. Must have a `type` field.
```js
{ type: 'cart/addItem', payload: { id: 'p1', name: 'T-Shirt', qty: 1 } }
```

#### **Action Creator**
A function that returns an action object:
```js
const addItem = (item) => ({ type: 'cart/addItem', payload: item });
dispatch(addItem({ id: 'p1', name: 'T-Shirt', qty: 1 }));
```

#### **Reducer**
A pure function that takes current state + action → returns new state:
```js
function cartReducer(state = [], action) {
  switch (action.type) {
    case 'cart/addItem':
      return [...state, action.payload];  // new array — immutable update
    case 'cart/removeItem':
      return state.filter(item => item.id !== action.payload.id);
    default:
      return state;
  }
}
```

#### **Store**
Holds the entire state tree:
```js
import { createStore } from 'redux';
const store = createStore(rootReducer);

store.getState();         // read current state
store.dispatch(action);   // trigger state change
store.subscribe(fn);      // listen for state changes
```

#### **`combineReducers`** — split state by feature:
```js
const rootReducer = combineReducers({
  cart: cartReducer,
  user: userReducer,
  products: productsReducer,
});
// state.cart, state.user, state.products
```

---

### Q: What is immutability and why does Redux require it?

**A:** Redux requires reducers to **never mutate** the existing state. Instead, they must return new objects/arrays.

```js
// ❌ WRONG — mutates existing state (Redux will NOT detect this change)
case 'addItem':
  state.push(action.payload); // mutation!
  return state;               // same reference → React thinks nothing changed

// ✅ CORRECT — returns new array
case 'addItem':
  return [...state, action.payload];

// ✅ CORRECT — nested update
case 'updateQty':
  return state.map(item =>
    item.id === action.payload.id
      ? { ...item, qty: action.payload.qty }
      : item
  );
```

**Why:** React (and Redux) uses **reference equality** (`===`) to detect changes. Mutating an object keeps the same reference — React sees no change and skips re-renders.

---

## 2. Redux Data Flow

```
┌─────────────────────────────────────────────────────┐
│                     React Component                  │
│                                                      │
│  const count = useSelector(state => state.counter)  │
│  const dispatch = useDispatch()                     │
│  <button onClick={() => dispatch(increment())}>+</button>
└────────────────────┬──────────────────┬─────────────┘
                     │ dispatch(action)  │ re-render
                     ▼                  │
              ┌─────────────┐           │
              │   Middleware │           │
              │  (thunk/saga)│           │
              └──────┬──────┘           │
                     │                  │
                     ▼                  │
              ┌─────────────┐      ┌────┴────┐
              │   Reducer   │─────▶│  Store  │
              │ (pure fn)   │      │  state  │
              └─────────────┘      └─────────┘
```

1. User interacts → component calls `dispatch(action)`
2. Middleware intercepts (for async, logging, etc.)
3. Reducer computes new state
4. Store updates, notifies all subscribed components
5. `useSelector` selectors re-run — components that care re-render

---

## 3. Redux Toolkit

### Q: What is Redux Toolkit and why was it created?

**A:** Redux Toolkit (RTK) is the **official, opinionated, batteries-included** toolset for Redux. It was created because plain Redux had too much boilerplate:

| Problem with plain Redux | RTK Solution |
|---|---|
| Too much boilerplate (action types, creators, reducers) | `createSlice` generates them all |
| Accidental mutations are silent bugs | Built-in **Immer** — write "mutating" code that is actually immutable |
| Setting up `thunk`, `DevTools`, `combineReducers` by hand | `configureStore` adds all defaults |
| Complex async patterns | `createAsyncThunk` |
| Manual data fetching/caching | RTK Query |

### Project Setup

```bash
npm install @reduxjs/toolkit react-redux
```

```ts
// store.ts
import { configureStore } from '@reduxjs/toolkit';
import cartSlice from './features/cart/cartSlice';
import userSlice from './features/user/userSlice';

export const store = configureStore({
  reducer: {
    cart: cartSlice,
    user: userSlice,
  },
  // Automatically includes:
  // - redux-thunk middleware
  // - Redux DevTools Extension
  // - Serializable state check middleware (development)
  // - Immutability check middleware (development)
});

// Infer types from store
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

```tsx
// main.tsx
import { Provider } from 'react-redux';
import { store } from './store';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

```ts
// hooks.ts — typed hooks (use these instead of raw useSelector/useDispatch)
import { useDispatch, useSelector } from 'react-redux';
import type { AppDispatch, RootState } from './store';

export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector = <T>(selector: (state: RootState) => T) =>
  useSelector<RootState, T>(selector);
```

---

## 4. Slices — `createSlice`

### Q: What is a Slice and what does `createSlice` generate?

**A:** A **slice** is a self-contained piece of Redux state — its own state, reducers, and actions grouped together (co-location by feature).

`createSlice` accepts: `name`, `initialState`, `reducers` (and optionally `extraReducers`)  
It auto-generates: **action creators** and **action type strings**.

```ts
// features/cart/cartSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CartItem {
  id: string;
  name: string;
  price: number;
  qty: number;
}

interface CartState {
  items: CartItem[];
  couponCode: string | null;
}

const initialState: CartState = {
  items: [],
  couponCode: null,
};

const cartSlice = createSlice({
  name: 'cart',             // prefix for generated action types: 'cart/addItem'
  initialState,
  reducers: {
    // Immer lets us write "mutating" code — RTK converts it to immutable update internally
    addItem(state, action: PayloadAction<CartItem>) {
      const existing = state.items.find(i => i.id === action.payload.id);
      if (existing) {
        existing.qty += 1;  // looks like mutation — but Immer makes it immutable ✓
      } else {
        state.items.push(action.payload); // safe — Immer handles this ✓
      }
    },

    removeItem(state, action: PayloadAction<string>) {
      state.items = state.items.filter(i => i.id !== action.payload);
    },

    updateQty(state, action: PayloadAction<{ id: string; qty: number }>) {
      const item = state.items.find(i => i.id === action.payload.id);
      if (item) item.qty = action.payload.qty;
    },

    clearCart(state) {
      state.items = [];
      state.couponCode = null;
    },

    applyCoupon(state, action: PayloadAction<string>) {
      state.couponCode = action.payload;
    },
  },
});

// Auto-generated action creators:
export const { addItem, removeItem, updateQty, clearCart, applyCoupon } = cartSlice.actions;

// Reducer (default export — goes into configureStore)
export default cartSlice.reducer;
```

### Using the slice in a component

```tsx
// CartPage.tsx
import { useAppSelector, useAppDispatch } from '../../store/hooks';
import { addItem, removeItem, clearCart } from './cartSlice';

function CartPage() {
  const items = useAppSelector(state => state.cart.items);
  const total = useAppSelector(state =>
    state.cart.items.reduce((sum, i) => sum + i.price * i.qty, 0)
  );
  const dispatch = useAppDispatch();

  return (
    <div>
      {items.map(item => (
        <div key={item.id}>
          <span>{item.name} × {item.qty}</span>
          <button onClick={() => dispatch(removeItem(item.id))}>Remove</button>
        </div>
      ))}
      <p>Total: ${total.toFixed(2)}</p>
      <button onClick={() => dispatch(clearCart())}>Clear Cart</button>
    </div>
  );
}
```

### Q: How does Immer work inside RTK?

**A:** RTK uses **Immer** under the hood. When you call a reducer, Immer creates a **draft** (Proxy) of the state. Your code mutates the draft. When done, Immer produces a new immutable object by applying the changes — the original state is untouched.

```js
// What you write (looks like mutation):
addItem(state, action) {
  state.items.push(action.payload);
}

// What actually happens (Immer behind the scenes):
// 1. Creates a draft proxy of state
// 2. Applies: draft.items.push(action.payload)
// 3. Produces new immutable state from draft
// 4. Returns NEW object (original state unchanged)
```

**Immer rules:**
- Either **mutate the draft** OR **return a new value** — never both
- Don't use spread on the draft and mutate the copy — pick one approach

```js
// ❌ Both mutate AND return — Immer throws
clearCart(state) {
  state.items = [];
  return state; // error!
}

// ✅ Mutate the draft
clearCart(state) {
  state.items = [];
}

// ✅ Return new value (replacement — useful when you want to replace root state)
clearCart() {
  return initialState;
}
```

---

## 5. Async Actions — `createAsyncThunk`

### Q: How does async work in Redux? What is a Thunk?

**A:** Redux reducers are **synchronous and pure**. For async operations (API calls, timers), Redux uses **middleware** that intercepts dispatched functions (thunks).

A **thunk** is a function that returns another function:
```js
// Plain Redux thunk (manual)
const fetchUser = (userId) => async (dispatch, getState) => {
  dispatch({ type: 'user/loading' });
  try {
    const user = await api.getUser(userId);
    dispatch({ type: 'user/loaded', payload: user });
  } catch (error) {
    dispatch({ type: 'user/error', payload: error.message });
  }
};
```

### `createAsyncThunk` — RTK's built-in thunk creator

Automatically generates `pending`, `fulfilled`, and `rejected` action types:

```ts
// features/user/userSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

// 1. Define the async thunk
export const fetchUserById = createAsyncThunk(
  'user/fetchById',        // action type prefix
  async (userId: string, thunkAPI) => {
    // thunkAPI: { dispatch, getState, signal, rejectWithValue }
    try {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) throw new Error('User not found');
      return await response.json();   // becomes action.payload on fulfillment
    } catch (error) {
      // rejectWithValue passes a custom value to the rejected case
      return thunkAPI.rejectWithValue({ message: (error as Error).message });
    }
  }
);

// 2. Add extraReducers to handle lifecycle actions
interface UserState {
  data: User | null;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, status: 'idle', error: null } as UserState,
  reducers: {
    resetUser(state) { state.data = null; state.status = 'idle'; }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUserById.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(fetchUserById.fulfilled, (state, action: PayloadAction<User>) => {
        state.status = 'succeeded';
        state.data = action.payload;
      })
      .addCase(fetchUserById.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload?.message ?? 'Something went wrong';
      });
  },
});
```

```tsx
// UserProfile.tsx — using the thunk
function UserProfile({ userId }: { userId: string }) {
  const dispatch = useAppDispatch();
  const { data: user, status, error } = useAppSelector(state => state.user);

  useEffect(() => {
    dispatch(fetchUserById(userId));
  }, [dispatch, userId]);

  if (status === 'loading') return <Skeleton />;
  if (status === 'failed') return <ErrorAlert message={error} />;
  if (!user) return null;

  return <h1>{user.name}</h1>;
}
```

### Thunk with `getState` — reading other slices

```ts
export const addToCartIfLoggedIn = createAsyncThunk(
  'cart/addIfLoggedIn',
  async (item: CartItem, { getState, dispatch, rejectWithValue }) => {
    const state = getState() as RootState;
    
    if (!state.user.data) {
      // Access another slice's state
      return rejectWithValue({ message: 'Must be logged in' });
    }
    
    const response = await api.addToCart(item);
    return response.data;
  }
);
```

### Thunk with `dispatch` — chaining thunks

```ts
export const checkoutThunk = createAsyncThunk(
  'checkout/submit',
  async (_, { getState, dispatch }) => {
    const { cart } = getState() as RootState;
    const order = await api.createOrder(cart.items);
    dispatch(clearCart());              // dispatch another slice's action
    dispatch(fetchOrderHistory());      // dispatch another thunk
    return order;
  }
);
```

### Cancellation with `AbortController`

```ts
export const searchProducts = createAsyncThunk(
  'products/search',
  async (query: string, { signal }) => {
    // signal is automatically tied to the thunk's abort controller
    const response = await fetch(`/api/products?q=${query}`, { signal });
    return response.json();
  }
);

// In component:
useEffect(() => {
  const promise = dispatch(searchProducts(query));
  return () => promise.abort(); // cancel on re-run or unmount
}, [query, dispatch]);
```

---

## 6. RTK Query

### Q: What is RTK Query and how is it different from `createAsyncThunk`?

**A:** RTK Query is a **full data-fetching and caching library** built into RTK. It handles:
- API calls
- Caching (with configurable TTL)
- Loading/error/success states automatically
- Cache invalidation (tags)
- Polling
- Optimistic updates

**When to use which:**

| Use `createAsyncThunk` | Use RTK Query |
|---|---|
| Complex mutations with side effects | Standard CRUD operations |
| Non-HTTP async (WebSocket, timers) | REST/GraphQL APIs |
| You need full control over dispatch flow | You want caching + auto-refetch |
| One-off operations | Repeated fetches that benefit from caching |

```ts
// services/api.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const apiSlice = createApi({
  reducerPath: 'api',           // key in Redux store
  baseQuery: fetchBaseQuery({
    baseUrl: '/api',
    prepareHeaders: (headers, { getState }) => {
      // Attach auth token from state
      const token = (getState() as RootState).user.token;
      if (token) headers.set('Authorization', `Bearer ${token}`);
      return headers;
    },
  }),
  tagTypes: ['User', 'Product', 'Order'], // cache tags for invalidation

  endpoints: (builder) => ({

    // Query (GET — cached)
    getUsers: builder.query<User[], void>({
      query: () => '/users',
      providesTags: ['User'],           // this cache entry provides the 'User' tag
    }),

    getUserById: builder.query<User, string>({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: 'User', id }],
    }),

    // Mutation (POST/PUT/DELETE — not cached)
    createUser: builder.mutation<User, Partial<User>>({
      query: (body) => ({
        url: '/users',
        method: 'POST',
        body,
      }),
      invalidatesTags: ['User'],        // invalidates all 'User' cache entries
    }),

    updateUser: builder.mutation<User, { id: string; data: Partial<User> }>({
      query: ({ id, data }) => ({
        url: `/users/${id}`,
        method: 'PATCH',
        body: data,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'User', id }], // granular invalidation
    }),

    deleteUser: builder.mutation<void, string>({
      query: (id) => ({ url: `/users/${id}`, method: 'DELETE' }),
      invalidatesTags: ['User'],
    }),
  }),
});

// Auto-generated hooks
export const {
  useGetUsersQuery,
  useGetUserByIdQuery,
  useCreateUserMutation,
  useUpdateUserMutation,
  useDeleteUserMutation,
} = apiSlice;
```

```ts
// store.ts — add RTK Query reducer and middleware
import { apiSlice } from './services/api';

export const store = configureStore({
  reducer: {
    [apiSlice.reducerPath]: apiSlice.reducer, // required
    cart: cartSlice,
    user: userSlice,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(apiSlice.middleware), // required for caching/polling
});
```

### Using RTK Query in components

```tsx
// UserList.tsx
function UserList() {
  const {
    data: users,       // cached data
    isLoading,         // true on first load (no cached data)
    isFetching,        // true when re-fetching (stale data exists)
    isError,
    error,
    refetch,           // manually trigger refetch
  } = useGetUsersQuery();

  if (isLoading) return <Skeleton />;
  if (isError) return <ErrorAlert error={error} />;

  return (
    <ul>
      {users?.map(user => <li key={user.id}>{user.name}</li>)}
      <button onClick={refetch}>Refresh</button>
    </ul>
  );
}

// UserForm.tsx — mutation
function UserForm() {
  const [createUser, { isLoading, isSuccess, error }] = useCreateUserMutation();

  const handleSubmit = async (data: Partial<User>) => {
    try {
      await createUser(data).unwrap(); // .unwrap() throws on error (vs promise.reject)
      toast.success('User created!');
    } catch (err) {
      toast.error('Failed to create user');
    }
  };

  return <form onSubmit={...}> ... </form>;
}
```

### RTK Query — Advanced: Optimistic Updates

```ts
updateUser: builder.mutation<User, { id: string; data: Partial<User> }>({
  query: ({ id, data }) => ({ url: `/users/${id}`, method: 'PATCH', body: data }),
  
  async onQueryStarted({ id, data }, { dispatch, queryFulfilled }) {
    // Optimistically update the cache before the request completes
    const patchResult = dispatch(
      apiSlice.util.updateQueryData('getUserById', id, (draft) => {
        Object.assign(draft, data); // Immer draft — safe to mutate
      })
    );
    
    try {
      await queryFulfilled; // wait for actual server response
    } catch {
      patchResult.undo(); // server failed → revert optimistic update
    }
  },
}),
```

### RTK Query — Polling

```tsx
// Refetch every 30 seconds
const { data } = useGetDashboardStatsQuery(undefined, {
  pollingInterval: 30_000,
});

// Skip query conditionally
const { data } = useGetUserByIdQuery(userId, {
  skip: !userId,          // don't fetch if userId is null/undefined
});

// Refetch on window focus
const { data } = useGetUsersQuery(undefined, {
  refetchOnFocus: true,
  refetchOnReconnect: true,
});
```

---

## 7. Selectors & Reselect

### Q: What is a selector? Why use them?

**A:** A selector is a function that extracts data from the Redux state:

```ts
// Basic selector (inline)
const total = useAppSelector(state =>
  state.cart.items.reduce((sum, i) => sum + i.price * i.qty, 0)
);
// Problem: this function is recreated every render → runs every render
```

### Q: What is `createSelector` (Reselect)?

**A:** `createSelector` creates **memoized selectors** — they cache the result and only recompute when their inputs change.

```ts
import { createSelector } from '@reduxjs/toolkit'; // built into RTK

// Input selectors — extract raw data
const selectCartItems = (state: RootState) => state.cart.items;
const selectCoupon = (state: RootState) => state.cart.couponCode;

// Memoized derived selector
export const selectCartTotal = createSelector(
  [selectCartItems],   // input selectors
  (items) => {
    // This only recomputes when items reference changes
    return items.reduce((sum, i) => sum + i.price * i.qty, 0);
  }
);

// Composed selectors — one selector feeding another
export const selectDiscountedTotal = createSelector(
  [selectCartTotal, selectCoupon],
  (total, coupon) => {
    if (coupon === 'SAVE20') return total * 0.8;
    if (coupon === 'SAVE10') return total * 0.9;
    return total;
  }
);

// Parameterized selector (factory pattern)
export const makeSelectItemById = (itemId: string) =>
  createSelector(
    [selectCartItems],
    (items) => items.find(i => i.id === itemId)
  );

// Usage in component
function CartTotal() {
  const total = useAppSelector(selectCartTotal); // memoized — stable reference
  const discounted = useAppSelector(selectDiscountedTotal);
  return <p>Total: ${discounted.toFixed(2)} (was ${total.toFixed(2)})</p>;
}
```

### Q (Cross-question): Why does `useSelector` run on every dispatch?

**A:** `useSelector` subscribes to the store. On every dispatch, React-Redux calls the selector with the new state. If the returned value is **referentially different** from the previous value (`===`), the component re-renders.

- Primitive selectors (`state.cart.items.length`) are cheap — same number = no re-render
- Object/array selectors without `createSelector` re-create new references every call → always re-render
- `createSelector` fixes this by caching — returns **same reference** if inputs didn't change

### Q (Cross-question): What's the problem with `useSelector` returning a new object?

```ts
// ❌ Always creates a new object → component re-renders every dispatch
const { name, role } = useAppSelector(state => ({
  name: state.user.name,
  role: state.user.role,
}));

// ✅ Fix 1: separate selectors
const name = useAppSelector(state => state.user.name);
const role = useAppSelector(state => state.user.role);

// ✅ Fix 2: shallowEqual from react-redux
import { shallowEqual } from 'react-redux';
const { name, role } = useAppSelector(state => ({
  name: state.user.name,
  role: state.user.role,
}), shallowEqual);

// ✅ Fix 3: createSelector
const selectUserInfo = createSelector(
  [(state: RootState) => state.user],
  (user) => ({ name: user.name, role: user.role })
);
```

---

## 8. Middleware

### Q: What is Redux middleware?

**A:** Middleware is a **pipeline** between dispatching an action and the moment it reaches the reducer. Each middleware can inspect, modify, delay, or replace actions.

```
dispatch(action) → middleware1 → middleware2 → ... → reducer
```

### Built-in middleware in RTK

`configureStore` automatically adds:
1. **redux-thunk** — allows dispatching functions (async)
2. **Serializable check** (dev) — warns if you put non-serializable values in state (Promises, classes, functions)
3. **Immutability check** (dev) — warns if you accidentally mutate state outside Immer

### Writing custom middleware

```ts
// Signature: store => next => action => result
const loggerMiddleware = (store) => (next) => (action) => {
  console.group(action.type);
  console.log('Previous state:', store.getState());
  console.log('Action:', action);

  const result = next(action);  // pass action to next middleware/reducer

  console.log('Next state:', store.getState());
  console.groupEnd();
  return result;
};
```

**Real-world middleware examples:**

```ts
// Analytics middleware — track specific actions
const analyticsMiddleware = (store) => (next) => (action) => {
  if (action.type === 'cart/checkout') {
    const { cart } = store.getState();
    analytics.track('checkout', {
      items: cart.items.length,
      total: cart.total,
    });
  }
  return next(action);
};

// Error reporting middleware
const errorMiddleware = (store) => (next) => (action) => {
  try {
    return next(action);
  } catch (err) {
    Sentry.captureException(err, { extra: { action, state: store.getState() } });
    throw err;
  }
};

// Debounce middleware — delay specific actions
const debounceMiddleware = (store) => {
  const timers = {};
  return (next) => (action) => {
    if (action.meta?.debounce) {
      clearTimeout(timers[action.type]);
      timers[action.type] = setTimeout(() => next(action), action.meta.debounce);
      return;
    }
    return next(action);
  };
};

// Add to store
export const store = configureStore({
  reducer: { ... },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware()
      .concat(loggerMiddleware)
      .concat(analyticsMiddleware)
      .concat(apiSlice.middleware),
});
```

### Q (Cross-question): What is the difference between middleware and `extraReducers`?

| Middleware | `extraReducers` |
|---|---|
| Runs **before** the reducer | Runs **inside** the reducer |
| Can dispatch new actions, delay, cancel | Can only compute new state |
| Cross-cutting concerns (logging, analytics) | React to actions from other slices/thunks |
| Has access to `dispatch` and `getState` | Only has access to `state` and `action` |

### Q: Redux Thunk vs Redux Saga — when to use which?

| Thunk | Saga |
|---|---|
| Simple async (fetch, timeout) | Complex async flows (race conditions, cancellation, retries) |
| Functions dispatched as actions | Generator functions (`function*`) |
| Easier to learn | Steeper learning curve |
| Built into RTK | Separate package (`redux-saga`) |
| Sequential: async/await | Concurrent: `takeEvery`, `takeLatest`, `fork`, `race` |

```ts
// Thunk — simple and direct
const fetchData = createAsyncThunk('data/fetch', async () => {
  const res = await fetch('/api/data');
  return res.json();
});

// Saga — for complex flows
function* watchFetchData() {
  yield takeLatest('data/fetch', function* () {
    try {
      const data = yield call(fetch, '/api/data');
      yield put({ type: 'data/succeeded', payload: data });
    } catch (e) {
      yield put({ type: 'data/failed', payload: e.message });
    }
  });
}
```

> **MANG recommendation:** Use thunks (with RTK) for 90% of cases. Use sagas only if you need `takeLatest`/`race`/`fork` patterns — which can often be replaced by `createAsyncThunk` with `AbortController`.

---

## 9. Redux DevTools

### Q: What can you do with Redux DevTools?

| Feature | Description |
|---|---|
| **Action log** | See every dispatched action with type + payload |
| **State tree** | Inspect the full Redux state at any point in time |
| **Time-travel debugging** | Step backward/forward through actions, see state at each step |
| **Action replay** | Re-dispatch past actions |
| **Diff view** | See exactly what state changed between actions |
| **Jump to state** | Click any past action → app reverts to that state |
| **Export/Import** | Serialize full action history for bug reports |
| **Persist on reload** | Actions survive page refresh |

```ts
// configureStore enables DevTools automatically in development
// To disable in production:
export const store = configureStore({
  reducer: { ... },
  devTools: process.env.NODE_ENV !== 'production',
});
```

### Q (Cross-question): Why does time-travel debugging work?

**A:** Because Redux state transitions are **pure functions of (state, action) → newState**. Given the same initial state and the same sequence of actions, you always get the same result. Redux DevTools replays the action sequence from the initial state to reconstruct any past state.

---

## 10. Redux vs Other State Solutions

### Q: When to use Redux vs Context vs Zustand vs React Query?

| | Redux (RTK) | React Context | Zustand | React Query |
|---|---|---|---|---|
| **Best for** | Complex client state, large teams | Simple global state (theme, auth) | Medium client state, small API | Server/async state |
| **Boilerplate** | Medium (RTK reduced it) | Low | Very low | Low |
| **DevTools** | Excellent (time-travel) | None | Redux DevTools compatible | React Query DevTools |
| **Selectors** | `createSelector` (memoized) | None (all consumers re-render) | Built-in selectors | N/A (query keys) |
| **Middleware** | Full pipeline | None | `middleware` option | N/A |
| **Learning curve** | Medium | Low | Low | Low-medium |
| **Performance** | Good (with selectors) | Poor at scale (no selectors) | Excellent (atomic subscriptions) | Excellent (query-level caching) |
| **When to avoid** | Only server state (use RQ) | Frequently-changing data | Need time-travel debugging | Pure client-side state |

### Q (Cross-question): Why would you NOT use Redux?

**A:**
1. **Only server state** → React Query/SWR is far better (caching, refetch, stale-while-revalidate)
2. **Simple app** → `useState` + `useContext` is sufficient
3. **Team prefers atomic state** → Zustand or Jotai have less boilerplate
4. **Bundle size matters** → Redux + RTK is ~11KB min+gzip; Zustand is ~1KB

### Q (Cross-question): Can Redux and React Query coexist?

**A:** **Yes — this is the recommended pattern at scale:**
- React Query for **server state** (API responses, caching, sync)
- Redux for **client UI state** (modals, filters, wizard steps, cart)
- This avoids the anti-pattern of putting API responses in Redux and manually managing cache invalidation

---

## 11. Testing Redux

### Testing reducers (pure functions — easy)

```ts
import cartReducer, { addItem, removeItem, clearCart } from './cartSlice';

describe('cartSlice reducer', () => {
  const initialState = { items: [], couponCode: null };

  it('should handle initial state', () => {
    expect(cartReducer(undefined, { type: 'unknown' })).toEqual(initialState);
  });

  it('should add an item', () => {
    const item = { id: 'p1', name: 'T-Shirt', price: 29.99, qty: 1 };
    const state = cartReducer(initialState, addItem(item));
    expect(state.items).toHaveLength(1);
    expect(state.items[0]).toEqual(item);
  });

  it('should increment qty for existing item', () => {
    const item = { id: 'p1', name: 'T-Shirt', price: 29.99, qty: 1 };
    const withItem = cartReducer(initialState, addItem(item));
    const state = cartReducer(withItem, addItem(item));
    expect(state.items).toHaveLength(1);
    expect(state.items[0].qty).toBe(2);
  });

  it('should remove an item', () => {
    const item = { id: 'p1', name: 'T-Shirt', price: 29.99, qty: 1 };
    const withItem = cartReducer(initialState, addItem(item));
    const state = cartReducer(withItem, removeItem('p1'));
    expect(state.items).toHaveLength(0);
  });

  it('should clear cart', () => {
    const item = { id: 'p1', name: 'T-Shirt', price: 29.99, qty: 1 };
    const withItem = cartReducer(initialState, addItem(item));
    const state = cartReducer(withItem, clearCart());
    expect(state).toEqual(initialState);
  });
});
```

### Testing selectors

```ts
import { selectCartTotal, selectDiscountedTotal } from './cartSelectors';

describe('cart selectors', () => {
  const state = {
    cart: {
      items: [
        { id: 'p1', name: 'Shirt', price: 30, qty: 2 },
        { id: 'p2', name: 'Hat', price: 15, qty: 1 },
      ],
      couponCode: 'SAVE20',
    },
  } as RootState;

  it('calculates total', () => {
    expect(selectCartTotal(state)).toBe(75); // 30*2 + 15*1
  });

  it('applies discount', () => {
    expect(selectDiscountedTotal(state)).toBe(60); // 75 * 0.8
  });
});
```

### Testing async thunks

```ts
import { fetchUserById } from './userSlice';
import { configureStore } from '@reduxjs/toolkit';

describe('fetchUserById thunk', () => {
  it('dispatches fulfilled on success', async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ id: '1', name: 'Alice' }),
    });

    const store = configureStore({ reducer: { user: userReducer } });
    await store.dispatch(fetchUserById('1'));

    const state = store.getState().user;
    expect(state.status).toBe('succeeded');
    expect(state.data).toEqual({ id: '1', name: 'Alice' });
  });

  it('dispatches rejected on failure', async () => {
    global.fetch = vi.fn().mockResolvedValue({ ok: false });

    const store = configureStore({ reducer: { user: userReducer } });
    await store.dispatch(fetchUserById('999'));

    const state = store.getState().user;
    expect(state.status).toBe('failed');
    expect(state.error).toBe('User not found');
  });
});
```

### Testing connected components

```tsx
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './cartSlice';
import { CartPage } from './CartPage';

function renderWithStore(ui, preloadedState = {}) {
  const store = configureStore({
    reducer: { cart: cartReducer },
    preloadedState,
  });
  return render(<Provider store={store}>{ui}</Provider>);
}

it('renders cart items', () => {
  renderWithStore(<CartPage />, {
    cart: {
      items: [{ id: 'p1', name: 'T-Shirt', price: 29.99, qty: 2 }],
      couponCode: null,
    },
  });

  expect(screen.getByText('T-Shirt × 2')).toBeInTheDocument();
  expect(screen.getByText('Total: $59.98')).toBeInTheDocument();
});
```

---

## 12. Performance Patterns

### Q: How do you prevent unnecessary re-renders with Redux?

**1. Use memoized selectors (`createSelector`)**
```ts
// ❌ New reference every run → re-renders every dispatch
useSelector(state => state.cart.items.filter(i => i.qty > 0));

// ✅ Memoized — same reference if items didn't change
const selectActiveItems = createSelector(
  [(state: RootState) => state.cart.items],
  (items) => items.filter(i => i.qty > 0)
);
```

**2. Select smaller slices of state**
```ts
// ❌ Selects entire user object — re-renders if any user field changes
const user = useAppSelector(state => state.user);

// ✅ Select only what you need
const userName = useAppSelector(state => state.user.name);
```

**3. Use `shallowEqual` when returning objects**
```ts
const { name, age } = useAppSelector(
  state => ({ name: state.user.name, age: state.user.age }),
  shallowEqual
);
```

**4. Normalize state shape (entity adapter)**

```ts
import { createEntityAdapter } from '@reduxjs/toolkit';

const productsAdapter = createEntityAdapter<Product>();

const productsSlice = createSlice({
  name: 'products',
  initialState: productsAdapter.getInitialState({ loading: false }),
  // State shape: { ids: ['p1', 'p2'], entities: { p1: {...}, p2: {...} }, loading: false }
  reducers: {
    addProduct: productsAdapter.addOne,
    updateProduct: productsAdapter.updateOne,
    removeProduct: productsAdapter.removeOne,
    setProducts: productsAdapter.setAll,
  },
});

// Auto-generated selectors
const { selectAll, selectById, selectIds } = productsAdapter.getSelectors(
  (state: RootState) => state.products
);
```

**Why normalize?**
- O(1) lookups by ID (vs O(n) array scanning)
- Updating one entity doesn't change references for others
- Prevents data duplication across slices

**5. Batch dispatches (rarely needed in RTK)**
```ts
import { batch } from 'react-redux';
// Causes only ONE re-render instead of two
batch(() => {
  dispatch(setUser(user));
  dispatch(setNotifications(notifications));
});
// Note: React 18 auto-batches, so batch() is mainly for React 17
```

---

## 13. MANG Cross-Questions & Scenarios

---

### Scenario 1: E-commerce Cart with Inventory Sync

> **"Design the Redux state for an e-commerce cart that must sync with server inventory in real-time. Items can go out of stock while a user browses."**

**Answer framework:**

```ts
// State design
interface AppState {
  cart: {
    items: Array<{ productId: string; qty: number; addedAt: number }>;
    status: 'idle' | 'syncing' | 'checked-out';
  };
  products: {
    ids: string[];
    entities: Record<string, Product & { stock: number }>;
  };
}
```

- **RTK Query** polling on products endpoint to refresh stock counts every 30s
- **Cart slice** stores only `productId + qty`, not full product data (avoids duplication)
- **Derived selector** combines cart items with product entities to compute display data:
  ```ts
  const selectCartWithProducts = createSelector(
    [selectCartItems, selectProductEntities],
    (cartItems, products) => cartItems.map(ci => ({
      ...ci,
      product: products[ci.productId],
      isOutOfStock: products[ci.productId]?.stock < ci.qty,
    }))
  );
  ```
- **Middleware** intercepts `cart/addItem` and checks current stock:
  ```ts
  const stockCheckMiddleware = (store) => (next) => (action) => {
    if (action.type === 'cart/addItem') {
      const { products } = store.getState();
      const product = products.entities[action.payload.productId];
      if (product.stock <= 0) {
        // Block the add action, show toast instead
        toast.error('Item is out of stock');
        return;
      }
    }
    return next(action);
  };
  ```
- **Checkout thunk** does final server-side stock validation before placing order

---

### Scenario 2: Multi-Tab State Sync

> **"Users open your app in two browser tabs. They add items to cart in Tab 1. Tab 2 should reflect the updated cart immediately. How?"**

**Answer framework:**

```ts
// BroadcastChannel middleware
const channel = new BroadcastChannel('redux-sync');

const syncMiddleware = (store) => {
  // Listen for actions from other tabs
  channel.onmessage = (event) => {
    store.dispatch({ ...event.data, meta: { fromOtherTab: true } });
  };

  return (next) => (action) => {
    const result = next(action);
    
    // Broadcast this tab's actions to others (avoid infinite loop)
    if (!action.meta?.fromOtherTab && action.type.startsWith('cart/')) {
      channel.postMessage(action);
    }
    return result;
  };
};
```

Alternative approaches:
- **`redux-persist`** with `localStorage` + `storage` event listener
- **Service Worker** as central state coordinator
- **Server-side cart** — all tabs fetch from server (RTK Query with `refetchOnFocus`)

Cross-question: *"What about race conditions between tabs?"*
→ Use **optimistic locking** — include a version number with the cart. Server rejects stale updates.

---

### Scenario 3: Undo/Redo System

> **"Implement undo/redo for a drawing app. The state includes canvas objects, layers, and tool settings. How would you design this in Redux?"**

**Answer framework:**

```ts
// Higher-order reducer pattern (wrapper around existing reducer)
function undoable(reducer) {
  const initialState = {
    past: [],        // stack of previous states
    present: reducer(undefined, { type: '@@INIT' }),
    future: [],      // stack of undone states
  };

  return function (state = initialState, action) {
    switch (action.type) {
      case 'UNDO': {
        if (state.past.length === 0) return state;
        return {
          past: state.past.slice(0, -1),
          present: state.past[state.past.length - 1],
          future: [state.present, ...state.future],
        };
      }
      case 'REDO': {
        if (state.future.length === 0) return state;
        return {
          past: [...state.past, state.present],
          present: state.future[0],
          future: state.future.slice(1),
        };
      }
      default: {
        const newPresent = reducer(state.present, action);
        if (newPresent === state.present) return state; // no change
        return {
          past: [...state.past, state.present].slice(-50), // cap history at 50
          present: newPresent,
          future: [], // clear redo stack on new action
        };
      }
    }
  };
}

// Usage
const canvasReducer = createSlice({ name: 'canvas', ... }).reducer;
const undoableCanvas = undoable(canvasReducer);
```

Cross-question: *"What about performance with large state objects in undo history?"*
→ Use **structural sharing** (Immer does this) — unchanged parts aren't deep-cloned.
→ Store **diffs** instead of full snapshots for very large states.
→ Use **`redux-undo`** library for production.

---

### Scenario 4: Feature Flags via Redux

> **"Your team uses feature flags that come from an API and control which UI features are visible. How would you integrate this with Redux?"**

```ts
// Feature flag slice
const featureFlagsSlice = createSlice({
  name: 'featureFlags',
  initialState: {} as Record<string, boolean>,
  reducers: {},
  extraReducers: (builder) => {
    builder.addCase(fetchFeatureFlags.fulfilled, (state, action) => {
      return action.payload; // { darkMode: true, newCheckout: false, betaSearch: true }
    });
  },
});

// Typed selector
export const selectFeatureFlag = (flag: string) =>
  createSelector(
    [(state: RootState) => state.featureFlags],
    (flags) => flags[flag] ?? false
  );

// Custom hook
function useFeatureFlag(flag: string): boolean {
  return useAppSelector(selectFeatureFlag(flag));
}

// Usage in component
function Checkout() {
  const newCheckoutEnabled = useFeatureFlag('newCheckout');
  return newCheckoutEnabled ? <NewCheckoutFlow /> : <LegacyCheckout />;
}
```

Cross-question: *"How do you handle flag changes after the app has loaded?"*
→ Poll the flags endpoint with RTK Query (`pollingInterval: 60_000`)  
→ Or use **Server-Sent Events** to push flag changes in real-time  
→ Components automatically re-render because `useSelector` reacts to store changes

---

### Scenario 5: Large-Scale Form Wizard with Redux

> **"You have a 7-step insurance application form. Users can save progress, navigate back, and resume later. How do you architect the Redux state?"**

```ts
// State design
interface WizardState {
  currentStep: number;
  completedSteps: number[];
  data: {
    personal: { name: string; dob: string; email: string } | null;
    address: { street: string; city: string; zip: string } | null;
    coverage: { plan: string; amount: number } | null;
    // ... steps 4-7
  };
  savedAt: string | null;
  isDirty: boolean;       // unsaved changes?
}

const wizardSlice = createSlice({
  name: 'wizard',
  initialState: { currentStep: 0, completedSteps: [], data: {}, savedAt: null, isDirty: false },
  reducers: {
    goToStep(state, action: PayloadAction<number>) {
      if (state.completedSteps.includes(action.payload) || action.payload === state.currentStep + 1) {
        state.currentStep = action.payload;
      }
    },
    saveStepData(state, action: PayloadAction<{ step: string; data: any }>) {
      state.data[action.step] = action.payload.data;
      state.completedSteps = [...new Set([...state.completedSteps, state.currentStep])];
      state.isDirty = true;
    },
    markSaved(state) {
      state.savedAt = new Date().toISOString();
      state.isDirty = false;
    },
  },
});
```

- **Auto-save middleware**: debounce saves to server on every `saveStepData` action
- **`redux-persist`** to survive page refreshes (localStorage)
- **Resume thunk**: loads saved progress from server on mount
- **Validation**: per-step Zod schemas validated before allowing `goToStep(next)`

Cross-question: *"What if the form has interdependent fields across steps?"*
→ Use `createSelector` to derive validation for Step 5 based on Step 2's data  
→ The wizard reducer has access to full `state.data` — cross-step logic belongs in thunks/selectors, not in individual step components

---

### Scenario 6: Debugging a Redux Bug

> **"A customer reports that after adding 3 specific items and applying a coupon, the total shows $0. How do you debug this?"**

**Answer framework (step-by-step):**

1. **Ask customer for Redux DevTools export** — they can export the full action log JSON
2. **Import into DevTools** — replay every action. Jump to the state after coupon was applied.
3. **Check the diff view** — what exact state change caused total to become 0?
4. **Inspect the selector** — is `selectDiscountedTotal` computing correctly?
5. **Check for**:
   - Coupon reducer setting `items` to `[]` by mistake (wrong action handled)
   - Selector treating the coupon code as 100% discount
   - Floating-point rounding error causing negative → clamp to 0
   - `clearCart` being dispatched by a side effect after coupon apply

```ts
// Likely bug: coupon reducer that matches wrong action
case 'cart/applyCoupon':
  state.items = [];  // ← developer meant to clear coupon, not items!
  state.couponCode = action.payload;
```

6. **Write a regression test**:
```ts
it('applying coupon does not clear items', () => {
  const withItems = cartReducer(initialState, addItem(item));
  const state = cartReducer(withItems, applyCoupon('SAVE20'));
  expect(state.items).toHaveLength(1);    // regression guard
  expect(state.couponCode).toBe('SAVE20');
});
```

---

### Scenario 7: Normalizing Deeply Nested API Data

> **"Your API returns deeply nested data: orders contain products, products contain categories, categories contain parent categories. How do you structure Redux state?"**

```json
// API response (nested)
{
  "orders": [{
    "id": "o1",
    "products": [{
      "id": "p1",
      "category": { "id": "c1", "name": "Clothing", "parent": { "id": "c0", "name": "All" } }
    }]
  }]
}
```

```ts
// ✅ Normalized state (flat)
{
  orders: { ids: ['o1'], entities: { o1: { id: 'o1', productIds: ['p1'] } } },
  products: { ids: ['p1'], entities: { p1: { id: 'p1', categoryId: 'c1' } } },
  categories: { ids: ['c0', 'c1'], entities: { c0: {...}, c1: { id: 'c1', parentId: 'c0' } } }
}
```

Use `createEntityAdapter` per entity type. Use a thunk to normalize API response:

```ts
import { normalize, schema } from 'normalizr';

const categorySchema = new schema.Entity('categories');
categorySchema.define({ parent: categorySchema }); // recursive

const productSchema = new schema.Entity('products', { category: categorySchema });
const orderSchema = new schema.Entity('orders', { products: [productSchema] });

export const fetchOrders = createAsyncThunk('orders/fetch', async () => {
  const response = await api.getOrders();
  const normalized = normalize(response, [orderSchema]);
  return normalized.entities; // { orders: {...}, products: {...}, categories: {...} }
});
```

Cross-question: *"Why normalize instead of keeping nested structure?"*
1. **No data duplication** — same category referenced by 100 products is stored once
2. **O(1) updates** — update one product without traversing all orders
3. **Stable references** — updating product P1 doesn't change the reference of product P2

---

### Scenario 8: Migrating from Context to Redux

> **"Your app grew from 5 to 50 components consuming the same Context. Performance tanked. Walk me through migrating to Redux."**

**Answer framework:**

1. **Audit current Context**: identify what data is in context and how often it changes
2. **Split data by frequency**: 
   - Rarely changes (theme, locale) → **keep in Context**
   - Frequently changes (cart, notifications) → **move to Redux**
3. **Create slices** for each domain being migrated
4. **Replace `useContext` with `useSelector`** — one component at a time
5. **Create typed hooks** (`useAppSelector`, `useAppDispatch`) immediately
6. **Add memoized selectors** for computed values
7. **Test each migrated component** independently
8. **Remove old Context providers** once all consumers are migrated

Cross-question: *"How do you prove the migration improved performance?"*
→ **Before**: React DevTools Profiler shows all 50 consumers re-rendering on every context change  
→ **After**: Profiler shows only consumers of the changed slice re-rendering  
→ Measure with `<Profiler>` API and compare `actualDuration` metrics

---

### Scenario 9: Redux with WebSocket Real-time Data

> **"Your dashboard receives real-time stock prices via WebSocket. Hundreds of prices update per second. How do you handle this in Redux?"**

```ts
// Middleware approach — throttle WebSocket dispatches
const wsMiddleware = (store) => (next) => (action) => {
  if (action.type === 'ws/connect') {
    const ws = new WebSocket('wss://prices.example.com');
    let buffer = {};

    ws.onmessage = (event) => {
      const { symbol, price } = JSON.parse(event.data);
      buffer[symbol] = price; // buffer updates
    };

    // Flush buffer to Redux at 10fps (not per-message)
    setInterval(() => {
      if (Object.keys(buffer).length > 0) {
        store.dispatch({ type: 'prices/batchUpdate', payload: buffer });
        buffer = {};
      }
    }, 100);
  }
  return next(action);
};

// EntityAdapter for prices
const pricesSlice = createSlice({
  name: 'prices',
  initialState: pricesAdapter.getInitialState(),
  reducers: {
    batchUpdate(state, action: PayloadAction<Record<string, number>>) {
      Object.entries(action.payload).forEach(([symbol, price]) => {
        pricesAdapter.upsertOne(state, { id: symbol, symbol, price, updatedAt: Date.now() });
      });
    },
  },
});
```

Cross-question: *"Won't this cause all price components to re-render?"*
→ No — each component uses a **per-symbol selector**: `useAppSelector(state => selectPriceById(state, 'AAPL'))`  
→ Only the component for the changed symbol re-renders  
→ `createEntityAdapter` provides `selectById` which returns stable references for unchanged entities

---

### Scenario 10: Action Design — Fat Actions vs Thin Actions

> **"Should the action payload contain computed/derived data, or should the reducer compute it?"**

| Fat Action (logic in action creator) | Thin Action (logic in reducer) |
|---|---|
| `{ type: 'order/placed', payload: { items, total: 99.99, tax: 7.99 } }` | `{ type: 'order/placed', payload: { items } }` |
| Easier to test action creator logic | All logic in reducer — pure, testable |
| Multiple reducers can react to pre-computed values | Single responsibility — reducer computes what it needs |
| **Risky**: if logic changes, stale events in replay won't recompute | **Safe**: replaying actions always recomputes from current logic |

**MANG best practice:** Prefer **thin actions** — put minimal event data in the payload, let the reducer compute derived values. This makes **time-travel debugging** and **event sourcing** correct even if business logic changes.

```ts
// ✅ Thin action — reducer computes total
case 'cart/checkout':
  const total = state.items.reduce((s, i) => s + i.price * i.qty, 0);
  const tax = total * 0.08;
  return { ...state, status: 'ordered', total, tax };

// ❌ Fat action — pre-computed values may be wrong if replayed with different logic
dispatch({ type: 'cart/checkout', payload: { total: 99.99, tax: 7.99 } });
```

---

*Last updated: February 2026 | Covers Redux Toolkit 2.x · React-Redux 9.x · RTK Query*
