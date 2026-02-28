# Accordion / FAQ Component

## Problem
Create an accordion component with single or multi-expand functionality.

## Key Concepts
- `useState` for managing expanded state
- Conditional rendering for content
- Array state management
- CSS transitions for smooth animations

## MANG-Level Expectations
- **State Management**: Handle both single and multi-expand modes
- **UX**: Smooth animations, keyboard navigation, icons
- **Accessibility**: ARIA attributes, keyboard support, focus management
- **Performance**: Avoid unnecessary re-renders, lazy content loading
- **Extensibility**: Nested accordions, controlled/uncontrolled modes

## Solution

```jsx
import { useState } from 'react';

/**
 * TRICKY PART #1: Single vs Multi-expand mode
 * - Single: Only one item open at a time (use single value)
 * - Multi: Multiple items can be open (use Set or array)
 * - MANG expectation: Explain the data structure choice
 */
function Accordion({ items, allowMultiple = false, defaultExpanded = [] }) {
  // For single mode: store single index or null
  // For multi mode: store Set of indices
  const [expanded, setExpanded] = useState(() => {
    if (allowMultiple) {
      return new Set(defaultExpanded);
    }
    return defaultExpanded.length > 0 ? defaultExpanded[0] : null;
  });

  /**
   * TRICKY PART #2: Toggle logic for both modes
   * - Single mode: Close current if clicking same, otherwise switch
   * - Multi mode: Add/remove from Set
   * - MANG will ask: "How do you handle the state differently?"
   */
  const toggleItem = (index) => {
    if (allowMultiple) {
      setExpanded(prev => {
        const newSet = new Set(prev);
        if (newSet.has(index)) {
          newSet.delete(index);
        } else {
          newSet.add(index);
        }
        return newSet;
      });
    } else {
      setExpanded(prev => prev === index ? null : index);
    }
  };

  const isExpanded = (index) => {
    if (allowMultiple) {
      return expanded.has(index);
    }
    return expanded === index;
  };

  /**
   * TRICKY PART #3: Keyboard navigation
   * - Arrow keys to navigate between items
   * - Enter/Space to toggle
   * - Home/End to jump to first/last
   * - MANG expectation: Full keyboard accessibility
   */
  const handleKeyDown = (e, index) => {
    switch (e.key) {
      case 'Enter':
      case ' ':
        e.preventDefault();
        toggleItem(index);
        break;
      case 'ArrowDown':
        e.preventDefault();
        if (index < items.length - 1) {
          document.getElementById(`accordion-header-${index + 1}`)?.focus();
        }
        break;
      case 'ArrowUp':
        e.preventDefault();
        if (index > 0) {
          document.getElementById(`accordion-header-${index - 1}`)?.focus();
        }
        break;
      case 'Home':
        e.preventDefault();
        document.getElementById('accordion-header-0')?.focus();
        break;
      case 'End':
        e.preventDefault();
        document.getElementById(`accordion-header-${items.length - 1}`)?.focus();
        break;
    }
  };

  return (
    <div className="accordion" role="region" aria-label="Accordion">
      {items.map((item, index) => {
        const expanded = isExpanded(index);
        
        return (
          <div key={index} className="accordion-item">
            <button
              id={`accordion-header-${index}`}
              className="accordion-header"
              onClick={() => toggleItem(index)}
              onKeyDown={(e) => handleKeyDown(e, index)}
              aria-expanded={expanded}
              aria-controls={`accordion-content-${index}`}
            >
              <span className="accordion-title">{item.title}</span>
              <span 
                className={`accordion-icon ${expanded ? 'expanded' : ''}`}
                aria-hidden="true"
              >
                {expanded ? '−' : '+'}
              </span>
            </button>
            
            {/**
             * TRICKY PART #4: Animated content reveal
             * - Use max-height for smooth animation
             * - MANG might ask: "Why not height: auto?"
             * - height: auto doesn't animate, need specific value
             */}
            <div
              id={`accordion-content-${index}`}
              className={`accordion-content ${expanded ? 'expanded' : ''}`}
              role="region"
              aria-labelledby={`accordion-header-${index}`}
              hidden={!expanded}
            >
              <div className="accordion-content-inner">
                {item.content}
              </div>
            </div>
          </div>
        );
      })}
    </div>
  );
}

export default Accordion;
```

## CSS

