# Tabs Component

## Problem
Create a tabs component with switchable tab content and active indicator.

## Key Concepts
- `useState` for active tab
- Composition with `children` mapping
- CSS for active indicator animation
- Keyboard navigation

## MANG-Level Expectations
- **Composition**: Flexible API with Tab and TabPanel components
- **Accessibility**: ARIA attributes, keyboard navigation, focus management
- **UX**: Smooth indicator animation, lazy loading, URL sync
- **Performance**: Avoid re-rendering inactive tabs
- **Extensibility**: Vertical tabs, icons, badges, disabled tabs

## Solution

```jsx
import { useState, useRef, useEffect, createContext, useContext } from 'react';

/**
 * TRICKY PART #1: Context for tab state
 * - Share active tab state between Tabs, TabList, and TabPanels
 * - MANG expectation: Explain why Context instead of prop drilling
 * - Cleaner API, easier to use, more flexible
 */
const TabsContext = createContext();

function Tabs({ children, defaultTab = 0, onChange }) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  const handleTabChange = (index) => {
    setActiveTab(index);
    onChange?.(index);
  };

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab: handleTabChange }}>
      <div className="tabs">
        {children}
      </div>
    </TabsContext.Provider>
  );
}

/**
 * TRICKY PART #2: Active indicator animation
 * - Calculate position and width dynamically
 * - MANG will ask: "How do you animate the indicator?"
 * - Use refs to get tab dimensions, CSS transform for smooth animation
 */
function TabList({ children }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  const tabRefs = useRef([]);
  const [indicatorStyle, setIndicatorStyle] = useState({});

  /**
   * TRICKY PART #3: Update indicator on tab change
   * - Measure active tab dimensions
   * - Update indicator position with CSS transform
   * - CRITICAL: Handle window resize
   */
  useEffect(() => {
    const updateIndicator = () => {
      const activeTabElement = tabRefs.current[activeTab];
      if (activeTabElement) {
        setIndicatorStyle({
          width: activeTabElement.offsetWidth,
          transform: `translateX(${activeTabElement.offsetLeft}px)`
        });
      }
    };

    updateIndicator();
    window.addEventListener('resize', updateIndicator);
    return () => window.removeEventListener('resize', updateIndicator);
  }, [activeTab]);

  /**
   * TRICKY PART #4: Keyboard navigation
   * - Arrow keys to navigate tabs
   * - Home/End to jump to first/last
   * - MANG expectation: Full keyboard accessibility
   */
  const handleKeyDown = (e, index) => {
    const tabCount = tabRefs.current.length;

    switch (e.key) {
      case 'ArrowRight':
        e.preventDefault();
        const nextIndex = (index + 1) % tabCount;
        setActiveTab(nextIndex);
        tabRefs.current[nextIndex]?.focus();
        break;
      case 'ArrowLeft':
        e.preventDefault();
        const prevIndex = (index - 1 + tabCount) % tabCount;
        setActiveTab(prevIndex);
        tabRefs.current[prevIndex]?.focus();
        break;
      case 'Home':
        e.preventDefault();
        setActiveTab(0);
        tabRefs.current[0]?.focus();
        break;
      case 'End':
        e.preventDefault();
        setActiveTab(tabCount - 1);
        tabRefs.current[tabCount - 1]?.focus();
        break;
    }
  };

  return (
    <div className="tab-list" role="tablist">
      {children.map((child, index) => (
        <button
          key={index}
          ref={(el) => (tabRefs.current[index] = el)}
          className={`tab ${activeTab === index ? 'active' : ''}`}
          onClick={() => setActiveTab(index)}
          onKeyDown={(e) => handleKeyDown(e, index)}
          role="tab"
          aria-selected={activeTab === index}
          aria-controls={`tabpanel-${index}`}
          id={`tab-${index}`}
          tabIndex={activeTab === index ? 0 : -1}
        >
          {child}
        </button>
      ))}
      <div className="tab-indicator" style={indicatorStyle} />
    </div>
  );
}

function TabPanels({ children }) {
  const { activeTab } = useContext(TabsContext);

  return (
    <div className="tab-panels">
      {children.map((child, index) => (
        <div
          key={index}
          className={`tab-panel ${activeTab === index ? 'active' : ''}`}
          role="tabpanel"
          id={`tabpanel-${index}`}
          aria-labelledby={`tab-${index}`}
          hidden={activeTab !== index}
        >
          {child}
        </div>
      ))}
    </div>
  );
}

export { Tabs, TabList, TabPanels };
```

