# Compound Components — The Context Pattern (Modern Approach)

## Why Context?

The `cloneElement` pattern breaks when children are nested inside wrapper elements. The **Context API** solves this by broadcasting state to **any descendant**, regardless of depth.

```
┌─ Parent (Provider) ────────────────────┐
│                                        │
│  ┌─ Wrapper Div ───────────────────┐   │
│  │  ┌─ Child A ─┐  ┌─ Child B ─┐  │   │
│  │  │ useCtx ✅ │  │ useCtx ✅ │  │   │
│  │  └───────────┘  └───────────┘  │   │
│  └─────────────────────────────────┘   │
│                                        │
│  ┌─ Deep Nesting ──────────────────┐   │
│  │  ┌─ Div ──────────────────┐     │   │
│  │  │  ┌─ Child C ─────┐    │     │   │
│  │  │  │ useCtx ✅      │    │     │   │
│  │  │  └───────────────┘    │     │   │
│  │  └────────────────────────┘     │   │
│  └─────────────────────────────────┘   │
└────────────────────────────────────────┘
```

---

## The 3-Step Recipe

Every Context-based compound component follows the same structure:

### Step 1: Create Context + Custom Hook

```jsx
import { createContext, useContext, useState } from 'react';

// 1. Create a Context
const AccordionContext = createContext(null);

// 2. Custom hook with guard clause
function useAccordion() {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error(
      'Accordion compound components must be used within <Accordion>'
    );
  }
  return context;
}
```

> **Why the guard clause?** If someone uses `<Accordion.Item>` outside of `<Accordion>`, they get a **clear error message** instead of a cryptic `Cannot read property 'isOpen' of null`.

### Step 2: Create the Parent (Provider)

```jsx
function Accordion({ children, allowMultiple = false }) {
  const [openIndices, setOpenIndices] = useState(new Set());

  const toggle = (index) => {
    setOpenIndices(prev => {
      const next = new Set(prev);
      if (next.has(index)) {
        next.delete(index);
      } else {
        if (!allowMultiple) next.clear();
        next.add(index);
      }
      return next;
    });
  };

  const isOpen = (index) => openIndices.has(index);

  return (
    <AccordionContext.Provider value={{ isOpen, toggle }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}
```

### Step 3: Create the Children (Consumers)

```jsx
function AccordionItem({ index, title, children }) {
  const { isOpen, toggle } = useAccordion();
  const open = isOpen(index);

  return (
    <div className="accordion-item">
      <button
        className="accordion-header"
        onClick={() => toggle(index)}
        aria-expanded={open}
      >
        {title}
        <span>{open ? '▲' : '▼'}</span>
      </button>
      {open && (
        <div className="accordion-body">{children}</div>
      )}
    </div>
  );
}
```

### Attach + Export

```jsx
Accordion.Item = AccordionItem;
export default Accordion;
```

### Usage

```jsx
// Works even with wrapper elements! ✅
<Accordion allowMultiple>
  <div className="section">
    <h2>FAQ Section</h2>
    <Accordion.Item index={0} title="What is React?">
      A JavaScript library for building UIs.
    </Accordion.Item>
  </div>
  <Accordion.Item index={1} title="What is JSX?">
    A syntax extension for JavaScript.
  </Accordion.Item>
</Accordion>
```

---

## Avoiding the `index` Prop — Automatic Indexing

Manually passing `index` is error-prone. Here's a cleaner approach using a counter ref:

```jsx
import { createContext, useContext, useState, useRef, useMemo } from 'react';

const AccordionContext = createContext(null);

function useAccordion() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('Must be used within <Accordion>');
  return ctx;
}

function Accordion({ children, allowMultiple = false }) {
  const [openIndices, setOpenIndices] = useState(new Set());
  const counterRef = useRef(0);

  // Reset counter on each render so children get consistent indices
  counterRef.current = 0;

  const getNextIndex = () => counterRef.current++;

  const toggle = (index) => {
    setOpenIndices(prev => {
      const next = new Set(prev);
      if (next.has(index)) {
        next.delete(index);
      } else {
        if (!allowMultiple) next.clear();
        next.add(index);
      }
      return next;
    });
  };

  const isOpen = (index) => openIndices.has(index);

  return (
    <AccordionContext.Provider value={{ isOpen, toggle, getNextIndex }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ title, children }) {
  const { isOpen, toggle, getNextIndex } = useAccordion();

  // useRef to get a stable index across re-renders
  const indexRef = useRef(null);
  if (indexRef.current === null) {
    indexRef.current = getNextIndex();
  }
  const index = indexRef.current;

  const open = isOpen(index);

  return (
    <div className="accordion-item">
      <button onClick={() => toggle(index)} aria-expanded={open}>
        {title} {open ? '▲' : '▼'}
      </button>
      {open && <div className="accordion-body">{children}</div>}
    </div>
  );
}

Accordion.Item = AccordionItem;
export default Accordion;
```

