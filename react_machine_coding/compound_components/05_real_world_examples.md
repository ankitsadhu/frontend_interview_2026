# Compound Components — Real-World Examples

Production-ready implementations of common UI components using the compound component pattern.

---

## 1. Tabs Component

### Implementation

```jsx
import { createContext, useContext, useState, useCallback, useMemo } from 'react';

// ───── Context ─────
const TabsContext = createContext(null);

function useTabs() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs components must be used within <Tabs>');
  return ctx;
}

// ───── Root Component ─────
function Tabs({ children, defaultIndex = 0, onChange }) {
  const [activeIndex, setActiveIndex] = useState(defaultIndex);

  const selectTab = useCallback((index) => {
    setActiveIndex(index);
    onChange?.(index);
  }, [onChange]);

  const value = useMemo(
    () => ({ activeIndex, selectTab }),
    [activeIndex, selectTab]
  );

  return (
    <TabsContext.Provider value={value}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

// ───── Tab List (Container for Tab buttons) ─────
function TabList({ children, className = '' }) {
  return (
    <div className={`tabs-list ${className}`} role="tablist">
      {children}
    </div>
  );
}

// ───── Individual Tab ─────
function Tab({ children, index, disabled = false }) {
  const { activeIndex, selectTab } = useTabs();
  const isActive = activeIndex === index;

  return (
    <button
      role="tab"
      aria-selected={isActive}
      aria-disabled={disabled}
      tabIndex={isActive ? 0 : -1}
      className={`tab ${isActive ? 'tab--active' : ''} ${disabled ? 'tab--disabled' : ''}`}
      onClick={() => !disabled && selectTab(index)}
      onKeyDown={(e) => {
        if (e.key === 'ArrowRight') {
          e.preventDefault();
          // Focus next tab logic would go here
        }
        if (e.key === 'ArrowLeft') {
          e.preventDefault();
          // Focus prev tab logic would go here
        }
      }}
    >
      {children}
    </button>
  );
}

// ───── Tab Panel ─────
function TabPanel({ children, index, lazy = false }) {
  const { activeIndex } = useTabs();
  const isActive = activeIndex === index;

  // Lazy: don't mount until first activated
  // Eager: mount but hide with CSS
  if (lazy && !isActive) return null;

  return (
    <div
      role="tabpanel"
      hidden={!isActive}
      className={`tab-panel ${isActive ? 'tab-panel--active' : ''}`}
    >
      {children}
    </div>
  );
}

// ───── Attach sub-components ─────
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

export default Tabs;
```

### Usage

```jsx
function App() {
  return (
    <Tabs defaultIndex={0} onChange={(i) => console.log('Tab:', i)}>
      <Tabs.List>
        <Tabs.Tab index={0}>Profile</Tabs.Tab>
        <Tabs.Tab index={1}>Settings</Tabs.Tab>
        <Tabs.Tab index={2} disabled>Admin</Tabs.Tab>
      </Tabs.List>

      <Tabs.Panel index={0}>
        <h2>Profile Content</h2>
        <p>Your profile details here.</p>
      </Tabs.Panel>
      <Tabs.Panel index={1}>
        <h2>Settings Content</h2>
        <p>Configure your preferences.</p>
      </Tabs.Panel>
      <Tabs.Panel index={2}>
        <h2>Admin Content</h2>
      </Tabs.Panel>
    </Tabs>
  );
}
```

### CSS

```css
.tabs-list {
  display: flex;
  border-bottom: 2px solid #e2e8f0;
  gap: 0;
}

.tab {
  padding: 0.75rem 1.5rem;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 0.95rem;
  color: #64748b;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  transition: all 0.2s ease;
}

.tab:hover:not(.tab--disabled) {
  color: #334155;
  background: #f8fafc;
}

.tab--active {
  color: #3b82f6;
  border-bottom-color: #3b82f6;
  font-weight: 600;
}

.tab--disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.tab-panel {
  padding: 1.5rem 0;
}
```

---

## 2. Select / Dropdown

### Implementation

