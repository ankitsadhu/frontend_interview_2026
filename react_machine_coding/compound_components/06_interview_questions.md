# Compound Components — Interview Questions

MAANG-level interview questions and answers on compound components.

---

## Level 1: Conceptual Understanding

### Q1: What are compound components? Give a simple analogy.

**Answer:**
Compound components are a React pattern where multiple components work together as a unit, sharing implicit state through React Context (or `cloneElement`).

**Analogy:** Think of `<select>` and `<option>` in HTML. `<select>` manages which option is selected. `<option>` renders itself differently based on that state. You never manually wire them together — the relationship is implicit.

```jsx
// HTML compound components
<select>
  <option>Apple</option>
  <option>Banana</option>
</select>

// React compound components
<Tabs>
  <Tabs.Tab>Profile</Tabs.Tab>
  <Tabs.Tab>Settings</Tabs.Tab>
  <Tabs.Panel>Profile content</Tabs.Panel>
  <Tabs.Panel>Settings content</Tabs.Panel>
</Tabs>
```

---

### Q2: What problem does the compound component pattern solve?

**Answer:**
It solves **prop drilling**, **prop explosion**, and **rigid APIs**.

Without compound components:
```jsx
// ❌ Rigid, monolithic API
<Tabs
  tabs={[{ label: 'Profile', content: <Profile />, icon: <UserIcon />, badge: 3, disabled: false }]}
  tabStyle={...}
  activeTabStyle={...}
  panelStyle={...}
  onTabChange={...}
/>
```

With compound components:
```jsx
// ✅ Flexible, composable API
<Tabs onChange={...}>
  <Tabs.Tab><UserIcon /> Profile <Badge count={3} /></Tabs.Tab>
  <Tabs.Panel><Profile /></Tabs.Panel>
</Tabs>
```

**Key benefits:**
- Consumer controls the markup and layout
- Adding features doesn't require new props on the parent
- Follows Inversion of Control

---

### Q3: Explain the two main implementation approaches and their trade-offs.

**Answer:**

| | `cloneElement` | Context API |
|---|---|---|
| **How** | Clone direct children, inject props | Create Context provider, children consume via `useContext` |
| **Deep nesting** | ❌ Breaks (only works on direct children) | ✅ Works at any depth |
| **TypeScript** | ⚠️ Awkward (injected props must be optional) | ✅ Clean (context type is explicit) |
| **Performance** | ✅ No context overhead | ⚠️ All consumers re-render on context change |
| **Testing** | ❌ Children can't be tested alone | ✅ Can mock the context |
| **When to use** | Quick prototypes, simple 2-level nesting | Production code, any real project |

---

## Level 2: Implementation Questions

### Q4: Implement a compound `Toggle` component using Context. It should have `Toggle.On`, `Toggle.Off`, and `Toggle.Button` sub-components.

**Answer:**

```jsx
import { createContext, useContext, useState, useMemo, useCallback } from 'react';

const ToggleContext = createContext(null);

function useToggle() {
  const ctx = useContext(ToggleContext);
  if (!ctx) throw new Error('Toggle components must be used within <Toggle>');
  return ctx;
}

function Toggle({ children, initialOn = false }) {
  const [on, setOn] = useState(initialOn);
  const toggle = useCallback(() => setOn(prev => !prev), []);
  const value = useMemo(() => ({ on, toggle }), [on, toggle]);

  return (
    <ToggleContext.Provider value={value}>
      {children}
    </ToggleContext.Provider>
  );
}

function ToggleOn({ children }) {
  const { on } = useToggle();
  return on ? <>{children}</> : null;
}

function ToggleOff({ children }) {
  const { on } = useToggle();
  return on ? null : <>{children}</>;
}

function ToggleButton() {
  const { on, toggle } = useToggle();
  return (
    <button onClick={toggle} aria-pressed={on}>
      {on ? 'ON' : 'OFF'}
    </button>
  );
}

Toggle.On = ToggleOn;
Toggle.Off = ToggleOff;
Toggle.Button = ToggleButton;

export default Toggle;
```

**Follow-up point:** The guard clause in `useToggle` is a production best practice — without it, consumers get `Cannot read property 'on' of null`.

---

### Q5: How would you make a compound component support both controlled and uncontrolled modes?

