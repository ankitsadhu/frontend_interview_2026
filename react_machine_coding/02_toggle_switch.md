# Toggle Switch Component

## Problem
Create an animated toggle switch (on/off) with label and controlled component pattern.

## Key Concepts
- `useState` for toggle state
- CSS transitions for smooth animation
- Controlled component pattern
- Accessibility with keyboard support

## MANG-Level Expectations
- **Accessibility**: ARIA attributes, keyboard support, focus states
- **UX**: Smooth animations, disabled state, loading state
- **Flexibility**: Controlled/uncontrolled modes, custom labels, sizes
- **Performance**: Avoid unnecessary re-renders
- **Extensibility**: Custom colors, icons, confirmation dialogs

## Solution

```jsx
import { useState } from 'react';

/**
 * TRICKY PART #1: Controlled vs Uncontrolled
 * - Support both modes for flexibility
 * - MANG expectation: Explain when to use each
 * - Controlled: Parent manages state (forms, validation)
 * - Uncontrolled: Component manages own state (simple cases)
 */
function ToggleSwitch({
  checked: controlledChecked,
  defaultChecked = false,
  onChange,
  disabled = false,
  loading = false,
  label,
  size = 'medium',
  onLabel = 'ON',
  offLabel = 'OFF',
  showLabels = false,
  id
}) {
  const [internalChecked, setInternalChecked] = useState(defaultChecked);

  // Use controlled value if provided, otherwise use internal state
  const isControlled = controlledChecked !== undefined;
  const checked = isControlled ? controlledChecked : internalChecked;

  /**
   * TRICKY PART #2: Handle toggle with validation
   * - Support async onChange (e.g., API calls)
   * - MANG will ask: "How do you handle async state changes?"
   * - Use loading state, prevent rapid toggles
   */
  const handleToggle = async () => {
    if (disabled || loading) return;

    const newValue = !checked;

    // For uncontrolled mode, update internal state
    if (!isControlled) {
      setInternalChecked(newValue);
    }

    // Call onChange callback
    if (onChange) {
      // Support both sync and async onChange
      await onChange(newValue);
    }
  };

  /**
   * TRICKY PART #3: Keyboard accessibility
   * - Space/Enter to toggle
   * - MANG expectation: Full keyboard support
   */
  const handleKeyDown = (e) => {
    if (e.key === ' ' || e.key === 'Enter') {
      e.preventDefault();
      handleToggle();
    }
  };

  const sizeClasses = {
    small: 'toggle-small',
    medium: 'toggle-medium',
    large: 'toggle-large'
  };

  const toggleId = id || `toggle-${Math.random().toString(36).substr(2, 9)}`;

  return (
    <div className={`toggle-container ${disabled ? 'disabled' : ''}`}>
      {label && (
        <label htmlFor={toggleId} className="toggle-label">
          {label}
        </label>
      )}

      <button
        id={toggleId}
        type="button"
        role="switch"
        aria-checked={checked}
        aria-label={label || (checked ? onLabel : offLabel)}
        disabled={disabled || loading}
        onClick={handleToggle}
        onKeyDown={handleKeyDown}
        className={`toggle-switch ${sizeClasses[size]} ${checked ? 'checked' : ''} ${
          loading ? 'loading' : ''
        }`}
      >
        <span className="toggle-track">
          {showLabels && (
            <span className="toggle-track-label">
              {checked ? onLabel : offLabel}
            </span>
          )}
        </span>
        <span className="toggle-thumb">
          {loading && <span className="toggle-spinner" />}
        </span>
      </button>
    </div>
  );
}

export default ToggleSwitch;
```

## CSS

```css
.toggle-container {
  display: inline-flex;
  align-items: center;
  gap: 0.75rem;
}

.toggle-container.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.toggle-label {
  font-size: 0.875rem;
  color: #333;
  cursor: pointer;
}

.toggle-switch {
  position: relative;
  border: none;
  background: none;
  cursor: pointer;
  padding: 0;
  transition: opacity 0.2s;
}

.toggle-switch:disabled {
  cursor: not-allowed;
}

.toggle-switch:focus {
  outline: 2px solid #007bff;
  outline-offset: 2px;
}

.toggle-track {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #ccc;
  border-radius: 100px;
  transition: background 0.3s ease;
  position: relative;
}

.toggle-switch.checked .toggle-track {
  background: #4caf50;
}

.toggle-thumb {
  position: absolute;
  background: white;
  border-radius: 50%;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  transition: transform 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Sizes */
.toggle-small .toggle-track {
  width: 40px;
  height: 20px;
  padding: 2px;
}

.toggle-small .toggle-thumb {
  width: 16px;
  height: 16px;
  left: 2px;
}

.toggle-small.checked .toggle-thumb {
  transform: translateX(20px);
}

.toggle-medium .toggle-track {
  width: 50px;
  height: 26px;
  padding: 3px;
}

.toggle-medium .toggle-thumb {
  width: 20px;
  height: 20px;
  left: 3px;
}

.toggle-medium.checked .toggle-thumb {
  transform: translateX(24px);
}

.toggle-large .toggle-track {
  width: 60px;
  height: 32px;
  padding: 4px;
}

.toggle-large .toggle-thumb {
  width: 24px;
  height: 24px;
  left: 4px;
}

.toggle-large.checked .toggle-thumb {
  transform: translateX(28px);
}

/* Track labels */
.toggle-track-label {
  font-size: 0.625rem;
  font-weight: bold;
  color: white;
  padding: 0 0.5rem;
  user-select: none;
}

/* Loading spinner */
.toggle-spinner {
  width: 12px;
  height: 12px;
  border: 2px solid #f3f3f3;
  border-top: 2px solid #666;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* Hover effect */
.toggle-switch:not(:disabled):hover .toggle-thumb {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}
```