## CSS

```css
.tabs {
  width: 100%;
}

.tab-list {
  position: relative;
  display: flex;
  border-bottom: 2px solid #e0e0e0;
  gap: 0.5rem;
}

.tab {
  padding: 0.75rem 1.5rem;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1rem;
  color: #666;
  transition: color 0.2s;
  position: relative;
}

.tab:hover {
  color: #333;
}

.tab.active {
  color: #007bff;
  font-weight: 500;
}

.tab:focus {
  outline: 2px solid #007bff;
  outline-offset: 2px;
}

/* Active indicator */
.tab-indicator {
  position: absolute;
  bottom: -2px;
  left: 0;
  height: 2px;
  background: #007bff;
  transition: transform 0.3s ease, width 0.3s ease;
}

.tab-panels {
  padding: 1.5rem 0;
}

.tab-panel {
  display: none;
  animation: fadeIn 0.3s ease-in;
}

.tab-panel.active {
  display: block;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Vertical tabs variant */
.tabs.vertical {
  display: flex;
}

.tabs.vertical .tab-list {
  flex-direction: column;
  border-bottom: none;
  border-right: 2px solid #e0e0e0;
  min-width: 200px;
}

.tabs.vertical .tab-indicator {
  width: 2px;
  height: auto;
  right: -2px;
  left: auto;
  bottom: auto;
  transform: translateY(0) !important;
}

.tabs.vertical .tab-panels {
  flex: 1;
  padding: 0 1.5rem;
}
```

## Usage