Now consumers don't need `index`:

```jsx
<Accordion>
  <Accordion.Item title="Q1">Answer 1</Accordion.Item>
  <Accordion.Item title="Q2">Answer 2</Accordion.Item>
</Accordion>
```

---

## Performance: Preventing Unnecessary Re-renders

### The Problem

Every time `openIndices` changes, **every** `AccordionItem` re-renders — even the ones that didn't change.

### Solution 1: `useMemo` the Context Value

```jsx
function Accordion({ children, allowMultiple = false }) {
  const [openIndices, setOpenIndices] = useState(new Set());

  const toggle = useCallback((index) => {
    setOpenIndices(prev => {
      const next = new Set(prev);
      if (next.has(index)) next.delete(index);
      else {
        if (!allowMultiple) next.clear();
        next.add(index);
      }
      return next;
    });
  }, [allowMultiple]);

  const isOpen = useCallback(
    (index) => openIndices.has(index),
    [openIndices]
  );

  // ✅ Memoize the context value
  const value = useMemo(
    () => ({ isOpen, toggle }),
    [isOpen, toggle]
  );

  return (
    <AccordionContext.Provider value={value}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}
```

### Solution 2: `React.memo` on Children

```jsx
const AccordionItem = React.memo(function AccordionItem({ title, children }) {
  const { isOpen, toggle } = useAccordion();
  // ...
});
```

> ⚠️ `React.memo` alone won't help here because the context value changes reference on each render. Both solutions should be used **together**.

### Solution 3: Split Context (Advanced)

Separate state from dispatch to minimize re-renders:

```jsx
const AccordionStateContext = createContext(null);
const AccordionDispatchContext = createContext(null);

function Accordion({ children }) {
  const [openIndices, setOpenIndices] = useState(new Set());

  const toggle = useCallback((index) => {
    setOpenIndices(prev => {
      const next = new Set(prev);
      if (next.has(index)) next.delete(index);
      else next.add(index);
      return next;
    });
  }, []);

  return (
    <AccordionDispatchContext.Provider value={toggle}>
      <AccordionStateContext.Provider value={openIndices}>
        {children}
      </AccordionStateContext.Provider>
    </AccordionDispatchContext.Provider>
  );
}

// Components that only DISPATCH don't re-render on state change
function AccordionHeader({ index, title }) {
  const toggle = useContext(AccordionDispatchContext);
  return <button onClick={() => toggle(index)}>{title}</button>;
}

// Components that READ state will re-render
function AccordionBody({ index, children }) {
  const openIndices = useContext(AccordionStateContext);
  return openIndices.has(index) ? <div>{children}</div> : null;
}
```

---

## TypeScript Integration

Context-based compound components work **beautifully** with TypeScript:

```tsx
// ───── Types ─────
interface AccordionContextValue {
  isOpen: (index: number) => boolean;
  toggle: (index: number) => void;
}

interface AccordionProps {
  children: React.ReactNode;
  allowMultiple?: boolean;
  defaultOpen?: number[];
}

interface AccordionItemProps {
  index: number;
  title: string;
  children: React.ReactNode;
}

// ───── Context ─────
const AccordionContext = createContext<AccordionContextValue | null>(null);

function useAccordion(): AccordionContextValue {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('Must be used within <Accordion>');
  }
  return context;
}

// ───── Components ─────
function Accordion({ children, allowMultiple = false, defaultOpen = [] }: AccordionProps) {
  const [openIndices, setOpenIndices] = useState<Set<number>>(
    new Set(defaultOpen)
  );
  // ... same logic
}

function AccordionItem({ index, title, children }: AccordionItemProps) {
  const { isOpen, toggle } = useAccordion();
  // ... same logic
}
```

No more optional props or ambiguity — TypeScript knows exactly what each component expects.

---

## cloneElement vs Context: Full Comparison

| Feature | `cloneElement` | Context |
|---|---|---|
| Deep nesting | ❌ Breaks | ✅ Works |
| TypeScript | ⚠️ Awkward | ✅ Clean |
| Performance | ✅ No extra context | ⚠️ Needs optimization |
| Flexibility | ❌ Direct children only | ✅ Any descendant |
| Complexity | ✅ Simple | ⚠️ More boilerplate |
| Testing | ❌ Must test with parent | ✅ Can mock context |
| Production use | ❌ Not recommended | ✅ Standard approach |

---

## Key Takeaways

1. **Context is the modern standard** for compound components
2. **Always create a custom hook** with a guard clause for clear error messages
3. **Memoize context values** with `useMemo` to prevent unnecessary re-renders
4. **Split contexts** (state vs dispatch) for heavy components
5. **TypeScript works naturally** with the context pattern — no optional prop hacks
6. The **3-step recipe** (Create Context → Provider Parent → Consumer Children) applies to every compound component
