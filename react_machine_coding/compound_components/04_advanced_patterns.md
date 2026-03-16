# Compound Components — Advanced Patterns

## 1. Controlled vs Uncontrolled Compound Components

Just like form inputs, compound components can be **controlled** (state managed by the consumer) or **uncontrolled** (state managed internally).

### Uncontrolled (Internal State)

```jsx
// The component manages its own state — consumer has no control
<Accordion>
  <Accordion.Item title="Q1">A1</Accordion.Item>
  <Accordion.Item title="Q2">A2</Accordion.Item>
</Accordion>
```

### Controlled (External State)

```jsx
// Consumer manages state — full control
const [openItems, setOpenItems] = useState([0]);

<Accordion
  openItems={openItems}
  onToggle={(index) => {
    setOpenItems(prev =>
      prev.includes(index)
        ? prev.filter(i => i !== index)
        : [...prev, index]
    );
  }}
>
  <Accordion.Item title="Q1">A1</Accordion.Item>
  <Accordion.Item title="Q2">A2</Accordion.Item>
</Accordion>
```

### The "Controllable" Pattern — Supporting Both

This is the **gold standard** used by production libraries:

```jsx
import { createContext, useContext, useState, useCallback, useMemo } from 'react';

const AccordionContext = createContext(null);

function useAccordion() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('Must be used within <Accordion>');
  return ctx;
}

function Accordion({
  children,
  // Controlled props
  openItems: controlledOpenItems,
  onToggle: controlledOnToggle,
  // Uncontrolled props
  defaultOpenItems = [],
  allowMultiple = false,
}) {
  // Internal state (used only when uncontrolled)
  const [internalOpenItems, setInternalOpenItems] = useState(
    new Set(defaultOpenItems)
  );

  // Determine if controlled
  const isControlled = controlledOpenItems !== undefined;

  // Use controlled or internal state
  const openItems = isControlled
    ? new Set(controlledOpenItems)
    : internalOpenItems;

  const toggle = useCallback((index) => {
    if (isControlled) {
      // In controlled mode, just call the handler — let consumer manage state
      controlledOnToggle?.(index);
    } else {
      // In uncontrolled mode, manage state internally
      setInternalOpenItems(prev => {
        const next = new Set(prev);
        if (next.has(index)) {
          next.delete(index);
        } else {
          if (!allowMultiple) next.clear();
          next.add(index);
        }
        return next;
      });
    }
  }, [isControlled, controlledOnToggle, allowMultiple]);

  const isOpen = useCallback(
    (index) => openItems.has(index),
    [openItems]
  );

  const value = useMemo(() => ({ isOpen, toggle }), [isOpen, toggle]);

  return (
    <AccordionContext.Provider value={value}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}
```

**Usage — both modes work:**

```jsx
// Uncontrolled (simple use)
<Accordion defaultOpenItems={[0]}>
  <Accordion.Item index={0} title="Q1">A1</Accordion.Item>
</Accordion>

// Controlled (full control)
<Accordion openItems={openItems} onToggle={handleToggle}>
  <Accordion.Item index={0} title="Q1">A1</Accordion.Item>
</Accordion>
```

---

## 2. State Reducers Pattern

Give consumers the power to **intercept and modify** state changes:

```jsx
function Accordion({
  children,
  stateReducer = (state, changes) => changes, // identity by default
  allowMultiple = false,
}) {
  const [openIndices, dispatch] = useReducer(
    (state, action) => {
      const changes = accordionReducer(state, action, allowMultiple);
      // Let the consumer modify the changes
      return stateReducer(state, changes);
    },
    new Set()
  );

  const toggle = (index) => dispatch({ type: 'TOGGLE', index });
  const isOpen = (index) => openIndices.has(index);

  return (
    <AccordionContext.Provider value={{ isOpen, toggle }}>
      {children}
    </AccordionContext.Provider>
  );
}

function accordionReducer(state, action, allowMultiple) {
  switch (action.type) {
    case 'TOGGLE': {
      const next = new Set(state);
      if (next.has(action.index)) {
        next.delete(action.index);
      } else {
        if (!allowMultiple) next.clear();
        next.add(action.index);
      }
      return next;
    }
    default:
      return state;
  }
}
```

**Consumer can modify behavior:**

```jsx
// Prevent closing the last open item
<Accordion
  stateReducer={(currentState, proposedChanges) => {
    if (proposedChanges.size === 0) {
      return currentState; // Don't allow empty state
    }
    return proposedChanges;
  }}
>
  ...
</Accordion>
```

---

