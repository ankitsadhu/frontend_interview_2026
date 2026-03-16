# Compound Components — Introduction

## What Are Compound Components?

Compound components are a **React pattern** where multiple components work together to form a cohesive UI unit, sharing **implicit state** without the consumer needing to wire things up manually.

Think of it like native HTML:

```html
<select>
  <option value="a">Apple</option>
  <option value="b">Banana</option>
</select>
```

`<select>` and `<option>` are **compound components** — they share state (which option is selected) implicitly. You never pass `onChange` to each `<option>` yourself. The parent manages the state; the children consume it.

The same idea in React:

```jsx
<Tabs>
  <Tabs.List>
    <Tabs.Tab>Profile</Tabs.Tab>
    <Tabs.Tab>Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel>Profile content</Tabs.Panel>
  <Tabs.Panel>Settings content</Tabs.Panel>
</Tabs>
```

---

## Why Do We Need This Pattern?

### The Problem: Prop Drilling + Rigid APIs

Without compound components, a `Tabs` component might look like this:

```jsx
<Tabs
  tabs={[
    { label: "Profile", content: <ProfileContent /> },
    { label: "Settings", content: <SettingsContent /> },
  ]}
  activeTab={0}
  onChange={setActiveTab}
  tabStyle={{ color: "blue" }}
  panelStyle={{ padding: "1rem" }}
/>
```

**Problems with this approach:**

1. **Rigid API** — Adding a new feature (icons, badges, disabled tabs) means more props
2. **No layout control** — Consumer can't control the DOM structure
3. **Prop explosion** — `tabStyle`, `panelStyle`, `tabClassName`, `activeTabClassName`...
4. **Hard to extend** — Every customization requires the component author to anticipate it

### The Solution: Compound Components Give You Flexibility

```jsx
<Tabs defaultIndex={0}>
  <Tabs.List>
    <Tabs.Tab>
      <Icon name="user" /> Profile
    </Tabs.Tab>
    <Tabs.Tab disabled>Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel>
    <ProfileContent />
  </Tabs.Panel>
  <Tabs.Panel>
    <p>Settings are disabled</p>
  </Tabs.Panel>
</Tabs>
```

**Wins:**

- ✅ Consumer controls the layout and markup
- ✅ Easy to add icons, badges, custom content
- ✅ Minimal API surface — each sub-component handles its own concern
- ✅ Follows the **Inversion of Control** principle

---

## Mental Model

```
┌──────────────────────────────────┐
│         Parent Component         │
│   (owns state + provides it      │
│    via Context)                   │
│                                  │
│  ┌────────┐  ┌────────┐         │
│  │ Child  │  │ Child  │  ...    │
│  │  A     │  │  B     │         │
│  └────────┘  └────────┘         │
│  (consume state from Context)    │
└──────────────────────────────────┘
```

- **Parent** = State owner + Context Provider
- **Children** = Context Consumers that read/modify shared state
- **Consumer** = The developer using your component (controls structure)

---

## Compound Components vs Alternatives

| Approach                        | Flexibility  | Complexity | Best For                        |
| ------------------------------- | ------------ | ---------- | ------------------------------- |
| **Single Monolithic Component** | ❌ Low       | ✅ Low     | Simple, unchanging UI           |
| **Render Props**                | ✅ High      | ⚠️ Medium  | Dynamic rendering logic         |
| **Compound Components**         | ✅ High      | ⚠️ Medium  | Multi-part UI with shared state |
| **Headless Components**         | ✅ Very High | ❌ High    | Full styling/structure control  |

### When to Choose Compound Components

- You're building a **multi-part UI** (tabs, accordion, dropdown, menu)
- Sub-parts need to **share implicit state**
- You want consumers to **control the layout/structure**
- You want a **declarative, readable API**

### When NOT to Use

- The component is simple (a button, an input)
- State is not shared between parts
- You're over-engineering for a one-off component

---

## Core React Concepts You Need

Before diving deeper, make sure you understand:

1. **`React.createContext` + `useContext`** — The backbone of implicit state sharing
2. **`React.Children` + `React.cloneElement`** — Used in the older pattern (file 02)
3. **Custom Hooks** — For clean context consumption
4. **Static properties on components** — `Tabs.Tab`, `Tabs.Panel` (dot notation)

---

## Road Map of This Guide

| File                        | Topic                                                                |
| --------------------------- | -------------------------------------------------------------------- |
| `01_introduction.md`        | 📍 You are here                                                      |
| `02_basic_pattern.md`       | Build your first compound component (Accordion) using `cloneElement` |
| `03_context_pattern.md`     | The modern approach using Context API                                |
| `04_advanced_patterns.md`   | Controlled vs Uncontrolled, Flexible Compound Components             |
| `05_real_world_examples.md` | Tabs, Select/Dropdown, Menu — production-grade implementations       |
| `06_interview_questions.md` | MAANG-level interview questions and answers                          |

---

## Key Takeaway

> **Compound Components = Implicit State Sharing + Consumer Layout Control**
>
> The parent owns the state, provides it via Context, and the children consume it — giving the end developer full control over structure while keeping the API minimal and declarative.