## Usage

```jsx
// Uncontrolled
<ToggleSwitch
  defaultChecked={false}
  onChange={(checked) => console.log('Toggled:', checked)}
  label="Enable notifications"
/>

// Controlled
function App() {
  const [isEnabled, setIsEnabled] = useState(false);

  return (
    <ToggleSwitch
      checked={isEnabled}
      onChange={setIsEnabled}
      label="Dark mode"
    />
  );
}

// With async onChange
<ToggleSwitch
  checked={settings.notifications}
  onChange={async (checked) => {
    await updateSettings({ notifications: checked });
  }}
  loading={isUpdating}
  label="Email notifications"
/>
```

## Test Cases (Vitest)

```jsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import ToggleSwitch from './ToggleSwitch';

describe('ToggleSwitch', () => {
  it('renders toggle switch', () => {
    render(<ToggleSwitch label="Test Toggle" />);
    expect(screen.getByRole('switch')).toBeInTheDocument();
    expect(screen.getByText('Test Toggle')).toBeInTheDocument();
  });

  it('starts unchecked by default', () => {
    render(<ToggleSwitch />);
    const toggle = screen.getByRole('switch');
    expect(toggle).toHaveAttribute('aria-checked', 'false');
  });

  it('respects defaultChecked prop', () => {
    render(<ToggleSwitch defaultChecked={true} />);
    const toggle = screen.getByRole('switch');
    expect(toggle).toHaveAttribute('aria-checked', 'true');
  });

  it('toggles on click', async () => {
    const user = userEvent.setup();
    render(<ToggleSwitch />);
    
    const toggle = screen.getByRole('switch');
    expect(toggle).toHaveAttribute('aria-checked', 'false');
    
    await user.click(toggle);
    expect(toggle).toHaveAttribute('aria-checked', 'true');
    
    await user.click(toggle);
    expect(toggle).toHaveAttribute('aria-checked', 'false');
  });

  it('calls onChange callback', async () => {
    const user = userEvent.setup();
    const handleChange = vi.fn();
    render(<ToggleSwitch onChange={handleChange} />);
    
    await user.click(screen.getByRole('switch'));
    
    expect(handleChange).toHaveBeenCalledWith(true);
  });

  it('works in controlled mode', async () => {
    const user = userEvent.setup();
    const handleChange = vi.fn();
    const { rerender } = render(
      <ToggleSwitch checked={false} onChange={handleChange} />
    );
    
    const toggle = screen.getByRole('switch');
    expect(toggle).toHaveAttribute('aria-checked', 'false');
    
    await user.click(toggle);
    expect(handleChange).toHaveBeenCalledWith(true);
    
    // Simulate parent updating state
    rerender(<ToggleSwitch checked={true} onChange={handleChange} />);
    expect(toggle).toHaveAttribute('aria-checked', 'true');
  });

  it('does not toggle when disabled', async () => {
    const user = userEvent.setup();
    const handleChange = vi.fn();
    render(<ToggleSwitch disabled onChange={handleChange} />);
    
    const toggle = screen.getByRole('switch');
    await user.click(toggle);
    
    expect(handleChange).not.toHaveBeenCalled();
    expect(toggle).toBeDisabled();
  });

  it('does not toggle when loading', async () => {
    const user = userEvent.setup();
    const handleChange = vi.fn();
    render(<ToggleSwitch loading onChange={handleChange} />);
    
    await user.click(screen.getByRole('switch'));
    
    expect(handleChange).not.toHaveBeenCalled();
  });

  it('shows loading spinner', () => {
    render(<ToggleSwitch loading />);
    expect(document.querySelector('.toggle-spinner')).toBeInTheDocument();
  });

  it('toggles with Space key', async () => {
    const user = userEvent.setup();
    render(<ToggleSwitch />);
    
    const toggle = screen.getByRole('switch');
    toggle.focus();
    
    await user.keyboard(' ');
    expect(toggle).toHaveAttribute('aria-checked', 'true');
  });

  it('toggles with Enter key', async () => {
    const user = userEvent.setup();
    render(<ToggleSwitch />);
    
    const toggle = screen.getByRole('switch');
    toggle.focus();
    
    await user.keyboard('{Enter}');
    expect(toggle).toHaveAttribute('aria-checked', 'true');
  });

  it('applies correct size class', () => {
    const { rerender } = render(<ToggleSwitch size="small" />);
    expect(screen.getByRole('switch')).toHaveClass('toggle-small');
    
    rerender(<ToggleSwitch size="large" />);
    expect(screen.getByRole('switch')).toHaveClass('toggle-large');
  });

  it('shows on/off labels when enabled', () => {
    render(<ToggleSwitch showLabels onLabel="YES" offLabel="NO" />);
    expect(screen.getByText('NO')).toBeInTheDocument();
  });

  it('updates label when toggled', async () => {
    const user = userEvent.setup();
    render(<ToggleSwitch showLabels onLabel="YES" offLabel="NO" />);
    
    expect(screen.getByText('NO')).toBeInTheDocument();
    
    await user.click(screen.getByRole('switch'));
    
    expect(screen.getByText('YES')).toBeInTheDocument();
    expect(screen.queryByText('NO')).not.toBeInTheDocument();
  });

  it('handles async onChange', async () => {
    const user = userEvent.setup();
    const asyncChange = vi.fn().mockResolvedValue(undefined);
    render(<ToggleSwitch onChange={asyncChange} />);
    
    await user.click(screen.getByRole('switch'));
    
    await waitFor(() => {
      expect(asyncChange).toHaveBeenCalledWith(true);
    });
  });

  it('has correct ARIA attributes', () => {
    render(<ToggleSwitch label="Test" />);
    
    const toggle = screen.getByRole('switch');
    expect(toggle).toHaveAttribute('aria-checked');
    expect(toggle).toHaveAttribute('aria-label');
  });

  it('associates label with toggle', () => {
    render(<ToggleSwitch label="Test Toggle" id="test-toggle" />);
    
    const label = screen.getByText('Test Toggle');
    const toggle = screen.getByRole('switch');
    
    expect(label).toHaveAttribute('for', 'test-toggle');
    expect(toggle).toHaveAttribute('id', 'test-toggle');
  });

  it('applies disabled class to container', () => {
    const { container } = render(<ToggleSwitch disabled />);
    expect(container.querySelector('.toggle-container')).toHaveClass('disabled');
  });

  it('applies checked class when checked', () => {
    render(<ToggleSwitch checked={true} />);
    expect(screen.getByRole('switch')).toHaveClass('checked');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add confirmation dialog for critical toggles?**
```jsx
function ToggleWithConfirmation({ onConfirm, confirmMessage, ...props }) {
  const [showConfirm, setShowConfirm] = useState(false);
  const [pendingValue, setPendingValue] = useState(null);

  const handleChange = (newValue) => {
    if (onConfirm && newValue) {
      setPendingValue(newValue);
      setShowConfirm(true);
    } else {
      props.onChange?.(newValue);
    }
  };

  const handleConfirm = () => {
    props.onChange?.(pendingValue);
    setShowConfirm(false);
  };

  return (
    <>
      <ToggleSwitch {...props} onChange={handleChange} />
      {showConfirm && (
        <ConfirmDialog
          message={confirmMessage}
          onConfirm={handleConfirm}
          onCancel={() => setShowConfirm(false)}
        />
      )}
    </>
  );
}
```

**Q: How would you add custom icons?**
```jsx
function ToggleSwitch({ checkedIcon, uncheckedIcon, ...props }) {
  return (
    <button className="toggle-switch">
      <span className="toggle-thumb">
        {checked ? checkedIcon : uncheckedIcon}
      </span>
    </button>
  );
}

