# JavaScript Async — MANG Level Interview Q&A

> Event Loop · Promises · async/await · Generators · Web APIs · Error Handling · Concurrency Patterns · Scenarios

---

## Table of Contents

1. [Event Loop Deep Dive](#1-event-loop-deep-dive)
2. [Callbacks](#2-callbacks)
3. [Promises](#3-promises)
4. [async / await](#4-async--await)
5. [Error Handling in Async Code](#5-error-handling)
6. [Promise Combinators](#6-promise-combinators)
7. [Concurrency Patterns](#7-concurrency-patterns)
8. [Generators & Iterators](#8-generators--iterators)
9. [Web APIs & Timers](#9-web-apis--timers)
10. [AbortController & Cancellation](#10-abortcontroller--cancellation)
11. [Real-world Patterns](#11-real-world-patterns)
12. [MANG Scenarios & Cross-Questions](#12-mang-scenarios)

---

## 1. Event Loop Deep Dive

### Q: Explain the JavaScript event loop.

**A:** JavaScript is **single-threaded** — it runs one piece of code at a time. The event loop is the mechanism that allows asynchronous operations by coordinating the **call stack**, **Web APIs**, **microtask queue**, and **macrotask queue**.

```
┌──────────────────────────────────────────────────┐
│                    Call Stack                      │
│  (executes one frame at a time)                   │
└───────────────┬──────────────────────────────────┘
                │  empty?
                ▼
┌──────────────────────────────────────────────────┐
│              Microtask Queue                      │
│  Promise.then, queueMicrotask, MutationObserver   │
│  (ALL drained before any macrotask)               │
└───────────────┬──────────────────────────────────┘
                │  empty?
                ▼
┌──────────────────────────────────────────────────┐
│              Macrotask Queue                      │
│  setTimeout, setInterval, I/O, UI rendering       │
│  (ONE task per event loop tick)                   │
└──────────────────────────────────────────────────┘
                │
                ▼
         🔄 Repeat (event loop)
```

**Key rules:**
1. **Call stack must be empty** before the event loop checks queues
2. **All microtasks** are drained before ANY macrotask runs
3. Only **one macrotask** is processed per event loop tick
4. After each macrotask, all microtasks are drained again
5. **Rendering** (paint/layout) happens between macrotasks, after microtasks

### Cross-question: *"Can microtasks starve the macrotask queue?"*

**Yes.** If a microtask schedules another microtask endlessly, macrotasks (including rendering) never get a chance to run — the page freezes:

```js
// ❌ Infinite microtask loop — freezes the browser
function bad() {
  Promise.resolve().then(bad); // keeps adding microtasks
}
bad(); // setTimeout, UI updates, clicks — NOTHING runs
```

---

### Q: What's the difference between microtasks and macrotasks?

| Microtasks | Macrotasks |
|---|---|
| `Promise.then/catch/finally` | `setTimeout` / `setInterval` |
| `queueMicrotask()` | `setImmediate()` (Node.js) |
| `MutationObserver` | `requestAnimationFrame` |
| `await` (continuation after) | I/O callbacks |
| `process.nextTick()` (Node.js) | UI rendering events |

**Priority: Microtasks > Macrotasks > Rendering**

### Cross-question: *"Where does `requestAnimationFrame` fit?"*

`rAF` runs **after microtasks** but **before paint** — it's technically not a macrotask but its own phase. It runs once per frame (~16ms at 60fps), right before the browser paints.

```
Macrotask → Microtasks (all) → rAF callbacks → Layout/Paint → next Macrotask
```

---

### Q: What is `queueMicrotask` and when would you use it?

```js
queueMicrotask(() => console.log('microtask'));
console.log('sync');

// Output: sync, microtask
```

**Use cases:**
1. Ensuring consistent async timing (always async, but ASAP)
2. Batching DOM reads/writes within the same frame
3. Running cleanup code before the next macrotask

```js
// Batching state updates
let pending = false;
const changes = [];

function scheduleUpdate(change) {
  changes.push(change);
  if (!pending) {
    pending = true;
    queueMicrotask(() => {
      flushChanges(changes);  // process all changes at once
      changes.length = 0;
      pending = false;
    });
  }
}
```

---

## 2. Callbacks

### Q: What is callback hell and how do you solve it?

```js
// ❌ Callback hell — "pyramid of doom"
getUser(userId, (err, user) => {
  if (err) return handleError(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handleError(err);
    getOrderDetails(orders[0].id, (err, details) => {
      if (err) return handleError(err);
      getShippingInfo(details.trackingId, (err, shipping) => {
        if (err) return handleError(err);
        display(shipping);
      });
    });
  });
});
```

**Problems:**
1. Deeply nested, hard to read
2. Error handling repeated at every level
3. Can't return values, can't use try/catch
4. Inversion of control — you trust the caller to call your callback correctly

**Solutions (evolution):**

```js
// ✅ Solution 1: Named functions (flatten)
getUser(userId, handleUser);
function handleUser(err, user) {
  if (err) return handleError(err);
  getOrders(user.id, handleOrders);
}
function handleOrders(err, orders) { /* ... */ }

// ✅ Solution 2: Promises
getUser(userId)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => getShippingInfo(details.trackingId))
  .then(display)
  .catch(handleError); // ONE error handler for all

// ✅ Solution 3: async/await
try {
  const user = await getUser(userId);
  const orders = await getOrders(user.id);
  const details = await getOrderDetails(orders[0].id);
  const shipping = await getShippingInfo(details.trackingId);
  display(shipping);
} catch (err) {
  handleError(err);
}
```

### Cross-question: *"What is 'inversion of control' in callbacks?"*

When you pass a callback to a third-party function, you **trust** it to:
- Call your callback (not forget it)
- Call it only **once** (not multiple times)
- Call it **asynchronously** (not synchronously sometimes)
- Pass the right arguments

Promises fix this: you control `.then()` — the Promise is a trustworthy intermediary.

---

## 3. Promises

### Q: What are the 3 states of a Promise?

```
           ┌─── fulfilled (resolved with a value)
pending ───┤
           └─── rejected (rejected with a reason)
```

- **Pending** → initial state
- **Fulfilled** → operation succeeded, has a `value`
- **Rejected** → operation failed, has a `reason`
- Once settled (fulfilled or rejected), a Promise **never changes state again**

---

### Q: Explain the Promise chain and how values propagate.

```js
fetch('/api/user')
  .then(res => res.json())        // returns a new Promise with parsed JSON
  .then(user => user.name)        // returns a new Promise with the name string
  .then(name => console.log(name)) // receives the name
  .catch(err => console.error(err)); // catches ANY error from above
```

**Rules of chaining:**
1. `.then()` **always returns a new Promise**
2. If the callback returns a **value** → next `.then` receives it
3. If the callback returns a **Promise** → chain waits for it to settle
4. If the callback **throws** → chain jumps to nearest `.catch`
5. `.catch` returns a fulfilled Promise (unless it throws) → **chain recovers**

```js
Promise.resolve(1)
  .then(x => x + 1)           // 2
  .then(x => { throw 'err'; }) // rejects
  .then(x => x + 1)           // SKIPPED
  .catch(e => 42)              // catches 'err', returns 42 (recovers!)
  .then(x => console.log(x)); // 42
```

---

### Q: What is `Promise.resolve` and `Promise.reject`?

```js
// Promise.resolve — wraps a value in a fulfilled Promise
Promise.resolve(42);          // Promise { 42 }
Promise.resolve(promise);     // returns same Promise if already a Promise (idempotent)

// Promise.reject — wraps a reason in a rejected Promise
Promise.reject(new Error('fail')); // Promise { <rejected> Error: fail }

// Common use: normalize sync/async values
function getData(useCache) {
  if (useCache) return Promise.resolve(cache); // sync value → Promise
  return fetch('/api/data').then(r => r.json()); // async value → Promise
}
// Caller always gets a Promise, regardless of the source
```

---

### Q: What is `finally` and when is it called?

```js
fetchData()
  .then(data => process(data))
  .catch(err => logError(err))
  .finally(() => {
    hideLoader(); // runs whether resolved OR rejected
  });
```

**Key behaviors:**
- `finally` callback receives **no arguments** (doesn't know if resolved/rejected)
- `finally` **passes through** the value/reason — doesn't alter the chain
- If `finally` **throws**, it replaces the chain's result with that error

```js
Promise.resolve(42)
  .finally(() => console.log('cleanup'))
  .then(x => console.log(x));
// Output: cleanup, 42  — value passes through
```

---

### Q: What is the difference between `.then(onFulfilled, onRejected)` and `.then().catch()`?

```js
// Version 1: Both handlers in .then()
promise.then(
  data => processAndMightThrow(data),
  err => handleOriginalError(err)
);
// If processAndMightThrow THROWS, it's NOT caught!
// The onRejected only catches errors from the ORIGINAL promise

// Version 2: Separate .catch()
promise
  .then(data => processAndMightThrow(data))
  .catch(err => handleAnyError(err));
// .catch catches errors from BOTH the original promise AND the .then callback
```

**Rule:** Always prefer `.then().catch()` — it catches errors from the entire chain above it.

---

## 4. async / await

### Q: What does `async` / `await` actually do under the hood?

```js
// This async function...
async function fetchUser() {
  const res = await fetch('/api/user');
  const user = await res.json();
  return user;
}

// ...is equivalent to this Promise chain:
function fetchUser() {
  return fetch('/api/user')
    .then(res => res.json())
    .then(user => user);
}
```

**Key facts:**
- `async` makes a function **always return a Promise**
- `await` **pauses** execution until the Promise settles
- Code after `await` runs as a **microtask** (like `.then`)
- `await` can be used with any **thenable** (not just Promises)

---

### Q: What happens if you forget `await`?

```js
async function saveUser(data) {
  // ❌ Missing await — fire and forget!
  fetch('/api/save', { method: 'POST', body: JSON.stringify(data) });
  console.log('Saved!'); // logs BEFORE fetch completes
  // Errors are silently swallowed — no catch
}

// ✅ With await — waits for completion
async function saveUser(data) {
  await fetch('/api/save', { method: 'POST', body: JSON.stringify(data) });
  console.log('Saved!'); // logs AFTER fetch completes
}
```

### Cross-question: *"What is an unhandled Promise rejection?"*

If a Promise rejects and there's no `.catch()` or `try/catch`, it becomes an **unhandled rejection**. In browsers, this fires the `unhandledrejection` event. In Node.js (v15+), it **crashes the process**.

```js
// ❌ Unhandled rejection
Promise.reject(new Error('oops')); // no .catch!

// ✅ Always handle rejections
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled:', event.reason);
  event.preventDefault(); // prevent default logging
});
```

---

### Q: `await` in loops — sequential vs parallel

```js
// ❌ Sequential — each request waits for the previous (SLOW)
async function fetchAllSequential(urls) {
  const results = [];
  for (const url of urls) {
    const res = await fetch(url);    // waits here each iteration
    results.push(await res.json());
  }
  return results; // Total time = sum of all request times
}

// ✅ Parallel — all requests fire simultaneously (FAST)
async function fetchAllParallel(urls) {
  const promises = urls.map(url => fetch(url).then(r => r.json()));
  return Promise.all(promises); // Total time = longest request
}

// ✅ Parallel with error isolation
async function fetchAllSafe(urls) {
  const promises = urls.map(url =>
    fetch(url).then(r => r.json()).catch(err => ({ error: err.message }))
  );
  return Promise.all(promises);
}
```

**When sequential IS correct:**
- Requests depend on each other (need result of previous)
- Rate limiting (don't overwhelm the server)
- Maintaining order with side effects

---

### Q: Can you use `await` at the top level?

```js
// ES Modules (type="module") — YES, top-level await works
const data = await fetch('/api/data').then(r => r.json());
console.log(data);

// CommonJS / classic scripts — NO, only inside async functions
// ❌ SyntaxError: await is only valid in async functions
```

### Cross-question: *"What problem does top-level await solve?"*

Initialization patterns that need async setup:

```js
// Without top-level await — messy IIFE
let connection;
(async () => {
  connection = await connectToDatabase();
})();
// Problem: connection might be undefined when other code runs!

// With top-level await — clean
const connection = await connectToDatabase();
// Module execution pauses until connection is ready
// Any module that imports this one also waits
```

**Caveat:** Top-level await **blocks** the importing module — can slow down app startup if overused.

---

## 5. Error Handling

### Q: How do you handle errors in async code?

**Pattern 1: try/catch with async/await**

```js
async function fetchUser() {
  try {
    const res = await fetch('/api/user');
    if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
    return await res.json();
  } catch (err) {
    if (err instanceof TypeError) {
      // Network error (offline, DNS failure)
      console.error('Network error:', err.message);
    } else {
      // HTTP error or JSON parse error
      console.error('Request failed:', err.message);
    }
    return null; // or re-throw
  }
}
```

**Pattern 2: Go-style tuple returns (no try/catch)**

```js
async function to(promise) {
  try {
    const data = await promise;
    return [null, data];
  } catch (err) {
    return [err, null];
  }
}

// Usage — clean, no try/catch blocks
const [err, user] = await to(fetchUser());
if (err) return handleError(err);

const [err2, orders] = await to(fetchOrders(user.id));
if (err2) return handleError(err2);
```

**Pattern 3: Error boundaries for async in React**

```js
// Async errors in event handlers don't trigger React error boundaries
// You must handle them explicitly
async function handleClick() {
  try {
    await submitForm(data);
  } catch (err) {
    setError(err.message); // update UI state
  }
}
```

### Cross-question: *"Does `try/catch` catch errors in callbacks inside the try block?"*

**No** — if the callback runs asynchronously:

```js
try {
  setTimeout(() => {
    throw new Error('boom');  // NOT caught by this try/catch!
  }, 100);
} catch (err) {
  console.log('Caught:', err); // never runs
}
// The throw happens in a DIFFERENT call stack (macrotask)

// ✅ Fix: use async/await or handle errors inside the callback
setTimeout(() => {
  try { throw new Error('boom'); }
  catch (err) { console.log('Caught:', err); } // ✅ caught
}, 100);
```

---

## 6. Promise Combinators

### Q: Compare all 4 Promise combinators

| Combinator | Resolves when | Rejects when | Use case |
|---|---|---|---|
| `Promise.all` | **All** fulfill | **First** rejection | Load multiple resources in parallel |
| `Promise.allSettled` | **All** settle (never rejects) | Never | Fire-and-forget batch, report per-item status |
| `Promise.race` | **First** settles (fulfill or reject) | **First** settles | Timeout pattern, fastest CDN |
| `Promise.any` | **First** fulfills | **All** reject (`AggregateError`) | Redundant sources, first success wins |

```js
// Promise.all — all or nothing
const [users, posts] = await Promise.all([fetchUsers(), fetchPosts()]);

// Promise.allSettled — collect all results
const results = await Promise.allSettled([api1(), api2(), api3()]);
const successes = results.filter(r => r.status === 'fulfilled');
const failures = results.filter(r => r.status === 'rejected');

// Promise.race — timeout pattern
const result = await Promise.race([
  fetch('/api/data'),
  new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000)),
]);

// Promise.any — first success (ES2021)
const fastest = await Promise.any([
  fetch('https://cdn1.example.com/data'),
  fetch('https://cdn2.example.com/data'),
  fetch('https://cdn3.example.com/data'),
]);
```

### Cross-question: *"What is `AggregateError`?"*

`Promise.any` rejects with `AggregateError` when ALL promises reject. It contains all individual errors:

```js
try {
  await Promise.any([
    Promise.reject('Error 1'),
    Promise.reject('Error 2'),
  ]);
} catch (err) {
  console.log(err instanceof AggregateError); // true
  console.log(err.errors); // ['Error 1', 'Error 2']
}
```

---

## 7. Concurrency Patterns

### Q: How do you limit concurrent async operations?

**Problem:** 1000 URLs to fetch, but you can only run 5 at a time (rate limiting).

```js
async function asyncPool(limit, items, fn) {
  const results = [];
  const executing = new Set();

  for (const [index, item] of items.entries()) {
    const promise = fn(item, index).then(result => {
      executing.delete(promise);
      return result;
    });

    results.push(promise);
    executing.add(promise);

    if (executing.size >= limit) {
      await Promise.race(executing); // wait for one to finish
    }
  }

  return Promise.all(results);
}

// Usage — fetch 100 URLs, max 5 concurrent
const urls = Array.from({ length: 100 }, (_, i) => `/api/item/${i}`);
const data = await asyncPool(5, urls, async (url) => {
  const res = await fetch(url);
  return res.json();
});
```

---

### Q: Implement `retry` with exponential backoff

```js
async function retry(fn, { retries = 3, baseDelay = 1000, factor = 2 } = {}) {
  let lastError;

  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;

      if (attempt === retries) break;

      const delay = baseDelay * Math.pow(factor, attempt);
      const jitter = delay * (0.5 + Math.random() * 0.5); // add randomness
      console.log(`Attempt ${attempt + 1} failed, retrying in ${Math.round(jitter)}ms...`);
      await sleep(jitter);
    }
  }

  throw lastError;
}

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

// Usage
const data = await retry(() => fetch('/api/flaky-endpoint').then(r => {
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
}), { retries: 3, baseDelay: 1000 });
```

### Cross-question: *"Why add jitter to backoff delay?"*

Without jitter, if 1000 clients fail simultaneously, they all retry at exactly the same time (thundering herd). Jitter randomizes retry timing → spreads the load.

---

### Q: Implement a `Promise` queue (serial execution)

```js
class AsyncQueue {
  #queue = [];
  #running = false;

  enqueue(asyncFn) {
    return new Promise((resolve, reject) => {
      this.#queue.push({ asyncFn, resolve, reject });
      this.#process();
    });
  }

  async #process() {
    if (this.#running) return;
    this.#running = true;

    while (this.#queue.length > 0) {
      const { asyncFn, resolve, reject } = this.#queue.shift();
      try {
        const result = await asyncFn();
        resolve(result);
      } catch (err) {
        reject(err);
      }
    }

    this.#running = false;
  }
}

// Usage — database writes that must be sequential
const queue = new AsyncQueue();
queue.enqueue(() => db.write('record 1'));
queue.enqueue(() => db.write('record 2')); // waits for record 1
queue.enqueue(() => db.write('record 3')); // waits for record 2
```

---

### Q: Implement `promisify` (convert callback-based function to Promise)

```js
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}

// Usage
const fs = require('fs');
const readFile = promisify(fs.readFile);
const content = await readFile('file.txt', 'utf8');

// Multi-argument callbacks
function promisifyMulti(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, ...results) => {
        if (err) reject(err);
        else resolve(results.length === 1 ? results[0] : results);
      });
    });
  };
}
```

---

## 8. Generators & Iterators

### Q: How do generators relate to async/await?

Generators were the **precursor** to async/await. Before `async/await` existed, libraries like `co` used generators to write async code that looked synchronous:

```js
// Generator-based async (old pattern)
function* fetchUserGen() {
  const res = yield fetch('/api/user'); // yield a Promise
  const user = yield res.json();       // yield another Promise
  return user;
}

// Runner that drives the generator
function run(genFn) {
  const gen = genFn();

  function step(value) {
    const result = gen.next(value);
    if (result.done) return Promise.resolve(result.value);
    return Promise.resolve(result.value).then(
      val => step(val),
      err => gen.throw(err)
    );
  }

  return step();
}

// Usage
run(fetchUserGen).then(user => console.log(user));

// Modern equivalent — async/await IS syntax sugar over this pattern
async function fetchUser() {
  const res = await fetch('/api/user');
  const user = await res.json();
  return user;
}
```

---

### Q: What are async generators?

```js
// Async generator — yields Promises
async function* streamData(url) {
  const response = await fetch(url);
  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    yield decoder.decode(value, { stream: true });
  }
}

// Consume with for-await-of
async function processStream() {
  for await (const chunk of streamData('/api/stream')) {
    console.log('Received chunk:', chunk);
  }
}
```

**Real-world uses:**
- Streaming API responses (SSE, chunked transfer)
- Paginated API data (fetch next page lazily)
- Processing large files line by line
- WebSocket message streams

```js
// Paginated API generator
async function* fetchAllPages(baseUrl) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const res = await fetch(`${baseUrl}?page=${page}`);
    const data = await res.json();
    yield data.items;
    hasMore = data.hasNextPage;
    page++;
  }
}

// Consumer pulls one page at a time
for await (const items of fetchAllPages('/api/products')) {
  renderProducts(items);
}
```

---

## 9. Web APIs & Timers

### Q: How do `setTimeout` and `setInterval` actually work?

```js
// setTimeout — schedules callback AFTER minimum delay
setTimeout(callback, 100);
// Does NOT guarantee execution at exactly 100ms
// Guarantees: at LEAST 100ms delay (could be more if call stack is busy)

// setInterval — schedules callback EVERY interval
const id = setInterval(callback, 1000);
// Problem: callback execution time adds up — drift occurs
```

### Q: Why might `setTimeout(fn, 0)` not execute immediately?

1. The callback goes to the **macrotask queue** — must wait for call stack + microtasks
2. **Minimum delay is ~4ms** in browsers (after 5+ nested timeouts — spec limitation)
3. **Background tabs** throttle timers to max 1 per second
4. **Busy call stack** delays execution further

```js
console.time('delay');
setTimeout(() => console.timeEnd('delay'), 0);
// Output: delay: ~4-10ms (NOT 0ms)
```

---

### Q: `setTimeout` vs `requestAnimationFrame`

| `setTimeout` | `requestAnimationFrame` |
|---|---|
| Runs after minimum delay | Runs at next frame paint (~16ms at 60fps) |
| May fire multiple times per frame or skip frames | Exactly once per frame |
| Runs in background tabs (throttled) | Paused in background tabs |
| Good for: delays, debouncing | Good for: animations, visual updates |

```js
// ❌ setTimeout animation — janky, not synced with refresh rate
function animate() {
  element.style.left = position + 'px';
  position++;
  setTimeout(animate, 16); // hoping for 60fps
}

// ✅ rAF animation — smooth, synced with display
function animate(timestamp) {
  element.style.left = position + 'px';
  position++;
  requestAnimationFrame(animate); // called right before paint
}
requestAnimationFrame(animate);
```

---

## 10. AbortController & Cancellation

### Q: How do you cancel async operations?

```js
// AbortController — cancel fetch requests
const controller = new AbortController();

// Start fetch with signal
fetch('/api/data', { signal: controller.signal })
  .then(res => res.json())
  .catch(err => {
    if (err.name === 'AbortError') {
      console.log('Request was cancelled');
    }
  });

// Cancel after 3 seconds
setTimeout(() => controller.abort(), 3000);

// Or cancel on user action
button.addEventListener('click', () => controller.abort());
```

### Advanced: Cancellable async operations

```js
function cancellable(asyncFn) {
  const controller = new AbortController();

  const promise = asyncFn(controller.signal);
  promise.cancel = () => controller.abort();

  return promise;
}

// Usage
const request = cancellable(async (signal) => {
  const res = await fetch('/api/heavy', { signal });
  return res.json();
});

// Later...
request.cancel(); // cancels the fetch
```

### Q: Implement a timeout wrapper for any Promise

```js
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) => {
    const id = setTimeout(() => {
      clearTimeout(id);
      reject(new Error(`Timed out after ${ms}ms`));
    }, ms);
  });

  return Promise.race([promise, timeout]);
}

// Usage
try {
  const data = await withTimeout(fetch('/api/slow'), 5000);
} catch (err) {
  console.log(err.message); // "Timed out after 5000ms"
}
```

### Cross-question: *"Does the original fetch cancel when the timeout rejects?"*

**No!** `Promise.race` only ignores the slower Promise — it doesn't cancel it. The fetch continues. To truly cancel, combine with `AbortController`:

```js
function withTimeout(promise, ms, controller) {
  const timeout = new Promise((_, reject) => {
    setTimeout(() => {
      controller.abort();  // actually cancel the request
      reject(new Error(`Timed out after ${ms}ms`));
    }, ms);
  });
  return Promise.race([promise, timeout]);
}

const controller = new AbortController();
await withTimeout(
  fetch('/api/slow', { signal: controller.signal }),
  5000,
  controller
);
```

---

## 11. Real-world Patterns

### Pattern 1: Debounced async search

```js
function useDebouncedSearch(searchFn, delay = 300) {
  let timerId;
  let abortController;

  return function (query) {
    return new Promise((resolve, reject) => {
      clearTimeout(timerId);
      abortController?.abort(); // cancel previous in-flight request

      timerId = setTimeout(async () => {
        abortController = new AbortController();
        try {
          const result = await searchFn(query, abortController.signal);
          resolve(result);
        } catch (err) {
          if (err.name !== 'AbortError') reject(err);
        }
      }, delay);
    });
  };
}
```

---

### Pattern 2: Lazy singleton (async initialization)

```js
class DatabaseConnection {
  static #instance = null;
  static #initPromise = null;

  static async getInstance() {
    if (this.#instance) return this.#instance;

    // Avoid multiple simultaneous initializations
    if (!this.#initPromise) {
      this.#initPromise = this.#create();
    }

    this.#instance = await this.#initPromise;
    return this.#instance;
  }

  static async #create() {
    const connection = new DatabaseConnection();
    await connection.connect();
    return connection;
  }
}

// Multiple callers — only ONE connection is created
const [conn1, conn2] = await Promise.all([
  DatabaseConnection.getInstance(),
  DatabaseConnection.getInstance(), // reuses the same init promise
]);
conn1 === conn2; // true
```

---

### Pattern 3: Event-to-Promise bridge

```js
function waitForEvent(target, event, options = {}) {
  const { timeout = 0 } = options;

  return new Promise((resolve, reject) => {
    const handler = (e) => {
      clearTimeout(timer);
      resolve(e);
    };

    let timer;
    if (timeout > 0) {
      timer = setTimeout(() => {
        target.removeEventListener(event, handler);
        reject(new Error(`Event "${event}" timed out after ${timeout}ms`));
      }, timeout);
    }

    target.addEventListener(event, handler, { once: true });
  });
}

// Usage
const event = await waitForEvent(button, 'click', { timeout: 10000 });
console.log('Button was clicked at', event.clientX, event.clientY);

// Wait for image to load
const img = new Image();
img.src = 'photo.jpg';
await waitForEvent(img, 'load', { timeout: 5000 });
```

---

### Pattern 4: Deferred Promise

```js
function defer() {
  let resolve, reject;
  const promise = new Promise((res, rej) => {
    resolve = res;
    reject = rej;
  });
  return { promise, resolve, reject };
}

// Usage — resolve from outside the Promise constructor
const { promise, resolve } = defer();

// Pass promise to consumer
showModal(promise);

// Resolve later when user clicks
button.addEventListener('click', () => resolve('confirmed'));
```

---

### Pattern 5: Async mutex (lock)

```js
class Mutex {
  #locked = false;
  #waiting = [];

  async acquire() {
    if (!this.#locked) {
      this.#locked = true;
      return;
    }

    // Wait in line
    await new Promise(resolve => this.#waiting.push(resolve));
  }

  release() {
    if (this.#waiting.length > 0) {
      const next = this.#waiting.shift();
      next(); // wake up next waiter
    } else {
      this.#locked = false;
    }
  }
}

// Usage — serialize access to a shared resource
const mutex = new Mutex();

async function criticalSection() {
  await mutex.acquire();
  try {
    await updateSharedResource();
  } finally {
    mutex.release(); // always release in finally
  }
}
```

---

## 12. MANG Scenarios & Cross-Questions

---

### Scenario 1: Race condition in autocomplete

> **"Users report stale search results — typing 'react' shows results for 'rea' instead. Diagnose and fix."**

**Problem:** Responses arrive out of order. Request for 'rea' takes 500ms, request for 'react' takes 200ms. 'react' result renders first, then 'rea' result overwrites it.

```js
// ❌ Race condition
async function search(query) {
  const res = await fetch(`/api/search?q=${query}`);
  const data = await res.json();
  setResults(data); // stale result may overwrite newer one!
}

// ✅ Fix 1: AbortController — cancel previous request
let controller;
async function search(query) {
  controller?.abort();
  controller = new AbortController();
  try {
    const res = await fetch(`/api/search?q=${query}`, { signal: controller.signal });
    const data = await res.json();
    setResults(data);
  } catch (err) {
    if (err.name !== 'AbortError') throw err;
  }
}

// ✅ Fix 2: Request ID — ignore stale responses
let latestRequestId = 0;
async function search(query) {
  const requestId = ++latestRequestId;
  const res = await fetch(`/api/search?q=${query}`);
  const data = await res.json();
  if (requestId === latestRequestId) {
    setResults(data); // only update if this is the latest request
  }
}
```

### Cross-question: *"Which fix is better — AbortController or request ID?"*

**AbortController is better** — it not only prevents stale updates but also **cancels the network request**, saving bandwidth. The request ID approach lets the network request complete wastefully.

---

### Scenario 2: Batch API with concurrency limit

> **"You need to upload 500 images. The API allows max 10 concurrent uploads. If one fails, retry it up to 3 times without blocking others."**

```js
async function uploadBatch(images, { maxConcurrent = 10, maxRetries = 3 } = {}) {
  const results = new Array(images.length);
  let index = 0;

  async function worker() {
    while (index < images.length) {
      const i = index++;
      results[i] = await retry(
        () => uploadImage(images[i]),
        { retries: maxRetries }
      );
    }
  }

  // Spawn workers
  const workers = Array.from({ length: maxConcurrent }, () => worker());
  await Promise.allSettled(workers);

  return results;
}

const results = await uploadBatch(images, { maxConcurrent: 10, maxRetries: 3 });
const failed = results.filter(r => r.status === 'rejected');
console.log(`${failed.length} uploads failed permanently`);
```

---

### Scenario 3: Memory leak from unawaited Promises

> **"Your Node.js server's memory grows over time. You suspect a Promise-related leak. How do you find it?"**

**Common causes:**
```js
// Leak 1: Promises never settled — handlers stay in memory forever
function processBatch(items) {
  items.forEach(item => {
    new Promise(resolve => {
      // resolve is never called if item is invalid → Promise stays pending
      // .then handler stays in memory
      if (item.valid) resolve(process(item));
    }).then(result => save(result));
  });
}

// Leak 2: Event listeners accumulating
app.get('/data', async (req, res) => {
  const listener = (data) => res.json(data);
  emitter.on('data', listener); // never removed!
  // After request ends, listener still exists
  // Fix: emitter.once() or explicit removeListener
});

// Leak 3: Closures in long-lived Promises
const cache = new Map();
async function getData(key) {
  if (!cache.has(key)) {
    const data = await fetchLargeDataset(key);
    cache.set(key, data); // data stays in memory forever
    // Fix: use LRU cache with TTL
  }
  return cache.get(key);
}
```

---

### Scenario 4: Implement `Promise.map` with ordered results

> **"Given an array and an async mapper, return results in the same order as input, with max concurrency."**

```js
async function promiseMap(arr, mapFn, concurrency = Infinity) {
  const results = new Array(arr.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < arr.length) {
      const index = nextIndex++;
      results[index] = await mapFn(arr[index], index);
    }
  }

  const workerCount = Math.min(concurrency, arr.length);
  await Promise.all(
    Array.from({ length: workerCount }, () => worker())
  );

  return results;
}

// Usage
const usernames = ['alice', 'bob', 'charlie', /* ...100 more */];
const profiles = await promiseMap(
  usernames,
  async (name) => fetchProfile(name),
  5 // max 5 concurrent
);
// profiles[0] = alice's profile, profiles[1] = bob's profile (order preserved)
```

---

### Scenario 5: Deadlock detection

> **"Two async functions each wait for the other to complete. How do you detect and prevent this?"**

```js
// ❌ Deadlock — A waits for B, B waits for A
const lockA = defer();
const lockB = defer();

async function taskA() {
  await lockB.promise; // waiting for B to finish
  lockA.resolve();
}

async function taskB() {
  await lockA.promise; // waiting for A to finish
  lockB.resolve();
}

// Both wait forever!
await Promise.all([taskA(), taskB()]); // hangs
```

**Prevention:**
1. **Always acquire locks in a consistent order** (alphabetical, by ID)
2. **Use timeouts** on all awaits — fail rather than hang
3. **Avoid circular dependencies** between async operations
4. **Use a task scheduler** that detects dependency cycles

```js
// Fix: timeout to detect deadlock
const result = await Promise.race([
  lockA.promise,
  new Promise((_, reject) => setTimeout(() => reject(new Error('Deadlock detected')), 5000)),
]);
```

---

## Quick Reference — Async Mental Model

```
When is my code running?

  SYNCHRONOUS               MICROTASK                    MACROTASK
  ─────────────             ──────────                   ──────────
  Function calls            Promise.then/catch/finally   setTimeout
  Variable assignments      await (continuation)         setInterval
  console.log               queueMicrotask               requestAnimationFrame
  throw                     MutationObserver             I/O callbacks
  new Promise(executor)     process.nextTick (Node)      UI events (click, scroll)

  Runs FIRST ──────────→    Runs SECOND ────────────→    Runs THIRD
```

```
Choosing the right combinator:

  Need ALL results?
    ├─ All must succeed → Promise.all
    └─ Some may fail OK → Promise.allSettled

  Need FIRST result?
    ├─ First settle (win or lose) → Promise.race
    └─ First SUCCESS only → Promise.any
```

---

*Last updated: February 2026 | ES2024 · Node.js 22*