**Answer:**

```jsx
function Toggle({
  children,
  // Controlled
  on: controlledOn,
  onToggle,
  // Uncontrolled
  initialOn = false,
}) {
  const [internalOn, setInternalOn] = useState(initialOn);

  const isControlled = controlledOn !== undefined;
  const on = isControlled ? controlledOn : internalOn;

  const toggle = useCallback(() => {
    if (isControlled) {
      onToggle?.(!controlledOn);
    } else {
      setInternalOn(prev => {
        onToggle?.(!prev); // Optional notif callback in uncontrolled too
        return !prev;
      });
    }
  }, [isControlled, controlledOn, onToggle]);

  const value = useMemo(() => ({ on, toggle }), [on, toggle]);

  return (
    <ToggleContext.Provider value={value}>
      {children}
    </ToggleContext.Provider>
  );
}
```

```jsx
// Uncontrolled — internal state
<Toggle initialOn={false}>
  <Toggle.Button />
</Toggle>

// Controlled — external state
const [isOn, setIsOn] = useState(false);
<Toggle on={isOn} onToggle={setIsOn}>
  <Toggle.Button />
</Toggle>
```

---

### Q6: What is the "prop getters" pattern and why would you use it?

**Answer:**

Prop getters are functions that return a set of props (including event handlers) that consumers can spread onto their own elements. They **merge** consumer-provided props with internal props instead of overriding them.

```jsx
function useToggle() {
  const [on, setOn] = useState(false);

  const toggle = () => setOn(prev => !prev);

  const getTogglerProps = (props = {}) => ({
    'aria-pressed': on,
    role: 'button',
    tabIndex: 0,
    ...props,
    // Merge onClick — run BOTH internal and consumer's handlers
    onClick: (...args) => {
      props.onClick?.(...args);
      toggle();
    },
  });

  return { on, toggle, getTogglerProps };
}
```

**Why?**
- Guarantees accessibility props are always applied
- Consumer's event handlers **compose** with internal handlers (not override)
- Single spread replaces multiple individual props

```jsx
// Consumer adds their own onClick — both handlers run
<button {...getTogglerProps({ onClick: () => analytics.track('toggled') })}>
  Toggle
</button>
```

---

## Level 3: Architecture & Trade-offs

### Q7: How do you prevent unnecessary re-renders in Context-based compound components?

**Answer:** Three strategies:

**1. `useMemo` the context value:**
```jsx
const value = useMemo(() => ({ on, toggle }), [on, toggle]);
return <Context.Provider value={value}>{children}</Context.Provider>;
```

**2. `useCallback` for stable handler references:**
```jsx
const toggle = useCallback(() => setOn(prev => !prev), []);
```

**3. Split Context (state vs dispatch):**
```jsx
const StateContext = createContext(null);    // changes often
const DispatchContext = createContext(null); // stable

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <DispatchContext.Provider value={dispatch}>
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}

// Components that only dispatch actions DON'T re-render on state change
```

---

### Q8: You're building a `Form` compound component. How would you handle validation across `Form.Field` children?

**Answer:**

```jsx
const FormContext = createContext(null);

function Form({ children, onSubmit, validationSchema }) {
  const [values, setValues] = useState({});
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});

  const setValue = useCallback((name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
  }, []);

  const setFieldTouched = useCallback((name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
  }, []);

  const validate = useCallback(() => {
    if (!validationSchema) return true;
    const result = validationSchema.validate(values);
    if (result.errors) {
      setErrors(result.errors);
      return false;
    }
    setErrors({});
    return true;
  }, [values, validationSchema]);

  const handleSubmit = (e) => {
    e.preventDefault();
    if (validate()) onSubmit?.(values);
  };

  const value = useMemo(
    () => ({ values, errors, touched, setValue, setFieldTouched }),
    [values, errors, touched, setValue, setFieldTouched]
  );

  return (
    <FormContext.Provider value={value}>
      <form onSubmit={handleSubmit}>{children}</form>
    </FormContext.Provider>
  );
}

function FormField({ name, label, type = 'text' }) {
  const { values, errors, touched, setValue, setFieldTouched } = useContext(FormContext);

  return (
    <div className="form-field">
      <label>{label}</label>
      <input
        type={type}
        value={values[name] || ''}
        onChange={e => setValue(name, e.target.value)}
        onBlur={() => setFieldTouched(name)}
      />
      {touched[name] && errors[name] && (
        <span className="error">{errors[name]}</span>
      )}
    </div>
  );
}

Form.Field = FormField;
```

