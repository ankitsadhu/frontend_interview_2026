# Compound Components — The Basic Pattern (`cloneElement`)

## Overview

The **first-generation** approach to compound components uses `React.Children` and `React.cloneElement` to inject props (including shared state) into child components.

> ⚠️ **Note:** This pattern is largely **superseded by Context** (covered in file 03), but you must understand it because:
> 1. Many open-source libraries still use it
> 2. Interviewers ask about its trade-offs
> 3. It's simpler for very small compound components

---

## Example: Building a Simple Accordion

### Step 1: Define the Parent (State Owner)

```jsx
import React, { useState } from 'react';

function Accordion({ children }) {
  const [openIndex, setOpenIndex] = useState(null);

  const toggle = (index) => {
    setOpenIndex(prev => (prev === index ? null : index));
  };

  // Clone each child and inject state + handlers
  return (
    <div className="accordion">
      {React.Children.map(children, (child, index) => {
        if (!React.isValidElement(child)) return child;

        return React.cloneElement(child, {
          isOpen: openIndex === index,
          onToggle: () => toggle(index),
        });
      })}
    </div>
  );
}
```

### Step 2: Define the Child (State Consumer)

```jsx
function AccordionItem({ title, children, isOpen, onToggle }) {
  return (
    <div className="accordion-item">
      <button
        className="accordion-header"
        onClick={onToggle}
        aria-expanded={isOpen}
      >
        {title}
        <span className="accordion-icon">{isOpen ? '▲' : '▼'}</span>
      </button>
      {isOpen && (
        <div className="accordion-body">
          {children}
        </div>
      )}
    </div>
  );
}
```

### Step 3: Usage (What the Consumer Writes)

```jsx
<Accordion>
  <AccordionItem title="What is React?">
    React is a JavaScript library for building user interfaces.
  </AccordionItem>
  <AccordionItem title="What are hooks?">
    Hooks are functions that let you use state and lifecycle features
    in function components.
  </AccordionItem>
  <AccordionItem title="What is JSX?">
    JSX is a syntax extension that looks like HTML but produces
    React elements.
  </AccordionItem>
</Accordion>
```

**Notice:** The consumer **never** passes `isOpen` or `onToggle` — the parent injects them via `cloneElement`.

---

## How `cloneElement` Works — Deep Dive

```jsx
React.cloneElement(element, props, ...children)
```

It creates a **new React element** using the original element as a starting point:
- Keeps the original `type` and `key`
- **Shallow merges** the new props with the original props
- New `ref` replaces old `ref` (if provided)

### What Happens Under the Hood

```jsx
// Original element
<AccordionItem title="What is React?">...</AccordionItem>

// After cloneElement
<AccordionItem
  title="What is React?"   // original prop preserved
  isOpen={false}            // injected by parent
  onToggle={() => toggle(0)} // injected by parent
>
  ...
</AccordionItem>
```

---

## The `React.Children` API

| Method | Description |
|---|---|
| `React.Children.map(children, fn)` | Map over children (handles non-array cases) |
| `React.Children.forEach(children, fn)` | Iterate without returning |
| `React.Children.count(children)` | Count children |
| `React.Children.only(children)` | Assert exactly one child |
| `React.Children.toArray(children)` | Convert to flat array with stable keys |

### Why Not Just `children.map()`?

```jsx
// ❌ Breaks if children is a single element (not an array)
children.map(child => ...)

// ✅ Handles single element, arrays, null, undefined
React.Children.map(children, child => ...)
```

---

## Adding Static Properties (Dot Notation)

To enable `<Accordion.Item>` syntax:

```jsx
function Accordion({ children }) {
  const [openIndex, setOpenIndex] = useState(null);
  const toggle = (index) => setOpenIndex(p => p === index ? null : index);

  return (
    <div className="accordion">
      {React.Children.map(children, (child, index) => {
        if (!React.isValidElement(child)) return child; // 👈 skip strings/nulls safely
        return React.cloneElement(child, {
          isOpen: openIndex === index,
          onToggle: () => toggle(index),
        });
      })}
    </div>
  );
}

function AccordionItem({ title, children, isOpen, onToggle }) {
  return (
    <div className="accordion-item">
      <button onClick={onToggle} aria-expanded={isOpen}>
        {title} {isOpen ? '▲' : '▼'}
      </button>
      {isOpen && <div className="accordion-body">{children}</div>}
    </div>
  );
}

// ✅ Attach as a static property
Accordion.Item = AccordionItem;

export default Accordion;
```

Now consumers can write:

```jsx
<Accordion>
  <Accordion.Item title="Question 1">Answer 1</Accordion.Item>
  <Accordion.Item title="Question 2">Answer 2</Accordion.Item>
</Accordion>
```

---

## Limitations of the `cloneElement` Pattern

### 1. Only Direct Children

```jsx
// ❌ This BREAKS — wrapped child won't get injected props
<Accordion>
  <div className="wrapper">
    <AccordionItem title="Oops">Won't work!</AccordionItem>
  </div>
</Accordion>
```

`cloneElement` only injects into **direct** children. Any wrapper element breaks the chain.

### 2. TypeScript Friction

The child component receives `isOpen` and `onToggle` as props, but the **consumer doesn't pass them**. This confuses TypeScript:

```tsx
// TypeScript complains: isOpen and onToggle are required but not provided!
<Accordion>
  <AccordionItem title="Q1">A1</AccordionItem>
</Accordion>
```

**Workaround:** Make injected props optional:

```tsx
interface AccordionItemProps {
  title: string;
  children: React.ReactNode;
  isOpen?: boolean;    // optional because parent injects it
  onToggle?: () => void; // optional because parent injects it
}
```

### 3. Implicit Contract

There's no explicit link between parent and child — it's all based on convention. If someone passes a non-AccordionItem child, it silently gets extra props it doesn't understand.

### 4. Testing Complexity

Children must be tested **within the parent** because they rely on injected props. You can't easily test `AccordionItem` in isolation.

---

## When to Use `cloneElement` Pattern

| Use Case | Recommended? |
|---|---|
| Very simple 2-level parent-child | ✅ Yes |
| Production library | ❌ No — use Context |
| Deeply nested children | ❌ No — use Context |
| TypeScript codebase | ⚠️ Possible but awkward |
| Quick prototype / interview | ✅ Yes — fast to implement |

---

## Complete Working Example

```jsx
import React, { useState } from 'react';

// ───── Parent Component ─────
function Toggle({ children }) {
  const [on, setOn] = useState(false);
  const toggle = () => setOn(prev => !prev);

  return React.Children.map(children, child => {
    if (!React.isValidElement(child)) return child;
    return React.cloneElement(child, { on, toggle });
  });
}

// ───── Child Components ─────
function ToggleOn({ on, children }) {
  return on ? children : null;
}

function ToggleOff({ on, children }) {
  return on ? null : children;
}

function ToggleButton({ on, toggle }) {
  return (
    <button onClick={toggle}>
      {on ? 'ON' : 'OFF'}
    </button>
  );
}

// ───── Attach as static properties ─────
Toggle.On = ToggleOn;
Toggle.Off = ToggleOff;
Toggle.Button = ToggleButton;

export default Toggle;

// ───── Usage ─────
function App() {
  return (
    <Toggle>
      <Toggle.On>The toggle is ON 🟢</Toggle.On>
      <Toggle.Off>The toggle is OFF 🔴</Toggle.Off>
      <Toggle.Button />
    </Toggle>
  );
}
```

---

## Key Takeaways

1. **`cloneElement`** injects props into direct children — simple but limited
2. **`React.Children.map`** handles edge cases that `children.map()` doesn't
3. **Static properties** (`Component.SubComponent`) give clean dot-notation APIs
4. This pattern **breaks with wrapper elements** and has **TypeScript friction**
5. For anything production-grade, prefer the **Context pattern** (next file)