```jsx
import { Tabs, TabList, TabPanels } from './Tabs';

function App() {
  return (
    <Tabs defaultTab={0} onChange={(index) => console.log('Tab changed:', index)}>
      <TabList>
        <span>Profile</span>
        <span>Settings</span>
        <span>Notifications</span>
      </TabList>
      
      <TabPanels>
        <div>
          <h2>Profile Content</h2>
          <p>Your profile information goes here.</p>
        </div>
        <div>
          <h2>Settings Content</h2>
          <p>Your settings go here.</p>
        </div>
        <div>
          <h2>Notifications Content</h2>
          <p>Your notifications go here.</p>
        </div>
      </TabPanels>
    </Tabs>
  );
}
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Tabs, TabList, TabPanels } from './Tabs';

const TabsExample = () => (
  <Tabs defaultTab={0}>
    <TabList>
      <span>Tab 1</span>
      <span>Tab 2</span>
      <span>Tab 3</span>
    </TabList>
    <TabPanels>
      <div>Content 1</div>
      <div>Content 2</div>
      <div>Content 3</div>
    </TabPanels>
  </Tabs>
);

describe('Tabs', () => {
  test('renders all tabs', () => {
    render(<TabsExample />);
    
    expect(screen.getByText('Tab 1')).toBeInTheDocument();
    expect(screen.getByText('Tab 2')).toBeInTheDocument();
    expect(screen.getByText('Tab 3')).toBeInTheDocument();
  });

  test('shows first tab content by default', () => {
    render(<TabsExample />);
    
    expect(screen.getByText('Content 1')).toBeVisible();
    expect(screen.queryByText('Content 2')).not.toBeVisible();
  });

  test('switches content on tab click', () => {
    render(<TabsExample />);
    
    fireEvent.click(screen.getByText('Tab 2'));
    
    expect(screen.queryByText('Content 1')).not.toBeVisible();
    expect(screen.getByText('Content 2')).toBeVisible();
  });

  test('applies active class to selected tab', () => {
    render(<TabsExample />);
    
    const tab1 = screen.getByText('Tab 1').closest('button');
    expect(tab1).toHaveClass('active');
    
    fireEvent.click(screen.getByText('Tab 2'));
    
    const tab2 = screen.getByText('Tab 2').closest('button');
    expect(tab2).toHaveClass('active');
    expect(tab1).not.toHaveClass('active');
  });

  test('navigates with arrow keys', () => {
    render(<TabsExample />);
    
    const tab1 = screen.getByText('Tab 1').closest('button');
    const tab2 = screen.getByText('Tab 2').closest('button');
    
    tab1.focus();
    fireEvent.keyDown(tab1, { key: 'ArrowRight' });
    
    expect(tab2).toHaveFocus();
    expect(screen.getByText('Content 2')).toBeVisible();
  });

  test('wraps around with arrow keys', () => {
    render(<TabsExample />);
    
    const tab1 = screen.getByText('Tab 1').closest('button');
    const tab3 = screen.getByText('Tab 3').closest('button');
    
    tab3.focus();
    fireEvent.keyDown(tab3, { key: 'ArrowRight' });
    
    expect(tab1).toHaveFocus();
  });

  test('jumps to first tab with Home key', () => {
    render(<TabsExample />);
    
    const tab1 = screen.getByText('Tab 1').closest('button');
    const tab3 = screen.getByText('Tab 3').closest('button');
    
    tab3.focus();
    fireEvent.keyDown(tab3, { key: 'Home' });
    
    expect(tab1).toHaveFocus();
    expect(screen.getByText('Content 1')).toBeVisible();
  });

  test('jumps to last tab with End key', () => {
    render(<TabsExample />);
    
    const tab1 = screen.getByText('Tab 1').closest('button');
    const tab3 = screen.getByText('Tab 3').closest('button');
    
    tab1.focus();
    fireEvent.keyDown(tab1, { key: 'End' });
    
    expect(tab3).toHaveFocus();
    expect(screen.getByText('Content 3')).toBeVisible();
  });

  test('calls onChange callback', () => {
    const handleChange = jest.fn();
    
    render(
      <Tabs defaultTab={0} onChange={handleChange}>
        <TabList>
          <span>Tab 1</span>
          <span>Tab 2</span>
        </TabList>
        <TabPanels>
          <div>Content 1</div>
          <div>Content 2</div>
        </TabPanels>
      </Tabs>
    );
    
    fireEvent.click(screen.getByText('Tab 2'));
    
    expect(handleChange).toHaveBeenCalledWith(1);
  });

  test('respects defaultTab prop', () => {
    render(
      <Tabs defaultTab={1}>
        <TabList>
          <span>Tab 1</span>
          <span>Tab 2</span>
        </TabList>
        <TabPanels>
          <div>Content 1</div>
          <div>Content 2</div>
        </TabPanels>
      </Tabs>
    );
    
    expect(screen.getByText('Content 2')).toBeVisible();
  });

  test('has correct ARIA attributes', () => {
    render(<TabsExample />);
    
    const tab1 = screen.getByText('Tab 1').closest('button');
    expect(tab1).toHaveAttribute('role', 'tab');
    expect(tab1).toHaveAttribute('aria-selected', 'true');
    expect(tab1).toHaveAttribute('aria-controls', 'tabpanel-0');
    
    const panel1 = screen.getByText('Content 1').closest('div');
    expect(panel1).toHaveAttribute('role', 'tabpanel');
    expect(panel1).toHaveAttribute('aria-labelledby', 'tab-0');
  });

  test('only active tab is in tab order', () => {
    render(<TabsExample />);
    
    const tab1 = screen.getByText('Tab 1').closest('button');
    const tab2 = screen.getByText('Tab 2').closest('button');
    
    expect(tab1).toHaveAttribute('tabIndex', '0');
    expect(tab2).toHaveAttribute('tabIndex', '-1');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add lazy loading for tab content?**
```jsx
function TabPanels({ children }) {
  const { activeTab } = useContext(TabsContext);
  const [loadedTabs, setLoadedTabs] = useState(new Set([activeTab]));

  useEffect(() => {
    setLoadedTabs(prev => new Set([...prev, activeTab]));
  }, [activeTab]);

  return (
    <div className="tab-panels">
      {children.map((child, index) => (
        <div
          key={index}
          className={`tab-panel ${activeTab === index ? 'active' : ''}`}
          hidden={activeTab !== index}
        >
          {loadedTabs.has(index) ? child : null}
        </div>
      ))}
    </div>
  );
}
```

**Q: How would you sync tabs with URL?**
```jsx
import { useHistory, useLocation } from 'react-router-dom';