```jsx
import {
  createContext, useContext, useState, useCallback,
  useMemo, useRef, useEffect
} from 'react';

// ───── Context ─────
const SelectContext = createContext(null);

function useSelect() {
  const ctx = useContext(SelectContext);
  if (!ctx) throw new Error('Select components must be used within <Select>');
  return ctx;
}

// ───── Root Component ─────
function Select({ children, value, onChange, placeholder = 'Select...' }) {
  const [isOpen, setIsOpen] = useState(false);
  const [selectedLabel, setSelectedLabel] = useState('');
  const containerRef = useRef(null);

  const toggle = useCallback(() => setIsOpen(prev => !prev), []);
  const close = useCallback(() => setIsOpen(false), []);

  const select = useCallback((optionValue, label) => {
    onChange?.(optionValue);
    setSelectedLabel(label);
    close();
  }, [onChange, close]);

  // Close on outside click
  useEffect(() => {
    const handleClick = (e) => {
      if (containerRef.current && !containerRef.current.contains(e.target)) {
        close();
      }
    };
    document.addEventListener('mousedown', handleClick);
    return () => document.removeEventListener('mousedown', handleClick);
  }, [close]);

  // Close on Escape
  useEffect(() => {
    const handleKey = (e) => {
      if (e.key === 'Escape') close();
    };
    if (isOpen) {
      document.addEventListener('keydown', handleKey);
      return () => document.removeEventListener('keydown', handleKey);
    }
  }, [isOpen, close]);

  const contextValue = useMemo(
    () => ({ isOpen, toggle, select, value, selectedLabel, placeholder }),
    [isOpen, toggle, select, value, selectedLabel, placeholder]
  );

  return (
    <SelectContext.Provider value={contextValue}>
      <div ref={containerRef} className="select-container">
        {children}
      </div>
    </SelectContext.Provider>
  );
}

// ───── Trigger ─────
function SelectTrigger({ children }) {
  const { isOpen, toggle, selectedLabel, placeholder } = useSelect();

  return (
    <button
      className={`select-trigger ${isOpen ? 'select-trigger--open' : ''}`}
      onClick={toggle}
      aria-haspopup="listbox"
      aria-expanded={isOpen}
    >
      <span className={selectedLabel ? '' : 'select-placeholder'}>
        {children || selectedLabel || placeholder}
      </span>
      <span className="select-chevron">{isOpen ? '▲' : '▼'}</span>
    </button>
  );
}

// ───── Options List ─────
function SelectOptions({ children }) {
  const { isOpen } = useSelect();

  if (!isOpen) return null;

  return (
    <ul className="select-options" role="listbox">
      {children}
    </ul>
  );
}

// ───── Individual Option ─────
function SelectOption({ children, value: optionValue, disabled = false }) {
  const { select, value: selectedValue } = useSelect();
  const isSelected = selectedValue === optionValue;
  const label = typeof children === 'string' ? children : '';

  return (
    <li
      role="option"
      aria-selected={isSelected}
      aria-disabled={disabled}
      className={`select-option ${isSelected ? 'select-option--selected' : ''} ${
        disabled ? 'select-option--disabled' : ''
      }`}
      onClick={() => !disabled && select(optionValue, label)}
    >
      {children}
      {isSelected && <span className="select-check">✓</span>}
    </li>
  );
}

// ───── Attach ─────
Select.Trigger = SelectTrigger;
Select.Options = SelectOptions;
Select.Option = SelectOption;

export default Select;
```

### Usage

```jsx
function App() {
  const [fruit, setFruit] = useState('');

  return (
    <Select value={fruit} onChange={setFruit} placeholder="Choose a fruit">
      <Select.Trigger />
      <Select.Options>
        <Select.Option value="apple">🍎 Apple</Select.Option>
        <Select.Option value="banana">🍌 Banana</Select.Option>
        <Select.Option value="cherry">🍒 Cherry</Select.Option>
        <Select.Option value="durian" disabled>🤢 Durian</Select.Option>
      </Select.Options>
    </Select>
  );
}
```

### CSS

```css
.select-container {
  position: relative;
  width: 250px;
}

.select-trigger {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.6rem 1rem;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  background: white;
  cursor: pointer;
  font-size: 0.95rem;
  transition: border-color 0.2s;
}

.select-trigger:hover {
  border-color: #94a3b8;
}

.select-trigger--open {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.select-placeholder {
  color: #94a3b8;
}

.select-options {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  list-style: none;
  margin: 0;
  padding: 4px;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  z-index: 50;
  max-height: 200px;
  overflow-y: auto;
}

.select-option {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 0.75rem;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s;
}

.select-option:hover:not(.select-option--disabled) {
  background: #f1f5f9;
}

.select-option--selected {
  background: #eff6ff;
  color: #3b82f6;
}

.select-option--disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
```

---

## 3. Menu / Dropdown Menu

### Implementation