```css
.accordion {
  width: 100%;
  max-width: 800px;
  margin: 0 auto;
}

.accordion-item {
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  margin-bottom: 0.5rem;
  overflow: hidden;
}

.accordion-header {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.5rem;
  background: white;
  border: none;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  text-align: left;
  transition: background 0.2s;
}

.accordion-header:hover {
  background: #f5f5f5;
}

.accordion-header:focus {
  outline: 2px solid #007bff;
  outline-offset: -2px;
}

.accordion-title {
  flex: 1;
}

.accordion-icon {
  font-size: 1.5rem;
  font-weight: bold;
  color: #666;
  transition: transform 0.3s ease;
  line-height: 1;
}

.accordion-icon.expanded {
  transform: rotate(180deg);
}

.accordion-content {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease-out;
}

.accordion-content.expanded {
  max-height: 1000px; /* Adjust based on content */
}

.accordion-content-inner {
  padding: 1rem 1.5rem;
  color: #666;
  line-height: 1.6;
}

/* Alternative: Use grid for smoother animation */
.accordion-content-alt {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 0.3s ease-out;
}

.accordion-content-alt.expanded {
  grid-template-rows: 1fr;
}

.accordion-content-alt > div {
  overflow: hidden;
}
```

## Usage

```jsx
const faqItems = [
  {
    title: 'What is React?',
    content: 'React is a JavaScript library for building user interfaces.'
  },
  {
    title: 'What are React Hooks?',
    content: 'Hooks are functions that let you use state and other React features in functional components.'
  },
  {
    title: 'What is JSX?',
    content: 'JSX is a syntax extension for JavaScript that looks similar to HTML.'
  }
];

// Single expand mode
<Accordion items={faqItems} allowMultiple={false} />

// Multi expand mode
<Accordion items={faqItems} allowMultiple={true} defaultExpanded={[0, 1]} />
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import Accordion from './Accordion';

const mockItems = [
  { title: 'Item 1', content: 'Content 1' },
  { title: 'Item 2', content: 'Content 2' },
  { title: 'Item 3', content: 'Content 3' }
];

describe('Accordion', () => {
  test('renders all accordion items', () => {
    render(<Accordion items={mockItems} />);
    
    expect(screen.getByText('Item 1')).toBeInTheDocument();
    expect(screen.getByText('Item 2')).toBeInTheDocument();
    expect(screen.getByText('Item 3')).toBeInTheDocument();
  });

  test('content is hidden by default', () => {
    render(<Accordion items={mockItems} />);
    
    expect(screen.queryByText('Content 1')).not.toBeVisible();
  });

  test('expands item on click', () => {
    render(<Accordion items={mockItems} />);
    
    const header = screen.getByText('Item 1');
    fireEvent.click(header);
    
    expect(screen.getByText('Content 1')).toBeVisible();
  });

  test('collapses item on second click', () => {
    render(<Accordion items={mockItems} />);
    
    const header = screen.getByText('Item 1');
    fireEvent.click(header);
    expect(screen.getByText('Content 1')).toBeVisible();
    
    fireEvent.click(header);
    expect(screen.queryByText('Content 1')).not.toBeVisible();
  });

  test('single mode: closes previous item when opening new one', () => {
    render(<Accordion items={mockItems} allowMultiple={false} />);
    
    fireEvent.click(screen.getByText('Item 1'));
    expect(screen.getByText('Content 1')).toBeVisible();
    
    fireEvent.click(screen.getByText('Item 2'));
    expect(screen.queryByText('Content 1')).not.toBeVisible();
    expect(screen.getByText('Content 2')).toBeVisible();
  });

  test('multi mode: keeps previous items open', () => {
    render(<Accordion items={mockItems} allowMultiple={true} />);
    
    fireEvent.click(screen.getByText('Item 1'));
    fireEvent.click(screen.getByText('Item 2'));
    
    expect(screen.getByText('Content 1')).toBeVisible();
    expect(screen.getByText('Content 2')).toBeVisible();
  });

  test('respects defaultExpanded in single mode', () => {
    render(<Accordion items={mockItems} allowMultiple={false} defaultExpanded={[1]} />);
    
    expect(screen.getByText('Content 2')).toBeVisible();
  });

  test('respects defaultExpanded in multi mode', () => {
    render(<Accordion items={mockItems} allowMultiple={true} defaultExpanded={[0, 2]} />);
    
    expect(screen.getByText('Content 1')).toBeVisible();
    expect(screen.getByText('Content 3')).toBeVisible();
  });

  test('toggles on Enter key', () => {
    render(<Accordion items={mockItems} />);
    
    const header = screen.getByText('Item 1');
    fireEvent.keyDown(header, { key: 'Enter' });
    
    expect(screen.getByText('Content 1')).toBeVisible();
  });

  test('toggles on Space key', () => {
    render(<Accordion items={mockItems} />);
    
    const header = screen.getByText('Item 1');
    fireEvent.keyDown(header, { key: ' ' });
    
    expect(screen.getByText('Content 1')).toBeVisible();
  });

  test('navigates with arrow keys', () => {
    render(<Accordion items={mockItems} />);
    
    const header1 = screen.getByText('Item 1');
    const header2 = screen.getByText('Item 2');
    
    header1.focus();
    fireEvent.keyDown(header1, { key: 'ArrowDown' });
    
    expect(header2).toHaveFocus();
  });

  test('jumps to first item with Home key', () => {
    render(<Accordion items={mockItems} />);
    
    const header1 = screen.getByText('Item 1');
    const header3 = screen.getByText('Item 3');
    
    header3.focus();
    fireEvent.keyDown(header3, { key: 'Home' });
    
    expect(header1).toHaveFocus();
  });

  test('jumps to last item with End key', () => {
    render(<Accordion items={mockItems} />);
    
    const header1 = screen.getByText('Item 1');
    const header3 = screen.getByText('Item 3');
    
    header1.focus();
    fireEvent.keyDown(header1, { key: 'End' });
    
    expect(header3).toHaveFocus();
  });

  test('has correct ARIA attributes', () => {
    render(<Accordion items={mockItems} />);
    
    const header = screen.getByText('Item 1');
    expect(header).toHaveAttribute('aria-expanded', 'false');
    
    fireEvent.click(header);
    expect(header).toHaveAttribute('aria-expanded', 'true');
  });

  test('shows correct icon for expanded/collapsed state', () => {
    render(<Accordion items={mockItems} />);
    
    const header = screen.getByText('Item 1');
    const icon = header.querySelector('.accordion-icon');
    
    expect(icon.textContent).toBe('+');
    
    fireEvent.click(header);
    expect(icon.textContent).toBe('−');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add controlled mode?**
```jsx
function Accordion({ 
  items, 
  allowMultiple = false,
  expanded: controlledExpanded,
  onToggle
}) {
  const [internalExpanded, setInternalExpanded] = useState(
    allowMultiple ? new Set() : null
  );

  // Use controlled value if provided, otherwise internal state
  const expanded = controlledExpanded !== undefined 
    ? controlledExpanded 
    : internalExpanded;

  const toggleItem = (index) => {
    if (onToggle) {
      onToggle(index);
    } else {
      setInternalExpanded(/* toggle logic */);
    }
  };

  // ... rest of component
}