**Key insight:** The `Form` parent **centralizes** validation logic. Each `FormField` registers itself via form values and accesses errors from the shared state.

---

### Q9: What's the difference between compound components and render props? When would you choose one over the other?

**Answer:**

| | Compound Components | Render Props |
|---|---|---|
| **API** | Declarative JSX tree | Function-as-children or render prop |
| **Layout control** | Consumer controls structure via nesting | Consumer controls via return value of function |
| **Readability** | ✅ Clean, HTML-like | ⚠️ Can get messy with nesting |
| **Shared state** | Context (implicit) | Function args (explicit) |
| **Use case** | Multi-part UI (Tabs, Accordion, Select) | Single component needing flexible rendering |

**Choose Compound Components:**
```jsx
// Clear, declarative structure
<Tabs>
  <Tabs.List>
    <Tabs.Tab>Profile</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel>Content</Tabs.Panel>
</Tabs>
```

**Choose Render Props:**
```jsx
// When you need access to internal state for completely custom rendering
<Mouse>
  {({ x, y }) => <div>Mouse at {x}, {y}</div>}
</Mouse>
```

---

### Q10: How do open-source libraries implement compound components? Walk through an example.

**Answer:**

**Radix UI** (used by shadcn/ui) is the gold standard:

```jsx
// Radix's Dialog
<Dialog.Root>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay />
    <Dialog.Content>
      <Dialog.Title>Heading</Dialog.Title>
      <Dialog.Description>Body</Dialog.Description>
      <Dialog.Close>X</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

**What Radix does well:**
1. **Headless** — provides behavior + accessibility, no styling
2. **Controlled + Uncontrolled** — `open` / `defaultOpen` props on Root
3. **Portals** — `Dialog.Portal` renders outside the DOM tree
4. **Composable** — every piece is a separate component
5. **Accessible by default** — ARIA attributes, focus trapping, keyboard navigation

**Architecture:**
- Each component family has a `createContext` for its shared state
- Sub-components use custom hooks (e.g., `useDialogContext`) to read state
- `forwardRef` is used on every component for ref forwarding
- `Slot` pattern (like Radix `asChild`) allows rendering as a different element

---

## Level 4: System Design & Coding Round

### Q11: Design a compound `Combobox` component that supports async search, keyboard navigation, and accessibility. Describe the sub-components, state, and Context shape.

**Answer Outline:**

**Sub-components:**
```jsx
<Combobox value={selected} onChange={setSelected}>
  <Combobox.Input placeholder="Search..." />
  <Combobox.Options>
    <Combobox.Option value="react">React</Combobox.Option>
    <Combobox.Option value="vue">Vue</Combobox.Option>
    <Combobox.Empty>No results found</Combobox.Empty>
  </Combobox.Options>
</Combobox>
```

**Context Shape:**
```tsx
interface ComboboxContextValue {
  // State
  isOpen: boolean;
  query: string;
  selectedValue: string | null;
  highlightedIndex: number;

  // Actions
  setQuery: (q: string) => void;
  select: (value: string) => void;
  open: () => void;
  close: () => void;
  setHighlightedIndex: (i: number) => void;

  // Refs
  inputRef: React.RefObject<HTMLInputElement>;
  listRef: React.RefObject<HTMLUListElement>;

  // Registration
  registerOption: (value: string) => void;
  unregisterOption: (value: string) => void;
}
```

**Key design decisions:**
1. Options register themselves on mount — enables dynamic option lists
2. Keyboard navigation updates `highlightedIndex` — managed centrally
3. `inputRef` and `listRef` enable focus management
4. ARIA: `role="combobox"`, `aria-autocomplete`, `aria-activedescendant`
5. For async: the consumer fetches data, filters, and renders `Combobox.Option` — the compound component doesn't manage data fetching

---

### Q12: In a live coding round, you're asked to build a compound `Stepper` component for a multi-step form. Walk through your approach in 10 minutes.

**Answer — Step-by-step approach:**

**Minute 0–2: Define the API**
```jsx
<Stepper initialStep={0}>
  <Stepper.Progress />
  <Stepper.Step title="Account">
    <AccountForm />
  </Stepper.Step>
  <Stepper.Step title="Profile">
    <ProfileForm />
  </Stepper.Step>
  <Stepper.Step title="Confirm">
    <Confirmation />
  </Stepper.Step>
  <Stepper.Controls />
