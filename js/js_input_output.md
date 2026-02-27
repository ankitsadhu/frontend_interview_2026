# JavaScript Input / Output — MANG Level Interview (40 Questions)

> "What's the output?" questions testing hoisting, closures, `this`, event loop, coercion, prototypes, async, and scope

---

## 1. Hoisting

### Q1
```js
console.log(a);
var a = 10;
```
**Output:** `undefined`

**Why:** `var` declarations are hoisted (moved to top) but not the assignment. Equivalent to:
```js
var a;           // hoisted
console.log(a);  // undefined
a = 10;
```

---

### Q2
```js
console.log(a);
let a = 10;
```
**Output:** `ReferenceError: Cannot access 'a' before initialization`

**Why:** `let` and `const` are hoisted but stay in the **Temporal Dead Zone (TDZ)** — from block start until the declaration line. Accessing them before declaration throws.

---

### Q3
```js
hello();
function hello() { console.log('Hello!'); }
```
**Output:** `Hello!`

**Why:** **Function declarations** are fully hoisted — both name and body. You can call them before their declaration.

---

### Q4
```js
hello();
var hello = function () { console.log('Hello!'); };
```
**Output:** `TypeError: hello is not a function`

**Why:** `var hello` is hoisted as `undefined`. Calling `undefined()` throws TypeError. **Function expressions** are NOT hoisted.

---

### Q5
```js
var a = 1;
function foo() {
  console.log(a);
  var a = 2;
}
foo();
```
**Output:** `undefined`

**Why:** The `var a` inside `foo` is hoisted to the top of `foo`. The local `a` shadows the global one, but its value is `undefined` at the point of `console.log`.

---

## 2. Closures

### Q6
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
```
**Output:**
```
3
3
3
```

**Why:** `var` is function-scoped, not block-scoped. There's ONE `i` shared across all callbacks. By the time the timeouts fire, the loop is done and `i === 3`.

---

### Q7
```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
```
**Output:**
```
0
1
2
```

**Why:** `let` is **block-scoped**. Each iteration creates a new `i` binding. Each closure captures its own copy.

---

### Q8
```js
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 100);
  })(i);
}
```
**Output:**
```
0
1
2
```

**Why:** The **IIFE** creates a new scope for each iteration. `j` is a local copy of `i` at that moment — the classic `var` closure fix before `let` existed.

---

### Q9
```js
function createCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    getCount: () => count,
  };
}

const counter = createCounter();
counter.increment();
counter.increment();
console.log(counter.getCount());
console.log(counter.count);
```
**Output:**
```
2
undefined
```

**Why:** `count` is enclosed in the closure — accessible via `increment`/`getCount` but not directly. `counter.count` doesn't exist on the returned object.

---

### Q10
```js
function makeAdder(x) {
  return function (y) {
    return x + y;
  };
}

const add5 = makeAdder(5);
const add10 = makeAdder(10);
console.log(add5(3));
console.log(add10(3));
console.log(add5 === add10);
```
**Output:**
```
8
13
false
```

**Why:** Each call to `makeAdder` creates a new closure with its own `x`. `add5` and `add10` are different function instances.

---

## 3. `this` Binding

### Q11
```js
const obj = {
  name: 'Alice',
  greet: function () {
    console.log(this.name);
  },
};

obj.greet();
const greet = obj.greet;
greet();
```
**Output:**
```
Alice
undefined
```

**Why:** `obj.greet()` — `this` is `obj`. `greet()` — detached from object, `this` is `undefined` (strict mode) or `window` (sloppy mode).

---

### Q12
```js
const obj = {
  name: 'Alice',
  greet: () => {
    console.log(this.name);
  },
};

obj.greet();
```
**Output:** `undefined`

**Why:** Arrow functions **don't have their own `this`** — they inherit from the enclosing lexical scope. Here that's the module/global scope, where `this.name` is `undefined`.

---

### Q13
```js
function Person(name) {
  this.name = name;
  this.greet = function () {
    console.log(this.name);
  };
}

const alice = new Person('Alice');
const greet = alice.greet;
alice.greet();
greet();
```
**Output:**
```
Alice
undefined
```

**Why:** Same principle — `alice.greet()` binds `this` to `alice`. `greet()` detaches it → `this` is global/undefined.

---

### Q14
```js
const obj = {
  name: 'Alice',
  inner: {
    name: 'Bob',
    greet: function () {
      console.log(this.name);
    },
  },
};

