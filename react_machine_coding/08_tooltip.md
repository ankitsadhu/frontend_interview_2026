# Tooltip Component

## Problem
Create a tooltip that shows on hover with positioning (top/bottom/left/right) and optional delay.

## Key Concepts
- `useState` for visibility
- `useRef` for positioning calculations
- `onMouseEnter/Leave` for hover detection
- Portal for rendering outside DOM hierarchy

## MANG-Level Expectations
- **Positioning**: Smart positioning with collision detection
- **Accessibility**: ARIA attributes, keyboard support, focus handling
- **UX**: Delay on show/hide, smooth animations, arrow indicator
- **Performance**: Avoid layout thrashing, efficient positioning
- **Extensibility**: Custom triggers, rich content, themes

## Solution

```jsx
import { useState, useRef, useEffect } from 'react';
import { createPortal } from 'react-dom';

/**
 * TRICKY PART #1: Dynamic positioning
 * - Calculate position based on trigger element
 * - MANG expectation: Handle viewport boundaries
 * - Flip tooltip if it would go off-screen
 */
function Tooltip({
  children,
  content,
  position = 'top',
  delay = 200,
  offset = 8
}) {
  const [isVisible, setIsVisible] = useState(false);
  const [tooltipPosition, setTooltipPosition] = useState({ top: 0, left: 0 });
  const [actualPosition, setActualPosition] = useState(position);
  
  const triggerRef = useRef(null);
  const tooltipRef = useRef(null);
  const showTimeoutRef = useRef(null);
  const hideTimeoutRef = useRef(null);

  /**
   * TRICKY PART #2: Calculate tooltip position
   * - Get trigger element dimensions
   * - Calculate position based on preference
   * - Check viewport boundaries and flip if needed
   * - MANG will ask: "How do you handle edge cases?"
   */
  const calculatePosition = () => {
    if (!triggerRef.current || !tooltipRef.current) return;

    const triggerRect = triggerRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    const viewportWidth = window.innerWidth;
    const viewportHeight = window.innerHeight;

    let top = 0;
    let left = 0;
    let finalPosition = position;

    // Calculate position based on preference
    switch (position) {
      case 'top':
        top = triggerRect.top - tooltipRect.height - offset;
        left = triggerRect.left + (triggerRect.width - tooltipRect.width) / 2;
        
        // Flip to bottom if not enough space
        if (top < 0) {
          finalPosition = 'bottom';
          top = triggerRect.bottom + offset;
        }
        break;

      case 'bottom':
        top = triggerRect.bottom + offset;
        left = triggerRect.left + (triggerRect.width - tooltipRect.width) / 2;
        
        // Flip to top if not enough space
        if (top + tooltipRect.height > viewportHeight) {
          finalPosition = 'top';
          top = triggerRect.top - tooltipRect.height - offset;
        }
        break;

      case 'left':
        top = triggerRect.top + (triggerRect.height - tooltipRect.height) / 2;
        left = triggerRect.left - tooltipRect.width - offset;
        
        // Flip to right if not enough space
        if (left < 0) {
          finalPosition = 'right';
          left = triggerRect.right + offset;
        }
        break;

      case 'right':
        top = triggerRect.top + (triggerRect.height - tooltipRect.height) / 2;
        left = triggerRect.right + offset;
        
        // Flip to left if not enough space
        if (left + tooltipRect.width > viewportWidth) {
          finalPosition = 'left';
          left = triggerRect.left - tooltipRect.width - offset;
        }
        break;
    }

    // Keep tooltip within viewport horizontally
    if (left < 0) left = 8;
    if (left + tooltipRect.width > viewportWidth) {
      left = viewportWidth - tooltipRect.width - 8;
    }

    // Keep tooltip within viewport vertically
    if (top < 0) top = 8;
    if (top + tooltipRect.height > viewportHeight) {
      top = viewportHeight - tooltipRect.height - 8;
    }

    setTooltipPosition({ top, left });
    setActualPosition(finalPosition);
  };

  /**
   * TRICKY PART #3: Delayed show/hide
   * - Use setTimeout for delay
   * - Clear timeout on opposite action
   * - CRITICAL: Cleanup timeouts on unmount
   */
  const handleMouseEnter = () => {
    clearTimeout(hideTimeoutRef.current);
    
    showTimeoutRef.current = setTimeout(() => {
      setIsVisible(true);
    }, delay);
  };

  const handleMouseLeave = () => {
    clearTimeout(showTimeoutRef.current);
    
    hideTimeoutRef.current = setTimeout(() => {
      setIsVisible(false);
    }, 100);
  };

  /**
   * TRICKY PART #4: Update position when visible
   * - Recalculate on visibility change
   * - Handle window resize and scroll
   * - MANG expectation: Efficient event handling
   */
  useEffect(() => {
    if (isVisible) {
      calculatePosition();
      
      const handleUpdate = () => calculatePosition();
      window.addEventListener('resize', handleUpdate);
      window.addEventListener('scroll', handleUpdate, true);
      
      return () => {
        window.removeEventListener('resize', handleUpdate);
        window.removeEventListener('scroll', handleUpdate, true);
      };
    }
  }, [isVisible]);

  // Cleanup timeouts on unmount
  useEffect(() => {
    return () => {
      clearTimeout(showTimeoutRef.current);
      clearTimeout(hideTimeoutRef.current);
    };
  }, []);

  return (
    <>
      <span
        ref={triggerRef}
        onMouseEnter={handleMouseEnter}
        onMouseLeave={handleMouseLeave}
        onFocus={handleMouseEnter}
        onBlur={handleMouseLeave}
        aria-describedby={isVisible ? 'tooltip' : undefined}
      >
        {children}
      </span>

      {isVisible && createPortal(
        <div
          ref={tooltipRef}
          id="tooltip"
          className={`tooltip tooltip-${actualPosition}`}
          style={{
            position: 'fixed',
            top: `${tooltipPosition.top}px`,
            left: `${tooltipPosition.left}px`,
            zIndex: 9999
          }}
          role="tooltip"
          onMouseEnter={handleMouseEnter}
          onMouseLeave={handleMouseLeave}
        >
          {content}
          <div className={`tooltip-arrow tooltip-arrow-${actualPosition}`} />
        </div>,
        document.body
      )}
    </>
  );
}

export default Tooltip;
```