## 3. Prop Getters Pattern

Instead of returning raw values, return **getter functions** that merge consumer props with internal props:

```jsx
function useAccordionItem(index) {
  const { isOpen, toggle } = useAccordion();
  const open = isOpen(index);

  // Prop getter — merges internal props with consumer's props
  const getToggleProps = (props = {}) => ({
    'aria-expanded': open,
    role: 'button',
    tabIndex: 0,
    ...props,
    onClick: callAll(props.onClick, () => toggle(index)),
    onKeyDown: callAll(props.onKeyDown, (e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        e.preventDefault();
        toggle(index);
      }
    }),
  });

  const getContentProps = (props = {}) => ({
    role: 'region',
    hidden: !open,
    ...props,
  });

  return { open, getToggleProps, getContentProps };
}

// Utility to call multiple event handlers
function callAll(...fns) {
  return (...args) => fns.forEach(fn => fn?.(...args));
}
```

**Usage with prop getters:**

```jsx
function CustomAccordionItem({ index, title, children }) {
  const { open, getToggleProps, getContentProps } = useAccordionItem(index);

  return (
    <div>
      {/* Consumer's onClick runs alongside internal onClick */}
      <button
        {...getToggleProps({
          onClick: () => console.log('Custom click handler!'),
          className: 'my-custom-class',
        })}
      >
        {title}
      </button>
      <div {...getContentProps()}>
        {open && children}
      </div>
    </div>
  );
}
```

**Why prop getters?**
- Ensures **accessibility props** are always applied
- Allows consumer to **add** event handlers without overriding internal ones
- The `callAll` utility runs both internal and consumer handlers

---

## 4. Flexible Compound Components (Mixed Approach)

Combine Context with `cloneElement` for the best of both worlds:

```jsx
function Accordion({ children, allowMultiple = false }) {
  const [openIndices, setOpenIndices] = useState(new Set());
  // ... state logic

  const value = useMemo(
    () => ({ isOpen, toggle }),
    [isOpen, toggle]
  );

  // Provide context for deeply nested children
  // Clone direct children for automatic index assignment
  return (
    <AccordionContext.Provider value={value}>
      <div className="accordion">
        {React.Children.map(children, (child, index) => {
          if (React.isValidElement(child) && child.type === AccordionItem) {
            return React.cloneElement(child, { index });
          }
          return child;
        })}
      </div>
    </AccordionContext.Provider>
  );
}
```

---

## 5. Component Validation

Ensure only valid children are used:

```jsx
function Accordion({ children }) {
  // In development, validate children
  if (process.env.NODE_ENV !== 'production') {
    React.Children.forEach(children, (child) => {
      if (React.isValidElement(child)) {
        const validTypes = [AccordionItem, AccordionHeader, AccordionBody];
        if (!validTypes.includes(child.type)) {
          console.warn(
            `Accordion: Unexpected child type "${
              child.type?.displayName || child.type?.name || child.type
            }". Expected one of: AccordionItem, AccordionHeader, AccordionBody.`
          );
        }
      }
    });
  }

  // ... rest of component
}
```

---

## 6. `displayName` for Debugging

Always set `displayName` for components used in compound patterns:

```jsx
const AccordionItem = React.memo(function AccordionItem(props) {
  // ...
});

AccordionItem.displayName = 'Accordion.Item';

// Now React DevTools shows "Accordion.Item" instead of "AccordionItem"
```

---

## Pattern Summary

| Pattern | Purpose | Complexity |
|---|---|---|
| **Controlled/Uncontrolled** | Support both internal and external state | ⚠️ Medium |
| **State Reducer** | Let consumers intercept state changes | ❌ High |
| **Prop Getters** | Merge accessibility + consumer props | ⚠️ Medium |
| **Flexible Compound** | Context + cloneElement for auto-indexing | ⚠️ Medium |
| **Component Validation** | Dev-time safety checks | ✅ Low |
| **displayName** | Better DevTools experience | ✅ Low |

---

## When to Reach for Advanced Patterns

```
Start simple
   │
   ▼
Context-based compound component (file 03)
   │
   ├── Need external state control? ──▶ Controlled/Uncontrolled
   │
   ├── Need consumers to modify behavior? ──▶ State Reducer
   │
   ├── Need guaranteed accessibility? ──▶ Prop Getters
   │
   └── Need auto-indexing? ──▶ Flexible Compound (Context + cloneElement)
```

> **Rule of thumb:** Start with the basic Context pattern. Add advanced patterns only when you hit a real limitation — not because they look cool in an article.