obj.inner.greet();
const greet = obj.inner.greet;
greet();
```
**Output:**
```
Bob
undefined
```

**Why:** `this` is determined by the **immediate caller** — `obj.inner.greet()` → `this` is `obj.inner` (Bob). Detached → global.

---

### Q15
```js
const obj = {
  name: 'Alice',
  greet: function () {
    setTimeout(function () {
      console.log(this.name);
    }, 100);
  },
};

obj.greet();
```
**Output:** `undefined`

**Why:** The callback inside `setTimeout` is called by the browser (not as a method) → `this` is `window`/`undefined`.

**Fix:** Arrow function captures `this` from `greet`:
```js
setTimeout(() => console.log(this.name), 100); // "Alice"
```

---

### Q16
```js
const obj = { a: 1 };
function foo() {
  console.log(this.a);
}

foo.call(obj);
foo.apply(obj);
foo.bind(obj)();
```
**Output:**
```
1
1
1
```

**Why:** `call`, `apply`, and `bind` all explicitly set `this` to `obj`.

---

## 4. Event Loop & Async

### Q17
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
console.log('3');
```
**Output:**
```
1
3
2
```

**Why:** `setTimeout` (even with 0ms) goes to the **macrotask queue**. The synchronous code finishes first (`1`, `3`), then the event loop picks up the timeout (`2`).

---

### Q18
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```
**Output:**
```
1
4
3
2
```

**Why:** Execution order:
1. Synchronous: `1`, `4`
2. **Microtask** queue (Promise): `3`
3. **Macrotask** queue (setTimeout): `2`

Microtasks always run before macrotasks after the call stack empties.

---

### Q19
```js
console.log('start');

setTimeout(() => console.log('timeout'), 0);

Promise.resolve()
  .then(() => console.log('promise 1'))
  .then(() => console.log('promise 2'));

console.log('end');
```
**Output:**
```
start
end
promise 1
promise 2
timeout
```

**Why:** Sync (`start`, `end`) → microtasks (`promise 1`, then `promise 2` — chained) → macrotask (`timeout`).

---

### Q20
```js
async function foo() {
  console.log('A');
  await Promise.resolve();
  console.log('B');
}

console.log('C');
foo();
console.log('D');
```
**Output:**
```
C
A
D
B
```

**Why:**
1. `C` — sync
2. `foo()` starts, prints `A` (sync part before `await`)
3. `await` yields control back — everything after it becomes a microtask
4. `D` — sync continues
5. Call stack empty → microtask runs → `B`

---

### Q21
```js
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}

async function async2() {
  console.log('async2');
}

console.log('script start');
setTimeout(() => console.log('setTimeout'), 0);
async1();
new Promise((resolve) => {
  console.log('promise1');
  resolve();
}).then(() => console.log('promise2'));
console.log('script end');
```
**Output:**
```
script start
async1 start
async2
promise1
script end
async1 end
promise2
setTimeout
```

**Why:**
1. `script start` — sync
2. `async1()` → `async1 start` → calls `async2()` → `async2` — all sync
3. `await` yields — `async1 end` becomes a microtask
4. `new Promise(executor)` — executor is sync → `promise1`, `.then` queues microtask
5. `script end` — sync
6. Microtask queue: `async1 end`, then `promise2`
7. Macrotask queue: `setTimeout`

---

### Q22
```js
setTimeout(() => console.log('1'), 0);
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => {
  console.log('3');
  setTimeout(() => console.log('4'), 0);
  Promise.resolve().then(() => console.log('5'));
});
Promise.resolve().then(() => console.log('6'));
```
**Output:**
```
3
6
5
1
2
4
```

**Why:**
1. Microtask queue processes: `3` (creates timeout for `4`, queues microtask `5`), then `6`
2. Microtask `5` runs (scheduled inside a microtask — runs before macrotasks)
3. Macrotask queue: `1`, `2`, `4` (in order they were scheduled)

---

## 5. Scope & Declarations

### Q23
```js
function foo() {
  var a = b = 5;
}
foo();
console.log(typeof a);
console.log(typeof b);
```
**Output:**
```
undefined
number
```

**Why:** `var a = b = 5` is parsed as `b = 5; var a = b;`. `b` has **no `var`** → becomes a global variable (implicit global). `a` is function-scoped → not accessible outside.

---

### Q24
```js
(function () {
  var a = (b = 3);
})();