</Stepper>
```

**Minute 2–4: Context**
```jsx
const StepperContext = createContext(null);

function useStepper() {
  const ctx = useContext(StepperContext);
  if (!ctx) throw new Error('Must be inside <Stepper>');
  return ctx;
}
```

**Minute 4–6: Root + Step**
```jsx
function Stepper({ children, initialStep = 0 }) {
  const [currentStep, setCurrentStep] = useState(initialStep);
  const totalSteps = React.Children.toArray(children)
    .filter(c => c.type === StepComponent).length;

  const next = () => setCurrentStep(p => Math.min(p + 1, totalSteps - 1));
  const prev = () => setCurrentStep(p => Math.max(p - 1, 0));
  const goTo = (i) => setCurrentStep(i);

  const value = useMemo(
    () => ({ currentStep, totalSteps, next, prev, goTo }),
    [currentStep, totalSteps]
  );

  return (
    <StepperContext.Provider value={value}>
      <div className="stepper">{children}</div>
    </StepperContext.Provider>
  );
}

function StepComponent({ title, children, index }) {
  const { currentStep } = useStepper();
  if (currentStep !== index) return null;
  return <div className="stepper-step">{children}</div>;
}
```

**Minute 6–8: Progress + Controls**
```jsx
function Progress() {
  const { currentStep, totalSteps } = useStepper();
  return (
    <div className="stepper-progress">
      {Array.from({ length: totalSteps }, (_, i) => (
        <div
          key={i}
          className={`step-dot ${i <= currentStep ? 'active' : ''}`}
        />
      ))}
    </div>
  );
}

function Controls() {
  const { currentStep, totalSteps, next, prev } = useStepper();
  return (
    <div className="stepper-controls">
      <button onClick={prev} disabled={currentStep === 0}>Back</button>
      <button onClick={next}>
        {currentStep === totalSteps - 1 ? 'Submit' : 'Next'}
      </button>
    </div>
  );
}
```

**Minute 8–10: Attach + polish**
```jsx
Stepper.Step = StepComponent;
Stepper.Progress = Progress;
Stepper.Controls = Controls;
```

---

## Quick-Fire Questions

| # | Question | Key Answer |
|---|---|---|
| 1 | Why use `createContext(null)` instead of `createContext(defaultValue)`? | So the guard clause in the custom hook can detect missing Provider |
| 2 | Why attach sub-components as static properties? | Enables dot-notation (`Tabs.Tab`) for clear API + tree-shaking is unaffected |
| 3 | Can you use compound components with `React.lazy`? | Yes, but lazy-loaded sub-components can't be static properties of the parent. Use named exports instead |
| 4 | How do you handle `forwardRef` in compound components? | Wrap each sub-component with `forwardRef` so consumers can access DOM nodes |
| 5 | What's `asChild` in Radix UI? | A pattern using `Slot` — renders the child element instead of a default element, merging props. Enables `<Button asChild><a href="...">Link</a></Button>` |
| 6 | Can you combine compound components with hooks? | Yes — headless UI libraries (Downshift, React Aria) expose hooks that can be used to build compound components |
| 7 | What happens if you nest two instances of the same compound component? | Each has its own Context Provider, so they work independently (Context uses closest Provider) |

---

## Key Takeaway for Interviews

> When asked to build any multi-part UI in a coding round:
> 1. Start by **defining the consumer API** (what the JSX looks like)
> 2. Create **Context + custom hook**
> 3. Build **Root component** (state + Provider)
> 4. Build **sub-components** (consumers)
> 5. Add **accessibility** (ARIA roles, keyboard events)
> 6. Mention **performance** (useMemo, useCallback, split context)
>
> This approach works for Tabs, Accordion, Select, Menu, Dialog, Stepper, and almost any multi-part component.
