# JavaScript Polyfills — MANG Level Interview (30 Implementations)

> Hand-written polyfills for Array, Object, Promise, Function, String, and utility methods  
> Each includes: implementation · explanation · edge cases · cross-questions

---

## Table of Contents

**Array Methods**
1. [`Array.prototype.map`](#1-arrayprototypemap)
2. [`Array.prototype.filter`](#2-arrayprototypefilter)
3. [`Array.prototype.reduce`](#3-arrayprototypereduce)
4. [`Array.prototype.forEach`](#4-arrayprototypeforeach)
5. [`Array.prototype.find`](#5-arrayprototypefind)
6. [`Array.prototype.findIndex`](#6-arrayprototypefindindex)
7. [`Array.prototype.some`](#7-arrayprototypesome)
8. [`Array.prototype.every`](#8-arrayprototypeevery)
9. [`Array.prototype.flat`](#9-arrayprototypeflat)
10. [`Array.prototype.flatMap`](#10-arrayprototypeflatmap)
11. [`Array.prototype.includes`](#11-arrayprototypeincludes)
12. [`Array.from`](#12-arrayfrom)

**Function Methods**
13. [`Function.prototype.bind`](#13-functionprototypebind)
14. [`Function.prototype.call`](#14-functionprototypecall)
15. [`Function.prototype.apply`](#15-functionprototypeapply)

**Object Methods**
16. [`Object.assign`](#16-objectassign)
17. [`Object.keys`](#17-objectkeys)
18. [`Object.values`](#18-objectvalues)
19. [`Object.entries`](#19-objectentries)
20. [`Object.freeze`](#20-objectfreeze)
21. [`Object.create`](#21-objectcreate)

**Promise**
22. [`Promise`](#22-promise)
23. [`Promise.all`](#23-promiseall)
24. [`Promise.allSettled`](#24-promiseallsettled)
25. [`Promise.race`](#25-promiserace)
26. [`Promise.any`](#26-promiseany)

**Utilities**
27. [`debounce`](#27-debounce)
28. [`throttle`](#28-throttle)
29. [`deepClone`](#29-deepclone)
30. [`curry`](#30-curry)

---

## Array Methods

---

### 1. `Array.prototype.map`

Creates a new array by calling a function on every element.

```js
Array.prototype.myMap = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this) {  // skip holes in sparse arrays: [1, , 3]
      result[i] = callback.call(thisArg, this[i], i, this);
    }
  }
  return result;
};

// Usage
[1, 2, 3].myMap(x => x * 2); // [2, 4, 6]
[1, , 3].myMap(x => x * 2);  // [2, empty, 6] — preserves holes
```

**Cross-question: *"Why `i in this` instead of `this[i] !== undefined`?"***

Sparse arrays have **holes** — indices that don't exist at all. `i in this` returns `false` for holes but `true` for `undefined` values. `this[i] !== undefined` would incorrectly skip actual `undefined` values.

```js
const arr = [1, , undefined, 3];
0 in arr;  // true  (value: 1)
1 in arr;  // false (hole — skip)
2 in arr;  // true  (value: undefined — DO process)
```

---

### 2. `Array.prototype.filter`

Returns a new array with elements that pass the test.

```js
Array.prototype.myFilter = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this && callback.call(thisArg, this[i], i, this)) {
      result.push(this[i]);
    }
  }
  return result;
};

// Usage
[1, 2, 3, 4, 5].myFilter(x => x % 2 === 0); // [2, 4]
```

---

### 3. `Array.prototype.reduce`

Reduces an array to a single value by accumulating.

```js
Array.prototype.myReduce = function (callback, initialValue) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  let accumulator;
  let startIndex = 0;

  if (arguments.length >= 2) {
    // initialValue provided
    accumulator = initialValue;
  } else {
    // No initialValue — use first non-hole element
    if (this.length === 0) throw new TypeError('Reduce of empty array with no initial value');

    // Find first non-hole element
    let found = false;
    for (let i = 0; i < this.length; i++) {
      if (i in this) {
        accumulator = this[i];
        startIndex = i + 1;
        found = true;
        break;
      }
    }
    if (!found) throw new TypeError('Reduce of empty array with no initial value');
  }

  for (let i = startIndex; i < this.length; i++) {
    if (i in this) {
      accumulator = callback(accumulator, this[i], i, this);
    }
  }

  return accumulator;
};

// Usage
[1, 2, 3, 4].myReduce((acc, val) => acc + val, 0);       // 10
[1, 2, 3, 4].myReduce((acc, val) => acc + val);           // 10 (1 is initial)
['a', 'b', 'a'].myReduce((acc, val) => {
  acc[val] = (acc[val] || 0) + 1;
  return acc;
}, {});  // { a: 2, b: 1 }
```

**Cross-question: *"What happens when you call reduce on an empty array without initialValue?"***

It throws `TypeError: Reduce of empty array with no initial value`. This is a common interview gotcha — you must handle this edge case in your polyfill.

---

### 4. `Array.prototype.forEach`

Executes a function on every element. Returns `undefined` (cannot be chained).

```js
Array.prototype.myForEach = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      callback.call(thisArg, this[i], i, this);
    }
  }
  // returns undefined — this is intentional
};
```

**Cross-question: *"Can you break out of forEach?"***

**No.** `forEach` runs the callback for every element — there's no way to stop it early (unlike `for...of` or `some`/`every`). This is a key difference from `for` loops and a common interview question.

---

### 5. `Array.prototype.find`

Returns the **first element** that satisfies the test, or `undefined`.

```js
Array.prototype.myFind = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  for (let i = 0; i < this.length; i++) {
    if (i in this && callback.call(thisArg, this[i], i, this)) {
      return this[i];  // return the VALUE
    }
  }
  return undefined;
};

// Usage
[5, 12, 8, 130].myFind(x => x > 10); // 12
```

---

### 6. `Array.prototype.findIndex`

Returns the **index** of the first matching element, or `-1`.

```js
Array.prototype.myFindIndex = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  for (let i = 0; i < this.length; i++) {
    if (i in this && callback.call(thisArg, this[i], i, this)) {
      return i;  // return the INDEX
    }
  }
  return -1;
};
```

---

### 7. `Array.prototype.some`

Returns `true` if **at least one** element passes the test.

```js
Array.prototype.mySome = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  for (let i = 0; i < this.length; i++) {
    if (i in this && callback.call(thisArg, this[i], i, this)) {
      return true;  // short-circuit on first match
    }
  }
  return false;
};

// Usage
[1, 2, 3].mySome(x => x > 2); // true
[1, 2, 3].mySome(x => x > 5); // false
```

---

### 8. `Array.prototype.every`

Returns `true` if **all** elements pass the test.

```js
Array.prototype.myEvery = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  for (let i = 0; i < this.length; i++) {
    if (i in this && !callback.call(thisArg, this[i], i, this)) {
      return false;  // short-circuit on first failure
    }
  }
  return true;
};

// Usage
[2, 4, 6].myEvery(x => x % 2 === 0); // true
[2, 3, 6].myEvery(x => x % 2 === 0); // false
[].myEvery(x => false);               // true (vacuous truth)
```

**Cross-question: *"Why does `[].every(() => false)` return `true`?"***

This is **vacuous truth** — "every element in an empty set satisfies any condition" is logically true because there are no counterexamples. Same in math: "all unicorns can fly" is true because there are no unicorns to disprove it.

---

### 9. `Array.prototype.flat`

Flattens nested arrays to a specified depth.

```js
Array.prototype.myFlat = function (depth = 1) {
  const result = [];

  const flatten = (arr, currentDepth) => {
    for (let i = 0; i < arr.length; i++) {
      if (i in arr) {
        if (Array.isArray(arr[i]) && currentDepth < depth) {
          flatten(arr[i], currentDepth + 1);
        } else {
          result.push(arr[i]);
        }
      }
    }
  };

  flatten(this, 0);
  return result;
};

// Usage
[1, [2, [3, [4]]]].myFlat();       // [1, 2, [3, [4]]]   — depth 1
[1, [2, [3, [4]]]].myFlat(2);      // [1, 2, 3, [4]]     — depth 2
[1, [2, [3, [4]]]].myFlat(Infinity); // [1, 2, 3, 4]     — fully flat
```

**Cross-question: *"Can you write flat without recursion?"***

```js
// Iterative with stack
Array.prototype.myFlatIterative = function (depth = 1) {
  const stack = this.map(item => [item, 0]); // [value, currentDepth]
  const result = [];

  while (stack.length) {
    const [item, d] = stack.shift();
    if (Array.isArray(item) && d < depth) {
      stack.unshift(...item.map(i => [i, d + 1]));
    } else {
      result.push(item);
    }
  }
  return result;
};
```

---

### 10. `Array.prototype.flatMap`

Maps each element, then flattens the result by 1 level. More efficient than `.map().flat()`.

```js
Array.prototype.myFlatMap = function (callback, thisArg) {
  if (typeof callback !== 'function') throw new TypeError(callback + ' is not a function');

  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this) {
      const mapped = callback.call(thisArg, this[i], i, this);
      if (Array.isArray(mapped)) {
        result.push(...mapped); // flatten 1 level
      } else {
        result.push(mapped);
      }
    }
  }
  return result;
};

// Usage — split words into characters
['hello', 'world'].myFlatMap(word => word.split(''));
// ['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']

// Usage — filter + map in one pass
[1, 2, 3, 4].myFlatMap(x => x % 2 === 0 ? [x * 2] : []);
// [4, 8] — empty array removes the element, single array keeps it
```

---

### 11. `Array.prototype.includes`

Checks if array contains a value. Uses **SameValueZero** comparison (handles `NaN`).

```js
Array.prototype.myIncludes = function (searchElement, fromIndex = 0) {
  const len = this.length;
  let start = fromIndex >= 0 ? fromIndex : Math.max(len + fromIndex, 0);

  for (let i = start; i < len; i++) {
    // SameValueZero: like === but NaN === NaN is true
    if (this[i] === searchElement || (Number.isNaN(this[i]) && Number.isNaN(searchElement))) {
      return true;
    }
  }
  return false;
};

// Usage
[1, 2, NaN].myIncludes(NaN);     // true  (unlike indexOf which returns -1)
[1, 2, 3].myIncludes(2, 2);      // false (search starts at index 2)
[1, 2, 3].myIncludes(1, -2);     // false (search starts at index 1)
```

**Cross-question: *"Why does `includes` find NaN but `indexOf` doesn't?"***

`indexOf` uses **strict equality** (`===`), where `NaN !== NaN`. `includes` uses **SameValueZero**, where `NaN` equals `NaN`. This was a deliberate improvement when `includes` was added in ES2016.

---

### 12. `Array.from`

Creates an array from array-like or iterable objects.

```js
Array.myFrom = function (arrayLike, mapFn, thisArg) {
  if (arrayLike == null) throw new TypeError('Cannot convert undefined or null to object');

  const result = [];

  // Handle iterables (Sets, Maps, generators)
  if (arrayLike[Symbol.iterator]) {
    for (const item of arrayLike) {
      result.push(mapFn ? mapFn.call(thisArg, item, result.length) : item);
    }
    return result;
  }

  // Handle array-likes (arguments, NodeList — have .length)
  const len = arrayLike.length >>> 0; // convert to uint32
  for (let i = 0; i < len; i++) {
    result.push(mapFn ? mapFn.call(thisArg, arrayLike[i], i) : arrayLike[i]);
  }
  return result;
};

// Usage
Array.myFrom('hello');                    // ['h', 'e', 'l', 'l', 'o']
Array.myFrom([1, 2, 3], x => x * 2);     // [2, 4, 6]
Array.myFrom({ length: 3 }, (_, i) => i); // [0, 1, 2]
Array.myFrom(new Set([1, 2, 2, 3]));     // [1, 2, 3]
```

---

## Function Methods

---

### 13. `Function.prototype.bind`

Returns a new function with `this` bound and optionally partial arguments.

```js
Function.prototype.myBind = function (context, ...boundArgs) {
  if (typeof this !== 'function') throw new TypeError('Bind must be called on a function');

  const originalFn = this;

  const boundFn = function (...callArgs) {
    // Handle `new` operator — when bound function is used as constructor
    const isNew = this instanceof boundFn;
    return originalFn.apply(
      isNew ? this : context,       // if called with new, use new instance as this
      [...boundArgs, ...callArgs]   // merge partial + call-time args
    );
  };

  // Maintain prototype chain for `new`
  if (originalFn.prototype) {
    boundFn.prototype = Object.create(originalFn.prototype);
  }

  return boundFn;
};

// Usage
const obj = { name: 'Alice' };
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const boundGreet = greet.myBind(obj, 'Hello');
boundGreet('!');  // "Hello, Alice!"

// Partial application
const double = multiply.myBind(null, 2);
double(5); // 10
```

**Cross-question: *"Why handle `new` in bind?"***

If someone does `new boundFn()`, the `this` inside should point to the newly created object — not the bound context. The native `bind` handles this, so our polyfill must too.

```js
function Person(name) { this.name = name; }
const BoundPerson = Person.myBind({ ignored: true });
const p = new BoundPerson('Bob');
p.name; // 'Bob' — { ignored: true } is correctly ignored when using new
```

---

### 14. `Function.prototype.call`

Calls a function with a given `this` and individual arguments.

```js
Function.prototype.myCall = function (context, ...args) {
  if (typeof this !== 'function') throw new TypeError(this + ' is not a function');

  // Handle null/undefined → globalThis (window in browser)
  context = context != null ? Object(context) : globalThis;

  // Attach function temporarily to context
  const uniqueKey = Symbol('fn');
  context[uniqueKey] = this;

  // Call as method of context → `this` is context
  const result = context[uniqueKey](...args);

  // Clean up
  delete context[uniqueKey];
  return result;
};

// Usage
function greet(greeting) {
  return `${greeting}, ${this.name}`;
}
greet.myCall({ name: 'Alice' }, 'Hello'); // "Hello, Alice"
```

**Cross-question: *"Why Symbol instead of a string key?"***

A string key like `'__fn__'` could collide with an existing property on the context object. `Symbol()` creates a **guaranteed unique** key — zero collision risk.

---

### 15. `Function.prototype.apply`

Same as `call`, but takes arguments as an **array**.

```js
Function.prototype.myApply = function (context, argsArray) {
  if (typeof this !== 'function') throw new TypeError(this + ' is not a function');

  context = context != null ? Object(context) : globalThis;

  const uniqueKey = Symbol('fn');
  context[uniqueKey] = this;

  const args = argsArray ? [...argsArray] : [];
  const result = context[uniqueKey](...args);

  delete context[uniqueKey];
  return result;
};

// Usage
Math.max.myApply(null, [1, 5, 3]); // 5
```

**Cross-question: *"What's the difference between `call` and `apply`?"***

Only the arguments format: `call(ctx, a, b, c)` vs `apply(ctx, [a, b, c])`. With spread (`...`), `call` is usually preferred:
```js
fn.call(ctx, ...args); // modern alternative to fn.apply(ctx, args)
```

---

## Object Methods

---

### 16. `Object.assign`

Copies own enumerable properties from source(s) to target.

```js
Object.myAssign = function (target, ...sources) {
  if (target == null) throw new TypeError('Cannot convert undefined or null to object');

  const to = Object(target);

  for (const source of sources) {
    if (source != null) {
      // Own enumerable properties (including Symbols)
      for (const key of [...Object.keys(source), ...Object.getOwnPropertySymbols(source)]) {
        if (Object.prototype.hasOwnProperty.call(source, key)) {
          to[key] = source[key]; // shallow copy
        }
      }
    }
  }
  return to;
};

// Usage
Object.myAssign({}, { a: 1 }, { b: 2 }, { a: 3 }); // { a: 3, b: 2 }
```

**Cross-question: *"Is Object.assign deep or shallow?"***

**Shallow.** Nested objects are copied **by reference**, not cloned:
```js
const obj = { nested: { x: 1 } };
const copy = Object.assign({}, obj);
copy.nested.x = 99;
obj.nested.x; // 99 — same reference!
```

---

### 17. `Object.keys`

Returns an array of own enumerable **string** property names.

```js
Object.myKeys = function (obj) {
  if (obj == null) throw new TypeError('Cannot convert undefined or null to object');

  const result = [];
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      result.push(key);
    }
  }
  return result;
};

// Usage
Object.myKeys({ a: 1, b: 2, c: 3 }); // ['a', 'b', 'c']
```

---

### 18. `Object.values`

```js
Object.myValues = function (obj) {
  if (obj == null) throw new TypeError('Cannot convert undefined or null to object');

  const result = [];
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      result.push(obj[key]);
    }
  }
  return result;
};

// Usage
Object.myValues({ a: 1, b: 2 }); // [1, 2]
```

---

### 19. `Object.entries`

```js
Object.myEntries = function (obj) {
  if (obj == null) throw new TypeError('Cannot convert undefined or null to object');

  const result = [];
  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      result.push([key, obj[key]]);
    }
  }
  return result;
};

// Usage
Object.myEntries({ a: 1, b: 2 }); // [['a', 1], ['b', 2]]
```

---

### 20. `Object.freeze`

Makes an object immutable (shallow).

```js
Object.myFreeze = function (obj) {
  if (typeof obj !== 'object' || obj === null) return obj;

  // Prevent adding/removing properties
  Object.preventExtensions(obj);

  // Make all own properties non-writable and non-configurable
  Object.getOwnPropertyNames(obj).forEach(name => {
    const desc = Object.getOwnPropertyDescriptor(obj, name);
    if (desc && 'value' in desc) {
      Object.defineProperty(obj, name, {
        writable: false,
        configurable: false,
      });
    }
  });

  return obj;
};

// Usage
const frozen = Object.myFreeze({ a: 1, b: { c: 2 } });
frozen.a = 99;     // silently fails (or throws in strict mode)
frozen.b.c = 99;   // ✅ WORKS — freeze is shallow!
```

**Cross-question: *"How would you deep freeze?"***

```js
function deepFreeze(obj) {
  Object.freeze(obj);
  Object.getOwnPropertyNames(obj).forEach(name => {
    const value = obj[name];
    if (typeof value === 'object' && value !== null && !Object.isFrozen(value)) {
      deepFreeze(value);
    }
  });
  return obj;
}
```

---

### 21. `Object.create`

Creates a new object with the specified prototype.

```js
Object.myCreate = function (proto, propertiesObject) {
  if (typeof proto !== 'object' && typeof proto !== 'function' && proto !== null) {
    throw new TypeError('Object prototype may only be an Object or null');
  }

  function F() {}
  F.prototype = proto;
  const obj = new F();

  if (propertiesObject !== undefined) {
    Object.defineProperties(obj, propertiesObject);
  }

  return obj;
};

// Usage
const animal = { speak() { return `${this.name} makes a sound`; } };
const dog = Object.myCreate(animal);
dog.name = 'Rex';
dog.speak(); // "Rex makes a sound"
```

**Cross-question: *"What is `Object.create(null)` used for?"***

Creates an object with **no prototype** — no `toString`, `hasOwnProperty`, nothing. Perfect for **pure dictionaries/maps** where you don't want inherited properties interfering:
```js
const dict = Object.create(null);
dict['hasOwnProperty'] = 'safe'; // no collision with Object.prototype.hasOwnProperty
```

---

## Promise

---

### 22. `Promise`

Full Promise implementation following Promises/A+ spec:

```js
class MyPromise {
  #state = 'pending';  // 'pending' | 'fulfilled' | 'rejected'
  #value = undefined;
  #handlers = [];

  constructor(executor) {
    const resolve = (value) => {
      if (this.#state !== 'pending') return; // can only transition once
      if (value instanceof MyPromise) {
        value.then(resolve, reject);         // resolve with another promise
        return;
      }
      this.#state = 'fulfilled';
      this.#value = value;
      this.#handlers.forEach(h => h.onFulfilled(value));
    };

    const reject = (reason) => {
      if (this.#state !== 'pending') return;
      this.#state = 'rejected';
      this.#value = reason;
      this.#handlers.forEach(h => h.onRejected(reason));
    };

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handle = () => {
        try {
          if (this.#state === 'fulfilled') {
            const result = typeof onFulfilled === 'function' ? onFulfilled(this.#value) : this.#value;
            resolve(result);
          } else if (this.#state === 'rejected') {
            if (typeof onRejected === 'function') {
              resolve(onRejected(this.#value));
            } else {
              reject(this.#value);
            }
          }
        } catch (error) {
          reject(error);
        }
      };

      if (this.#state === 'pending') {
        this.#handlers.push({
          onFulfilled: () => queueMicrotask(handle),
          onRejected: () => queueMicrotask(handle),
        });
      } else {
        queueMicrotask(handle); // always async (microtask)
      }
    });
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }

  finally(onFinally) {
    return this.then(
      value => MyPromise.resolve(onFinally()).then(() => value),
      reason => MyPromise.resolve(onFinally()).then(() => { throw reason; })
    );
  }

  static resolve(value) {
    if (value instanceof MyPromise) return value;
    return new MyPromise(resolve => resolve(value));
  }

  static reject(reason) {
    return new MyPromise((_, reject) => reject(reason));
  }
}
```

**Cross-question: *"Why `queueMicrotask` instead of `setTimeout`?"***

The Promises/A+ spec requires `.then` callbacks to run **asynchronously** but as soon as possible. `queueMicrotask` runs after the current call stack but **before** any macrotasks (`setTimeout`, `setInterval`). This matches native Promise behavior.

```
Call stack → Microtask queue (Promise, queueMicrotask) → Macrotask queue (setTimeout)
```

---

### 23. `Promise.all`

Resolves when **all** promises resolve. Rejects immediately on **first** rejection.

```js
MyPromise.all = function (promises) {
  return new MyPromise((resolve, reject) => {
    const results = [];
    let completed = 0;
    const items = [...promises]; // handle iterables

    if (items.length === 0) return resolve([]);

    items.forEach((promise, index) => {
      MyPromise.resolve(promise).then(
        value => {
          results[index] = value;  // maintain order (not completion order)
          completed++;
          if (completed === items.length) resolve(results);
        },
        reject  // first rejection rejects the whole thing
      );
    });
  });
};

// Usage
MyPromise.all([
  fetch('/api/users'),
  fetch('/api/posts'),
  fetch('/api/comments'),
]).then(([users, posts, comments]) => {
  // all resolved — results in ORDER, not completion order
});
```

**Cross-question: *"If one promise rejects, do the others cancel?"***

**No.** JavaScript promises are not cancellable. The other fetch requests still complete — their results are just ignored. To actually cancel, you need `AbortController`.

---

### 24. `Promise.allSettled`

Waits for **all** promises to settle (resolve OR reject). Never rejects.

```js
MyPromise.allSettled = function (promises) {
  return new MyPromise((resolve) => {
    const results = [];
    let completed = 0;
    const items = [...promises];

    if (items.length === 0) return resolve([]);

    items.forEach((promise, index) => {
      MyPromise.resolve(promise).then(
        value => {
          results[index] = { status: 'fulfilled', value };
          completed++;
          if (completed === items.length) resolve(results);
        },
        reason => {
          results[index] = { status: 'rejected', reason };
          completed++;
          if (completed === items.length) resolve(results);
        }
      );
    });
  });
};

// Usage — fire multiple API calls, handle each result independently
const results = await MyPromise.allSettled([
  fetch('/api/users'),
  fetch('/api/risky-endpoint'),
]);
// results: [
//   { status: 'fulfilled', value: Response },
//   { status: 'rejected', reason: Error }
// ]
```

---

### 25. `Promise.race`

Resolves/rejects with the **first** promise to settle.

```js
MyPromise.race = function (promises) {
  return new MyPromise((resolve, reject) => {
    const items = [...promises];
    if (items.length === 0) return; // race with no contestants never settles

    items.forEach(promise => {
      MyPromise.resolve(promise).then(resolve, reject);
      // First one to call resolve/reject wins — rest are ignored
    });
  });
};

// Usage — timeout pattern
MyPromise.race([
  fetch('/api/data'),
  new MyPromise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000)),
]);
```

---

### 26. `Promise.any`

Resolves with the **first fulfilled** promise. Rejects only if **all** reject.

```js
MyPromise.any = function (promises) {
  return new MyPromise((resolve, reject) => {
    const errors = [];
    let rejected = 0;
    const items = [...promises];

    if (items.length === 0) {
      return reject(new AggregateError([], 'All promises were rejected'));
    }

    items.forEach((promise, index) => {
      MyPromise.resolve(promise).then(
        resolve, // first fulfillment wins
        reason => {
          errors[index] = reason;
          rejected++;
          if (rejected === items.length) {
            reject(new AggregateError(errors, 'All promises were rejected'));
          }
        }
      );
    });
  });
};

// Usage — try multiple CDNs, use whichever responds first
MyPromise.any([
  fetch('https://cdn1.example.com/data.json'),
  fetch('https://cdn2.example.com/data.json'),
  fetch('https://cdn3.example.com/data.json'),
]).then(fastest => fastest.json());
```

**Cross-question: *"`Promise.race` vs `Promise.any` — what's the difference?"***

| `Promise.race` | `Promise.any` |
|---|---|
| First **settled** (fulfilled OR rejected) | First **fulfilled** only |
| One rejection immediately rejects | Rejections are collected, only rejects if ALL reject |
| Like "first across the finish line" | Like "first to succeed" |

---

## Utilities

---

### 27. `debounce`

Delays execution until a pause in calls.

```js
function debounce(fn, delay, { leading = false, trailing = true } = {}) {
  let timerId = null;
  let lastArgs = null;

  function debounced(...args) {
    lastArgs = args;
    const isFirstCall = timerId === null;

    clearTimeout(timerId);

    // Leading edge: fire immediately on first call
    if (leading && isFirstCall) {
      fn.apply(this, args);
    }

    timerId = setTimeout(() => {
      // Trailing edge: fire after delay
      if (trailing && lastArgs) {
        fn.apply(this, lastArgs);
      }
      timerId = null;
      lastArgs = null;
    }, delay);
  }

  debounced.cancel = () => {
    clearTimeout(timerId);
    timerId = null;
    lastArgs = null;
  };

  debounced.flush = () => {
    if (timerId) {
      fn.apply(undefined, lastArgs);
      debounced.cancel();
    }
  };

  return debounced;
}

// Usage
const debouncedSearch = debounce((query) => fetchResults(query), 300);
input.addEventListener('input', (e) => debouncedSearch(e.target.value));

// With leading: true — fires on FIRST call, then waits
const debouncedSave = debounce(saveData, 1000, { leading: true, trailing: true });
```

---

### 28. `throttle`

Ensures function executes at most once per interval.

```js
function throttle(fn, interval, { leading = true, trailing = true } = {}) {
  let timerId = null;
  let lastArgs = null;
  let lastCallTime = 0;

  function throttled(...args) {
    const now = Date.now();
    const elapsed = now - lastCallTime;

    lastArgs = args;

    if (!lastCallTime && !leading) {
      lastCallTime = now;
    }

    const remaining = interval - elapsed;

    if (remaining <= 0) {
      // Enough time has passed — execute
      clearTimeout(timerId);
      timerId = null;
      lastCallTime = now;
      fn.apply(this, args);
    } else if (!timerId && trailing) {
      // Schedule trailing call
      timerId = setTimeout(() => {
        lastCallTime = leading ? Date.now() : 0;
        timerId = null;
        fn.apply(this, lastArgs);
        lastArgs = null;
      }, remaining);
    }
  }

  throttled.cancel = () => {
    clearTimeout(timerId);
    timerId = null;
    lastArgs = null;
    lastCallTime = 0;
  };

  return throttled;
}

// Usage
const throttledScroll = throttle(() => {
  console.log('Scroll position:', window.scrollY);
}, 200);
window.addEventListener('scroll', throttledScroll);
```

---

### 29. `deepClone`

Deep clones an object, handling circular references, types, and edge cases.

```js
function deepClone(obj, seen = new WeakMap()) {
  // Primitives and null
  if (obj === null || typeof obj !== 'object') return obj;

  // Handle circular references
  if (seen.has(obj)) return seen.get(obj);

  // Handle Date
  if (obj instanceof Date) return new Date(obj.getTime());

  // Handle RegExp
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);

  // Handle Map
  if (obj instanceof Map) {
    const mapClone = new Map();
    seen.set(obj, mapClone);
    obj.forEach((value, key) => mapClone.set(deepClone(key, seen), deepClone(value, seen)));
    return mapClone;
  }

  // Handle Set
  if (obj instanceof Set) {
    const setClone = new Set();
    seen.set(obj, setClone);
    obj.forEach(value => setClone.add(deepClone(value, seen)));
    return setClone;
  }

  // Handle Array and plain objects
  const clone = Array.isArray(obj) ? [] : Object.create(Object.getPrototypeOf(obj));
  seen.set(obj, clone);

  // Copy own properties (including Symbols)
  for (const key of Reflect.ownKeys(obj)) {
    const desc = Object.getOwnPropertyDescriptor(obj, key);
    if (desc.value !== undefined) {
      clone[key] = deepClone(desc.value, seen);
    } else {
      Object.defineProperty(clone, key, desc); // preserve getters/setters
    }
  }

  return clone;
}

// Usage
const original = {
  date: new Date(),
  regex: /hello/gi,
  nested: { a: [1, 2, { b: 3 }] },
  map: new Map([['key', 'value']]),
};
original.circular = original; // circular reference!

const cloned = deepClone(original);
cloned.nested.a[2].b = 99;
original.nested.a[2].b; // still 3 ✅
```

**Cross-question: *"What about `structuredClone`?"***

`structuredClone` is a **native API** (2022+) that does deep cloning with circular reference support. Use it in production. The polyfill above is for interview knowledge:

```js
const cloned = structuredClone(original);
```

Limitations of `structuredClone`: can't clone **functions**, **DOM nodes**, or **property descriptors** (getters/setters).

---

### 30. `curry`

Transforms `fn(a, b, c)` into `fn(a)(b)(c)` — supports partial application.

```js
function curry(fn) {
  const arity = fn.length; // number of expected arguments

  return function curried(...args) {
    // If enough args collected, call original function
    if (args.length >= arity) {
      return fn.apply(this, args);
    }

    // Otherwise, return a function that collects more args
    return function (...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

// Usage
const add = curry((a, b, c) => a + b + c);

add(1, 2, 3);    // 6 — all at once
add(1)(2)(3);     // 6 — one at a time
add(1, 2)(3);     // 6 — partial application
add(1)(2, 3);     // 6 — mixed

// Real-world: reusable transformers
const multiply = curry((a, b) => a * b);
const double = multiply(2);
const triple = multiply(3);
[1, 2, 3].map(double); // [2, 4, 6]
[1, 2, 3].map(triple); // [3, 6, 9]
```

**Cross-question: *"What's the difference between currying and partial application?"***

| Currying | Partial Application |
|---|---|
| Always produces **unary** functions (one arg at a time) | Can fix **any number** of args at once |
| `f(a)(b)(c)` | `f(a, b)(c)` or `f(a)(b, c)` |
| All calls return a function until all args received | Returns function or result depending on args count |
| Strict mathematical definition | More flexible, practical |

The `curry` implementation above supports **both** — it accepts multiple args at each step, making it a hybrid.

---

## Summary — Polyfill Complexity Tiers

| Tier | Polyfills | Key Concepts Tested |
|---|---|---|
| **Tier 1 (Warm-up)** | `forEach`, `find`, `findIndex`, `some`, `every`, `includes`, `Object.keys/values/entries` | Basic iteration, `this` piping, edge cases |
| **Tier 2 (Expected)** | `map`, `filter`, `reduce`, `flat`, `flatMap`, `Array.from` | Sparse arrays, return value, recursive flattening |
| **Tier 3 (Impressive)** | `bind`, `call`, `apply`, `Object.assign`, `Object.create`, `Object.freeze` | `this` binding, Symbol keys, prototype chain, new operator |
| **Tier 4 (Senior)** | `Promise`, `Promise.all/allSettled/race/any` | Microtasks, state machine, error propagation |
| **Tier 5 (Staff+)** | `debounce`, `throttle`, `deepClone`, `curry` | Closures, timers, circular refs, function composition |

---

*Last updated: February 2026 | ES2024*