console.log(b);
console.log(typeof a);
```
**Output:**
```
3
undefined
```

**Why:** Same as above — `b` is an accidental global. `a` is scoped to the IIFE.

---

### Q25
```js
let x = 1;
{
  console.log(x);
  let x = 2;
}
```
**Output:** `ReferenceError: Cannot access 'x' before initialization`

**Why:** The inner `let x` creates a TDZ in the block. Even though outer `x` exists, the inner `x` shadows it — and accessing it before declaration throws.

---

## 6. Type Coercion

### Q26
```js
console.log(1 + '2');
console.log(1 - '2');
console.log('5' + 3);
console.log('5' - 3);
console.log(true + true);
console.log(true + '1');
console.log([] + []);
console.log([] + {});
console.log({} + []);
```
**Output:**
```
12
-1
53
2
2
true1
""
"[object Object]"
"[object Object]"   (or 0 in some consoles when {} is parsed as block)
```

**Rules:**
- `+` with a string → **concatenation**
- `-` always does **numeric** conversion
- `true` → `1` for numeric operations
- `[]` → `""` (empty string), `{}` → `"[object Object]"`
- `true + '1'` → string concat → `"true1"`

---

### Q27
```js
console.log(0 == false);
console.log(0 === false);
console.log('' == false);
console.log('' === false);
console.log(null == undefined);
console.log(null === undefined);
console.log(NaN == NaN);
console.log(NaN === NaN);
```
**Output:**
```
true
false
true
false
true
false
false
false
```

**Rules:**
- `==` does type coercion: `0`, `""`, `false` are all coerced to `0`
- `===` no coercion: different types → always `false`
- `null == undefined` is **specifically true** (special case in the spec)
- `NaN` is **never equal to anything**, including itself

---

### Q28
```js
console.log(typeof null);
console.log(typeof undefined);
console.log(typeof NaN);
console.log(typeof []);
console.log(typeof {});
console.log(typeof function(){});
```
**Output:**
```
object
undefined
number
object
object
function
```

**Why:** `typeof null === 'object'` is a historical bug from JavaScript's first implementation (the null pointer was represented as an object type internally). Arrays are objects. `NaN` is of type `number`.

---

## 7. Prototypes & Inheritance

### Q29
```js
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function () {
  return `${this.name} makes a sound`;
};