```jsx
import {
  createContext, useContext, useState, useCallback,
  useMemo, useRef, useEffect
} from 'react';

const MenuContext = createContext(null);

function useMenu() {
  const ctx = useContext(MenuContext);
  if (!ctx) throw new Error('Menu components must be used within <Menu>');
  return ctx;
}

function Menu({ children }) {
  const [isOpen, setIsOpen] = useState(false);
  const menuRef = useRef(null);

  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen(p => !p), []);

  // Close on outside click
  useEffect(() => {
    if (!isOpen) return;
    const handler = (e) => {
      if (menuRef.current && !menuRef.current.contains(e.target)) close();
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, [isOpen, close]);

  const value = useMemo(
    () => ({ isOpen, open, close, toggle }),
    [isOpen, open, close, toggle]
  );

  return (
    <MenuContext.Provider value={value}>
      <div ref={menuRef} className="menu-container">
        {children}
      </div>
    </MenuContext.Provider>
  );
}

function MenuTrigger({ children }) {
  const { toggle, isOpen } = useMenu();
  return (
    <button
      className="menu-trigger"
      onClick={toggle}
      aria-haspopup="menu"
      aria-expanded={isOpen}
    >
      {children}
    </button>
  );
}

function MenuList({ children }) {
  const { isOpen } = useMenu();
  if (!isOpen) return null;

  return (
    <ul className="menu-list" role="menu">
      {children}
    </ul>
  );
}

function MenuItem({ children, onClick, icon, danger = false, disabled = false }) {
  const { close } = useMenu();

  const handleClick = () => {
    if (disabled) return;
    onClick?.();
    close();
  };

  return (
    <li
      role="menuitem"
      className={`menu-item ${danger ? 'menu-item--danger' : ''} ${
        disabled ? 'menu-item--disabled' : ''
      }`}
      onClick={handleClick}
    >
      {icon && <span className="menu-item-icon">{icon}</span>}
      {children}
    </li>
  );
}

function MenuDivider() {
  return <li className="menu-divider" role="separator" />;
}

Menu.Trigger = MenuTrigger;
Menu.List = MenuList;
Menu.Item = MenuItem;
Menu.Divider = MenuDivider;

export default Menu;
```

### Usage

```jsx
<Menu>
  <Menu.Trigger>⚙️ Actions</Menu.Trigger>
  <Menu.List>
    <Menu.Item icon="✏️" onClick={() => handleEdit()}>
      Edit
    </Menu.Item>
    <Menu.Item icon="📋" onClick={() => handleDuplicate()}>
      Duplicate
    </Menu.Item>
    <Menu.Divider />
    <Menu.Item icon="🗑️" danger onClick={() => handleDelete()}>
      Delete
    </Menu.Item>
  </Menu.List>
</Menu>
```

---

## 4. Toggle Group

A group of toggle buttons where one (or multiple) can be active:

### Implementation

```jsx
const ToggleGroupContext = createContext(null);

function useToggleGroup() {
  const ctx = useContext(ToggleGroupContext);
  if (!ctx) throw new Error('Must be used within <ToggleGroup>');
  return ctx;
}

function ToggleGroup({ children, value, onChange, type = 'single' }) {
  const handleToggle = useCallback((itemValue) => {
    if (type === 'single') {
      onChange(value === itemValue ? null : itemValue);
    } else {
      // multiple
      const set = new Set(value || []);
      if (set.has(itemValue)) set.delete(itemValue);
      else set.add(itemValue);
      onChange(Array.from(set));
    }
  }, [value, onChange, type]);

  const isActive = useCallback((itemValue) => {
    if (type === 'single') return value === itemValue;
    return (value || []).includes(itemValue);
  }, [value, type]);

  const contextValue = useMemo(
    () => ({ handleToggle, isActive }),
    [handleToggle, isActive]
  );

  return (
    <ToggleGroupContext.Provider value={contextValue}>
      <div className="toggle-group" role="group">
        {children}
      </div>
    </ToggleGroupContext.Provider>
  );
}

function ToggleItem({ children, value, disabled = false }) {
  const { handleToggle, isActive } = useToggleGroup();
  const active = isActive(value);

  return (
    <button
      className={`toggle-item ${active ? 'toggle-item--active' : ''}`}
      onClick={() => !disabled && handleToggle(value)}
      aria-pressed={active}
      disabled={disabled}
    >
      {children}
    </button>
  );
}

ToggleGroup.Item = ToggleItem;

export default ToggleGroup;
```

### Usage

```jsx
const [alignment, setAlignment] = useState('left');

<ToggleGroup value={alignment} onChange={setAlignment} type="single">
  <ToggleGroup.Item value="left">⬅️ Left</ToggleGroup.Item>
  <ToggleGroup.Item value="center">↔️ Center</ToggleGroup.Item>
  <ToggleGroup.Item value="right">➡️ Right</ToggleGroup.Item>
</ToggleGroup>
```

---

## Common Patterns Across All Examples

| Pattern | Where Used | Purpose |
|---|---|---|
| `createContext(null)` + guard hook | All | Safe context consumption |
| `useMemo` on context value | All | Prevent unnecessary re-renders |
| `useCallback` on handlers | All | Stable function references |
| `useRef` for DOM references | Select, Menu | Outside click detection |
| ARIA attributes | All | Accessibility |
| `role` attributes | All | Screen reader support |
| Static property attachment | All | Dot-notation API |

---

## Key Takeaway

> Every compound component follows the same skeleton:
> 1. **Context** + **custom hook with guard**
> 2. **Root component** = state + Provider
> 3. **Sub-components** = context consumers
> 4. The magic is deciding **what state to share** and **what API to expose**