// Usage
const [expanded, setExpanded] = useState(new Set([0]));

<Accordion
  items={items}
  allowMultiple={true}
  expanded={expanded}
  onToggle={(index) => {
    const newSet = new Set(expanded);
    newSet.has(index) ? newSet.delete(index) : newSet.add(index);
    setExpanded(newSet);
  }}
/>
```

**Q: How would you add nested accordions?**
```jsx
function NestedAccordion({ items }) {
  return (
    <Accordion
      items={items.map(item => ({
        title: item.title,
        content: item.children ? (
          <NestedAccordion items={item.children} />
        ) : (
          item.content
        )
      }))}
    />
  );
}
```

**Q: How would you add lazy loading for content?**
```jsx
function Accordion({ items, loadContent }) {
  const [loadedContent, setLoadedContent] = useState({});

  const toggleItem = async (index) => {
    // Load content on first expand
    if (!loadedContent[index] && loadContent) {
      const content = await loadContent(index);
      setLoadedContent(prev => ({ ...prev, [index]: content }));
    }
    
    // ... toggle logic
  };

  return (
    <div className="accordion-content">
      {loadedContent[index] || 'Loading...'}
    </div>
  );
}
```

**Q: How would you add animation with CSS Grid?**
```css
/* Better animation using CSS Grid */
.accordion-content {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 0.3s ease-out;
}

.accordion-content.expanded {
  grid-template-rows: 1fr;
}

.accordion-content > div {
  overflow: hidden;
}
```

**Q: How would you add expand/collapse all functionality?**
```jsx
function AccordionWithControls({ items }) {
  const [expanded, setExpanded] = useState(new Set());

  const expandAll = () => {
    setExpanded(new Set(items.map((_, i) => i)));
  };

  const collapseAll = () => {
    setExpanded(new Set());
  };

  return (
    <>
      <div className="accordion-controls">
        <button onClick={expandAll}>Expand All</button>
        <button onClick={collapseAll}>Collapse All</button>
      </div>
      <Accordion
        items={items}
        allowMultiple={true}
        expanded={expanded}
        onToggle={(index) => {
          const newSet = new Set(expanded);
          newSet.has(index) ? newSet.delete(index) : newSet.add(index);
          setExpanded(newSet);
        }}
      />
    </>
  );
}
```

## Performance Considerations

1. **Avoid Re-renders**
   - Memoize accordion items if they're expensive to render
   - Use React.memo for individual accordion items
   - useCallback for toggle handlers

2. **Animation Performance**
   - Use CSS transforms instead of height when possible
   - CSS Grid animation is smoother than max-height
   - Consider using framer-motion for complex animations

3. **Large Lists**
   - Lazy load content on expand
   - Virtualize if rendering 100+ items
   - Debounce rapid toggle clicks

## Edge Cases Handled
- Single vs multi-expand modes
- Default expanded items
- Keyboard navigation (arrows, enter, space, home, end)
- ARIA attributes for accessibility
- Smooth animations with CSS
- Icon rotation on expand/collapse
- Focus management
- Empty items array
- Nested accordions support
- Controlled/uncontrolled modes