function Tabs({ children, defaultTab = 0 }) {
  const history = useHistory();
  const location = useLocation();
  
  const getTabFromURL = () => {
    const params = new URLSearchParams(location.search);
    return parseInt(params.get('tab') || defaultTab);
  };

  const [activeTab, setActiveTab] = useState(getTabFromURL);

  const handleTabChange = (index) => {
    setActiveTab(index);
    history.push(`?tab=${index}`);
  };

  // ... rest of component
}
```

**Q: How would you add icons and badges to tabs?**
```jsx
function Tab({ icon, badge, children }) {
  return (
    <span className="tab-content">
      {icon && <span className="tab-icon">{icon}</span>}
      <span className="tab-label">{children}</span>
      {badge && <span className="tab-badge">{badge}</span>}
    </span>
  );
}

// Usage
<TabList>
  <Tab icon={<UserIcon />} badge={3}>Profile</Tab>
  <Tab icon={<SettingsIcon />}>Settings</Tab>
</TabList>
```

**Q: How would you add disabled tabs?**
```jsx
function TabList({ children, disabledTabs = [] }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);

  return (
    <div className="tab-list" role="tablist">
      {children.map((child, index) => {
        const isDisabled = disabledTabs.includes(index);
        
        return (
          <button
            key={index}
            className={`tab ${activeTab === index ? 'active' : ''} ${isDisabled ? 'disabled' : ''}`}
            onClick={() => !isDisabled && setActiveTab(index)}
            disabled={isDisabled}
            aria-disabled={isDisabled}
          >
            {child}
          </button>
        );
      })}
    </div>
  );
}
```

**Q: How would you add vertical tabs?**
```jsx
function Tabs({ children, orientation = 'horizontal' }) {
  return (
    <div className={`tabs ${orientation}`}>
      {children}
    </div>
  );
}

// CSS for vertical
.tabs.vertical {
  display: flex;
}

.tabs.vertical .tab-list {
  flex-direction: column;
  border-bottom: none;
  border-right: 2px solid #e0e0e0;
}

.tabs.vertical .tab-indicator {
  width: 2px;
  height: var(--indicator-height);
  transform: translateY(var(--indicator-top)) !important;
}
```

**Q: How would you add controlled mode?**
```jsx
function Tabs({ 
  children, 
  defaultTab = 0,
  activeTab: controlledTab,
  onChange 
}) {
  const [internalTab, setInternalTab] = useState(defaultTab);
  
  const activeTab = controlledTab !== undefined ? controlledTab : internalTab;
  
  const handleTabChange = (index) => {
    if (onChange) {
      onChange(index);
    } else {
      setInternalTab(index);
    }
  };

  // ... rest of component
}

// Usage
const [activeTab, setActiveTab] = useState(0);

<Tabs activeTab={activeTab} onChange={setActiveTab}>
  {/* tabs */}
</Tabs>
```

## Performance Considerations

1. **Avoid Re-renders**
   - Use Context to prevent prop drilling
   - Memoize tab content if expensive
   - Only render active tab content (lazy loading)

2. **Indicator Animation**
   - Use CSS transform instead of left/right
   - Debounce resize events
   - Use requestAnimationFrame for smooth updates

3. **Large Number of Tabs**
   - Implement scrollable tab list
   - Add overflow indicators
   - Virtual scrolling for 100+ tabs

## Edge Cases Handled
- Context for state management
- Active indicator with smooth animation
- Keyboard navigation (arrows, home, end)
- ARIA attributes for accessibility
- Focus management (only active tab in tab order)
- Lazy loading support
- URL synchronization
- Controlled/uncontrolled modes
- Disabled tabs
- Vertical orientation
- Icons and badges
- Window resize handling