// Usage
<ToggleSwitch
  checkedIcon={<CheckIcon />}
  uncheckedIcon={<XIcon />}
/>
```

**Q: How would you add color customization?**
```jsx
function ToggleSwitch({ 
  checkedColor = '#4caf50',
  uncheckedColor = '#ccc',
  ...props 
}) {
  return (
    <button
      className="toggle-switch"
      style={{
        '--checked-color': checkedColor,
        '--unchecked-color': uncheckedColor
      }}
    >
      {/* ... */}
    </button>
  );
}

// CSS
.toggle-track {
  background: var(--unchecked-color);
}

.toggle-switch.checked .toggle-track {
  background: var(--checked-color);
}
```

## Performance Considerations

1. **Avoid Unnecessary Re-renders**
   - Use controlled mode only when needed
   - Memoize onChange callbacks
   - Don't create inline functions in render

2. **Animation Performance**
   - Use CSS transforms (not left/right)
   - Use will-change for smoother animations
   - Avoid animating expensive properties

3. **Accessibility**
   - Always include ARIA attributes
   - Support keyboard navigation
   - Provide focus indicators

## Edge Cases Handled
- Controlled/uncontrolled modes
- Disabled state
- Loading state with spinner
- Keyboard support (Space, Enter)
- ARIA attributes for accessibility
- Custom sizes
- On/off labels
- Async onChange support
- Focus management
- Label association