## CSS

```css
.tooltip {
  background: rgba(0, 0, 0, 0.9);
  color: white;
  padding: 0.5rem 0.75rem;
  border-radius: 4px;
  font-size: 0.875rem;
  max-width: 250px;
  word-wrap: break-word;
  pointer-events: none;
  animation: tooltipFadeIn 0.2s ease-out;
}

@keyframes tooltipFadeIn {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* Arrow */
.tooltip-arrow {
  position: absolute;
  width: 0;
  height: 0;
  border-style: solid;
}

.tooltip-arrow-top {
  bottom: -6px;
  left: 50%;
  transform: translateX(-50%);
  border-width: 6px 6px 0 6px;
  border-color: rgba(0, 0, 0, 0.9) transparent transparent transparent;
}

.tooltip-arrow-bottom {
  top: -6px;
  left: 50%;
  transform: translateX(-50%);
  border-width: 0 6px 6px 6px;
  border-color: transparent transparent rgba(0, 0, 0, 0.9) transparent;
}

.tooltip-arrow-left {
  right: -6px;
  top: 50%;
  transform: translateY(-50%);
  border-width: 6px 0 6px 6px;
  border-color: transparent transparent transparent rgba(0, 0, 0, 0.9);
}

.tooltip-arrow-right {
  left: -6px;
  top: 50%;
  transform: translateY(-50%);
  border-width: 6px 6px 6px 0;
  border-color: transparent rgba(0, 0, 0, 0.9) transparent transparent;
}

/* Light theme variant */
.tooltip.light {
  background: white;
  color: #333;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  border: 1px solid #e0e0e0;
}
```

## Usage

```jsx
// Basic usage
<Tooltip content="This is a tooltip">
  <button>Hover me</button>
</Tooltip>

// With custom position
<Tooltip content="Bottom tooltip" position="bottom" delay={500}>
  <span>Hover for bottom tooltip</span>
</Tooltip>

// With rich content
<Tooltip
  content={
    <div>
      <strong>Rich Content</strong>
      <p>You can put any React elements here</p>
    </div>
  }
  position="right"
>
  <button>Rich tooltip</button>
</Tooltip>
```

## Test Cases

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Tooltip from './Tooltip';

jest.useFakeTimers();