function Dog(name) {
  Animal.call(this, name);
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.speak = function () {
  return `${this.name} barks`;
};

const d = new Dog('Rex');
console.log(d.speak());
console.log(d instanceof Dog);
console.log(d instanceof Animal);
console.log(d.constructor === Dog);
```
**Output:**
```
Rex barks
true
true
true
```

---

### Q30
```js
const obj = { a: 1, b: 2, c: 3 };
const { a, ...rest } = obj;
console.log(a);
console.log(rest);
console.log(obj);
```
**Output:**
```
1
{ b: 2, c: 3 }
{ a: 1, b: 2, c: 3 }
```

**Why:** Destructuring with rest creates a **shallow copy** of remaining properties. Original `obj` is unchanged.

---

## 8. Tricky Edge Cases

### Q31
```js
console.log(0.1 + 0.2 === 0.3);
console.log(0.1 + 0.2);
```
**Output:**
```
false
0.30000000000000004
```

**Why:** IEEE 754 floating-point precision. `0.1` and `0.2` have infinite binary representations that get rounded. Their sum isn't exactly `0.3`.

**Fix:** `Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON`

---

### Q32
```js
console.log([] == ![]);
```
**Output:** `true`

**Why (step by step):**
1. `![]` → `!truthy` → `false`
2. `[] == false`
3. `[] → ""` (ToPrimitive) → `"" == false`
4. `"" → 0`, `false → 0`
5. `0 == 0` → `true`

---

### Q33
```js
const a = {};
const b = { key: 'b' };
const c = { key: 'c' };

a[b] = 123;
a[c] = 456;

console.log(a[b]);
```
**Output:** `456`

**Why:** Object keys must be strings. `b.toString()` → `"[object Object]"`. `c.toString()` → `"[object Object]"`. They produce the **same key** — so `a[c]` overwrites `a[b]`.

---

### Q34
```js
let a = 3;
let b = new Number(3);
let c = 3;

console.log(a == b);
console.log(a === b);
console.log(a === c);
console.log(typeof b);
```
**Output:**
```
true
false
true
object
```

**Why:** `new Number(3)` creates a **Number object** (boxed), not a primitive. `==` coerces → equal. `===` checks type: `number !== object`.

---

### Q35
```js
function foo() {
  return
  {
    bar: 'hello'
  };
}
console.log(foo());
```
**Output:** `undefined`

**Why:** JavaScript's **Automatic Semicolon Insertion (ASI)** adds a semicolon after `return`:
```js
return;         // returns undefined
{ bar: 'hello' } // unreachable (parsed as a labeled block)
```

**Fix:** Open brace on the same line as `return`:
```js
return {
  bar: 'hello'
};
```

---

### Q36
```js
const arr = [1, 2, 3];
arr[10] = 11;
console.log(arr.length);
console.log(arr[5]);
console.log(arr.filter(x => x === undefined).length);
```
**Output:**
```
11
undefined
0
```

**Why:** Setting `arr[10]` creates a **sparse array** with holes at indices 3-9. `arr.length` is `11` but only 4 slots have values. `arr[5]` is `undefined` (hole). `filter` **skips holes** — so the filter finds 0 undefined values.

---

## 9. Generators & Iterators

### Q37
```js
function* gen() {
  yield 1;
  yield 2;
  yield 3;
}

const g = gen();
console.log(g.next());
console.log(g.next());
console.log(g.next());
console.log(g.next());
```
**Output:**
```
{ value: 1, done: false }
{ value: 2, done: false }
{ value: 3, done: false }
{ value: undefined, done: true }
```

---

### Q38
```js
function* counter() {
  let i = 0;
  while (true) {
    const reset = yield i;
    if (reset) i = 0;
    else i++;
  }
}

const c = counter();
console.log(c.next().value);
console.log(c.next().value);
console.log(c.next().value);
console.log(c.next(true).value);
console.log(c.next().value);
```
**Output:**
```
0
1
2
0
1
```

**Why:** `next(value)` sends `value` back into the generator as the result of `yield`. `next(true)` sets `reset = true` → `i = 0`.

---

## 10. Advanced Async

### Q39
```js
const promise = new Promise((resolve, reject) => {
  resolve('first');
  resolve('second');
  reject('error');
});

promise
  .then(val => console.log(val))
  .catch(err => console.log(err));
```
**Output:** `first`

**Why:** A Promise can only be settled **once**. The second `resolve` and the `reject` are silently ignored.

---

### Q40
```js
Promise.resolve(1)
  .then(x => { throw new Error('fail'); })
  .then(x => console.log('A:', x))
  .catch(e => { console.log('B:', e.message); return 42; })
  .then(x => console.log('C:', x))
  .catch(e => console.log('D:', e.message));
```
**Output:**
```
B: fail
C: 42
```

**Why:**
1. First `.then` throws → skips the next `.then` (A), drops into `.catch` (B)
2. `.catch` returns `42` → the chain is **recovered** — it's fulfilled now
3. Next `.then` (C) receives `42`
4. Last `.catch` (D) never runs — chain is fulfilled

**Key insight:** `.catch` that doesn't throw **recovers the chain** — the next `.then` runs normally.

---

## Bonus: Quick-Fire Output Round

### B1
```js
console.log(1 < 2 < 3);
console.log(3 > 2 > 1);
```
**Output:** `true`, `false`

**Why:** `1 < 2` → `true`. `true < 3` → `1 < 3` → `true`.  
`3 > 2` → `true`. `true > 1` → `1 > 1` → `false`.

---

### B2
```js
console.log(+'');
console.log(+' ');
console.log(+true);
console.log(+null);
console.log(+undefined);
console.log(+'hello');
```
**Output:**
```
0
0
1
0
NaN
NaN
```

---

### B3
```js
console.log(typeof typeof 1);
```
**Output:** `"string"`

**Why:** `typeof 1` → `"number"` (a string). `typeof "number"` → `"string"`.

---

### B4
```js
const a = [1, 2, 3];
const b = [1, 2, 3];
const c = a;

console.log(a == b);
console.log(a === b);
console.log(a == c);
```
**Output:**
```
false
false
true
```

**Why:** Arrays are objects — compared **by reference**, not value. `a` and `b` are different objects. `c` points to the same object as `a`.

---

### B5
```js
console.log('start');

const p = new Promise((resolve) => {
  console.log('executor');
  resolve('done');
});

p.then(console.log);

console.log('end');
```
**Output:**
```
start
executor
end
done
```

**Why:** The Promise **executor runs synchronously** (immediately during construction). Only `.then` callbacks are async (microtask queue).

---

## Topic Index

| Topic | Questions |
|---|---|
| **Hoisting** | Q1–Q5 |
| **Closures** | Q6–Q10 |
| **`this` binding** | Q11–Q16 |
| **Event loop & async** | Q17–Q22, Q39–Q40 |
| **Scope & declarations** | Q23–Q25 |
| **Type coercion** | Q26–Q28, B1–B3 |
| **Prototypes** | Q29–Q30 |
| **Edge cases** | Q31–Q36, B4–B5 |
| **Generators** | Q37–Q38 |

---

*Last updated: February 2026 | ES2024*