describe('Tooltip', () => {
  test('does not show tooltip initially', () => {
    render(
      <Tooltip content="Tooltip text">
        <button>Trigger</button>
      </Tooltip>
    );
    
    expect(screen.queryByText('Tooltip text')).not.toBeInTheDocument();
  });

  test('shows tooltip on mouse enter after delay', async () => {
    render(
      <Tooltip content="Tooltip text" delay={200}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    fireEvent.mouseEnter(screen.getByText('Trigger'));
    
    expect(screen.queryByText('Tooltip text')).not.toBeInTheDocument();
    
    jest.advanceTimersByTime(200);
    
    await waitFor(() => {
      expect(screen.getByText('Tooltip text')).toBeInTheDocument();
    });
  });

  test('hides tooltip on mouse leave', async () => {
    render(
      <Tooltip content="Tooltip text" delay={0}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    fireEvent.mouseEnter(screen.getByText('Trigger'));
    
    await waitFor(() => {
      expect(screen.getByText('Tooltip text')).toBeInTheDocument();
    });
    
    fireEvent.mouseLeave(screen.getByText('Trigger'));
    
    jest.advanceTimersByTime(100);
    
    await waitFor(() => {
      expect(screen.queryByText('Tooltip text')).not.toBeInTheDocument();
    });
  });

  test('shows tooltip on focus', async () => {
    render(
      <Tooltip content="Tooltip text" delay={0}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    fireEvent.focus(screen.getByText('Trigger'));
    
    await waitFor(() => {
      expect(screen.getByText('Tooltip text')).toBeInTheDocument();
    });
  });

  test('hides tooltip on blur', async () => {
    render(
      <Tooltip content="Tooltip text" delay={0}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    const trigger = screen.getByText('Trigger');
    fireEvent.focus(trigger);
    
    await waitFor(() => {
      expect(screen.getByText('Tooltip text')).toBeInTheDocument();
    });
    
    fireEvent.blur(trigger);
    
    jest.advanceTimersByTime(100);
    
    await waitFor(() => {
      expect(screen.queryByText('Tooltip text')).not.toBeInTheDocument();
    });
  });

  test('cancels show timeout on quick mouse leave', () => {
    render(
      <Tooltip content="Tooltip text" delay={200}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    const trigger = screen.getByText('Trigger');
    fireEvent.mouseEnter(trigger);
    
    jest.advanceTimersByTime(100);
    
    fireEvent.mouseLeave(trigger);
    
    jest.advanceTimersByTime(200);
    
    expect(screen.queryByText('Tooltip text')).not.toBeInTheDocument();
  });

  test('renders with correct position class', async () => {
    render(
      <Tooltip content="Tooltip text" position="bottom" delay={0}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    fireEvent.mouseEnter(screen.getByText('Trigger'));
    
    await waitFor(() => {
      const tooltip = screen.getByRole('tooltip');
      expect(tooltip).toHaveClass('tooltip-bottom');
    });
  });

  test('renders arrow with correct position', async () => {
    render(
      <Tooltip content="Tooltip text" position="top" delay={0}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    fireEvent.mouseEnter(screen.getByText('Trigger'));
    
    await waitFor(() => {
      const arrow = document.querySelector('.tooltip-arrow-top');
      expect(arrow).toBeInTheDocument();
    });
  });

  test('has correct ARIA attributes', async () => {
    render(
      <Tooltip content="Tooltip text" delay={0}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    const trigger = screen.getByText('Trigger');
    expect(trigger).not.toHaveAttribute('aria-describedby');
    
    fireEvent.mouseEnter(trigger);
    
    await waitFor(() => {
      expect(trigger).toHaveAttribute('aria-describedby', 'tooltip');
      expect(screen.getByRole('tooltip')).toBeInTheDocument();
    });
  });

  test('renders rich content', async () => {
    render(
      <Tooltip
        content={
          <div>
            <strong>Title</strong>
            <p>Description</p>
          </div>
        }
        delay={0}
      >
        <button>Trigger</button>
      </Tooltip>
    );
    
    fireEvent.mouseEnter(screen.getByText('Trigger'));
    
    await waitFor(() => {
      expect(screen.getByText('Title')).toBeInTheDocument();
      expect(screen.getByText('Description')).toBeInTheDocument();
    });
  });

  test('stays visible when hovering over tooltip', async () => {
    render(
      <Tooltip content="Tooltip text" delay={0}>
        <button>Trigger</button>
      </Tooltip>
    );
    
    fireEvent.mouseEnter(screen.getByText('Trigger'));
    
    await waitFor(() => {
      expect(screen.getByText('Tooltip text')).toBeInTheDocument();
    });
    
    const tooltip = screen.getByRole('tooltip');
    fireEvent.mouseEnter(tooltip);
    
    jest.advanceTimersByTime(200);
    
    expect(screen.getByText('Tooltip text')).toBeInTheDocument();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add click-to-show mode?**
```jsx
function Tooltip({ trigger = 'hover', ...props }) {
  const [isVisible, setIsVisible] = useState(false);

  const handlers = trigger === 'click' ? {
    onClick: () => setIsVisible(!isVisible)
  } : {
    onMouseEnter: handleMouseEnter,
    onMouseLeave: handleMouseLeave
  };

  // Close on outside click for click mode
  useEffect(() => {
    if (trigger === 'click' && isVisible) {
      const handleClickOutside = (e) => {
        if (!triggerRef.current?.contains(e.target)) {
          setIsVisible(false);
        }
      };
      
      document.addEventListener('click', handleClickOutside);
      return () => document.removeEventListener('click', handleClickOutside);
    }
  }, [trigger, isVisible]);

  return <span ref={triggerRef} {...handlers}>{children}</span>;
}
```

**Q: How would you add smart positioning with Popper.js?**
```jsx
import { usePopper } from 'react-popper';

function Tooltip({ content, children }) {
  const [referenceElement, setReferenceElement] = useState(null);
  const [popperElement, setPopperElement] = useState(null);
  const [arrowElement, setArrowElement] = useState(null);

  const { styles, attributes } = usePopper(referenceElement, popperElement, {
    placement: 'top',
    modifiers: [
      { name: 'arrow', options: { element: arrowElement } },
      { name: 'offset', options: { offset: [0, 8] } },
      { name: 'flip', options: { fallbackPlacements: ['bottom', 'left', 'right'] } }
    ]
  });

  return (
    <>
      <span ref={setReferenceElement}>{children}</span>
      <div ref={setPopperElement} style={styles.popper} {...attributes.popper}>
        {content}
        <div ref={setArrowElement} style={styles.arrow} />
      </div>
    </>
  );
}
```

**Q: How would you add keyboard shortcuts (e.g., Escape to close)?**
```jsx
useEffect(() => {
  if (!isVisible) return;

  const handleKeyDown = (e) => {
    if (e.key === 'Escape') {
      setIsVisible(false);
      triggerRef.current?.focus();
    }
  };

  document.addEventListener('keydown', handleKeyDown);
  return () => document.removeEventListener('keydown', handleKeyDown);
}, [isVisible]);
```

**Q: How would you add themes?**
```jsx
function Tooltip({ theme = 'dark', ...props }) {
  return (
    <div className={`tooltip tooltip-${theme}`}>
      {content}
    </div>
  );
}

// CSS
.tooltip-dark {
  background: rgba(0, 0, 0, 0.9);
  color: white;
}

.tooltip-light {
  background: white;
  color: #333;
  border: 1px solid #e0e0e0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.tooltip-error {
  background: #d32f2f;
  color: white;
}
```

**Q: How would you optimize positioning calculations?**
```jsx
// Use requestAnimationFrame for smoother updates
const calculatePosition = useCallback(() => {
  if (!isVisible) return;
  
  requestAnimationFrame(() => {
    // ... positioning logic
  });
}, [isVisible]);

// Debounce resize/scroll events
const debouncedCalculate = useMemo(
  () => debounce(calculatePosition, 100),
  [calculatePosition]
);

useEffect(() => {
  if (isVisible) {
    window.addEventListener('resize', debouncedCalculate);
    window.addEventListener('scroll', debouncedCalculate, true);
    
    return () => {
      window.removeEventListener('resize', debouncedCalculate);
      window.removeEventListener('scroll', debouncedCalculate, true);
    };
  }
}, [isVisible, debouncedCalculate]);
```

## Performance Considerations

1. **Positioning Calculations**
   - Use requestAnimationFrame for smooth updates
   - Debounce resize/scroll events
   - Cache element dimensions when possible

2. **Portal Rendering**
   - Only render tooltip when visible
   - Use createPortal to avoid z-index issues
   - Clean up portal on unmount

3. **Event Listeners**
   - Remove listeners when tooltip is hidden
   - Use passive listeners for scroll
   - Cleanup on unmount

## Edge Cases Handled
- Dynamic positioning with viewport detection
- Position flipping when near edges
- Delayed show/hide with timeout management
- Keyboard support (focus/blur)
- ARIA attributes for accessibility
- Portal rendering for z-index
- Arrow indicator positioning
- Rich content support
- Hover over tooltip keeps it visible
- Cleanup timeouts on unmount
- Window resize and scroll handling
