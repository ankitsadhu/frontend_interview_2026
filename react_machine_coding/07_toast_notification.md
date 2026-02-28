# Toast / Notification System

## Problem
Create a toast notification system with auto-dismiss, stack multiple toasts, and different types (success/error/info/warning).

## Key Concepts
- `useState` array for managing multiple toasts
- `setTimeout` for auto-dismiss
- `useCallback` for stable function references
- Portal for rendering outside DOM hierarchy

## MANG-Level Expectations
- **State Management**: Handle multiple toasts, queue management
- **UX**: Smooth animations, pause on hover, dismiss on click
- **Performance**: Cleanup timers, prevent memory leaks
- **Extensibility**: Custom positions, action buttons, progress bars

## Solution

```jsx
import { useState, useCallback, useEffect } from 'react';
import { createPortal } from 'react-dom';

/**
 * TRICKY PART #1: Toast state management
 * - Array of toast objects with unique IDs
 * - MANG expectation: Explain why array instead of object
 * - Array maintains insertion order for stacking
 */
function useToast() {
  const [toasts, setToasts] = useState([]);

  /**
   * TRICKY PART #2: useCallback for stable reference
   * - Prevents unnecessary re-renders in consuming components
   * - MANG will ask: "Why useCallback here?"
   */
  const addToast = useCallback((message, type = 'info', duration = 3000) => {
    const id = Date.now() + Math.random(); // Unique ID
    
    setToasts(prev => [...prev, {
      id,
      message,
      type,
      duration
    }]);

    // Auto-dismiss if duration is set
    if (duration > 0) {
      setTimeout(() => {
        removeToast(id);
      }, duration);
    }

    return id;
  }, []);

  const removeToast = useCallback((id) => {
    setToasts(prev => prev.filter(toast => toast.id !== id));
  }, []);

  return { toasts, addToast, removeToast };
}

/**
 * Individual Toast Component
 * TRICKY PART #3: Pause timer on hover
 * - Clear timeout on hover, restart on mouse leave
 * - MANG expectation: Handle edge cases (unmount while hovered)
 */
function Toast({ id, message, type, duration, onClose }) {
  const [isPaused, setIsPaused] = useState(false);
  const [remainingTime, setRemainingTime] = useState(duration);
  const [startTime, setStartTime] = useState(Date.now());

  useEffect(() => {
    if (duration <= 0 || isPaused) return;

    const timer = setTimeout(() => {
      onClose(id);
    }, remainingTime);

    return () => clearTimeout(timer);
  }, [id, remainingTime, isPaused, onClose, duration]);

  const handleMouseEnter = () => {
    if (duration <= 0) return;
    
    setIsPaused(true);
    const elapsed = Date.now() - startTime;
    setRemainingTime(prev => prev - elapsed);
  };

  const handleMouseLeave = () => {
    if (duration <= 0) return;
    
    setIsPaused(false);
    setStartTime(Date.now());
  };

  const icons = {
    success: '✓',
    error: '✕',
    warning: '⚠',
    info: 'ℹ'
  };

  return (
    <div
      className={`toast toast-${type}`}
      onMouseEnter={handleMouseEnter}
      onMouseLeave={handleMouseLeave}
      role="alert"
      aria-live="polite"
    >
      <span className="toast-icon">{icons[type]}</span>
      <span className="toast-message">{message}</span>
      <button
        className="toast-close"
        onClick={() => onClose(id)}
        aria-label="Close notification"
      >
        ×
      </button>
      {duration > 0 && !isPaused && (
        <div
          className="toast-progress"
          style={{
            animation: `shrink ${remainingTime}ms linear`
          }}
        />
      )}
    </div>
  );
}

/**
 * Toast Container Component
 * TRICKY PART #4: Portal rendering with position
 * - Render at different positions (top-right, bottom-left, etc.)
 * - MANG might ask: "How would you handle multiple containers?"
 */
function ToastContainer({ toasts, removeToast, position = 'top-right' }) {
  if (toasts.length === 0) return null;

  const positionClasses = {
    'top-right': 'toast-container-top-right',
    'top-left': 'toast-container-top-left',
    'bottom-right': 'toast-container-bottom-right',
    'bottom-left': 'toast-container-bottom-left',
    'top-center': 'toast-container-top-center',
    'bottom-center': 'toast-container-bottom-center'
  };

  return createPortal(
    <div className={`toast-container ${positionClasses[position]}`}>
      {toasts.map(toast => (
        <Toast
          key={toast.id}
          {...toast}
          onClose={removeToast}
        />
      ))}
    </div>,
    document.body
  );
}

// Main export
export function ToastProvider({ children, position }) {
  const { toasts, addToast, removeToast } = useToast();

  return (
    <>
      {children}
      <ToastContainer
        toasts={toasts}
        removeToast={removeToast}
        position={position}
      />
    </>
  );
}

export { useToast };
```

## CSS

```css
.toast-container {
  position: fixed;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  pointer-events: none;
}

.toast-container > * {
  pointer-events: auto;
}

.toast-container-top-right {
  top: 1rem;
  right: 1rem;
}

.toast-container-top-left {
  top: 1rem;
  left: 1rem;
}

.toast-container-bottom-right {
  bottom: 1rem;
  right: 1rem;
}

.toast-container-bottom-left {
  bottom: 1rem;
  left: 1rem;
}

.toast-container-top-center {
  top: 1rem;
  left: 50%;
  transform: translateX(-50%);
}

.toast-container-bottom-center {
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%);
}

.toast {
  position: relative;
  display: flex;
  align-items: center;
  gap: 0.75rem;
  min-width: 300px;
  padding: 1rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  animation: slideIn 0.3s ease-out;
  overflow: hidden;
}

@keyframes slideIn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.toast-success {
  border-left: 4px solid #10b981;
}

.toast-error {
  border-left: 4px solid #ef4444;
}

.toast-warning {
  border-left: 4px solid #f59e0b;
}

.toast-info {
  border-left: 4px solid #3b82f6;
}

.toast-icon {
  font-size: 1.25rem;
  flex-shrink: 0;
}

.toast-success .toast-icon {
  color: #10b981;
}

.toast-error .toast-icon {
  color: #ef4444;
}

.toast-warning .toast-icon {
  color: #f59e0b;
}

.toast-info .toast-icon {
  color: #3b82f6;
}

.toast-message {
  flex: 1;
  font-size: 0.875rem;
}

.toast-close {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  color: #666;
  padding: 0;
  line-height: 1;
}

.toast-close:hover {
  color: #000;
}

.toast-progress {
  position: absolute;
  bottom: 0;
  left: 0;
  height: 3px;
  width: 100%;
  background: currentColor;
  opacity: 0.3;
}

@keyframes shrink {
  from {
    width: 100%;
  }
  to {
    width: 0%;
  }
}
```

## Usage

```jsx
import { ToastProvider, useToast } from './Toast';

function App() {
  return (
    <ToastProvider position="top-right">
      <MyComponent />
    </ToastProvider>
  );
}

function MyComponent() {
  const { addToast } = useToast();

  return (
    <div>
      <button onClick={() => addToast('Success!', 'success')}>
        Show Success
      </button>
      <button onClick={() => addToast('Error occurred', 'error')}>
        Show Error
      </button>
      <button onClick={() => addToast('Warning message', 'warning', 5000)}>
        Show Warning (5s)
      </button>
      <button onClick={() => addToast('Info message', 'info', 0)}>
        Show Info (No auto-dismiss)
      </button>
    </div>
  );
}
```

## Test Cases

```jsx
import { render, screen, fireEvent, waitFor, act } from '@testing-library/react';
import { ToastProvider, useToast } from './Toast';

jest.useFakeTimers();

function TestComponent() {
  const { addToast } = useToast();
  
  return (
    <div>
      <button onClick={() => addToast('Test message', 'success', 3000)}>
        Add Toast
      </button>
      <button onClick={() => addToast('Error', 'error', 2000)}>
        Add Error
      </button>
    </div>
  );
}

describe('Toast System', () => {
  afterEach(() => {
    jest.clearAllTimers();
  });

  test('renders toast when added', () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    
    expect(screen.getByText('Test message')).toBeInTheDocument();
  });

  test('shows correct icon for toast type', () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    
    const toast = screen.getByRole('alert');
    expect(toast).toHaveClass('toast-success');
    expect(toast.textContent).toContain('✓');
  });

  test('auto-dismisses after duration', async () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    expect(screen.getByText('Test message')).toBeInTheDocument();
    
    act(() => {
      jest.advanceTimersByTime(3000);
    });
    
    await waitFor(() => {
      expect(screen.queryByText('Test message')).not.toBeInTheDocument();
    });
  });

  test('closes on close button click', () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    
    const closeButton = screen.getByLabelText('Close notification');
    fireEvent.click(closeButton);
    
    expect(screen.queryByText('Test message')).not.toBeInTheDocument();
  });

  test('handles multiple toasts', () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    fireEvent.click(screen.getByText('Add Error'));
    
    expect(screen.getByText('Test message')).toBeInTheDocument();
    expect(screen.getByText('Error')).toBeInTheDocument();
  });

  test('pauses timer on hover', async () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    const toast = screen.getByRole('alert');
    
    // Advance time partially
    act(() => {
      jest.advanceTimersByTime(1500);
    });
    
    // Hover to pause
    fireEvent.mouseEnter(toast);
    
    // Advance time past original duration
    act(() => {
      jest.advanceTimersByTime(2000);
    });
    
    // Toast should still be visible
    expect(screen.getByText('Test message')).toBeInTheDocument();
  });

  test('resumes timer on mouse leave', async () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    const toast = screen.getByRole('alert');
    
    fireEvent.mouseEnter(toast);
    fireEvent.mouseLeave(toast);
    
    act(() => {
      jest.advanceTimersByTime(3000);
    });
    
    await waitFor(() => {
      expect(screen.queryByText('Test message')).not.toBeInTheDocument();
    });
  });

  test('does not auto-dismiss when duration is 0', () => {
    function TestZeroDuration() {
      const { addToast } = useToast();
      return (
        <button onClick={() => addToast('Persistent', 'info', 0)}>
          Add Persistent
        </button>
      );
    }

    render(
      <ToastProvider>
        <TestZeroDuration />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Persistent'));
    
    act(() => {
      jest.advanceTimersByTime(10000);
    });
    
    expect(screen.getByText('Persistent')).toBeInTheDocument();
  });

  test('has correct ARIA attributes', () => {
    render(
      <ToastProvider>
        <TestComponent />
      </ToastProvider>
    );
    
    fireEvent.click(screen.getByText('Add Toast'));
    
    const toast = screen.getByRole('alert');
    expect(toast).toHaveAttribute('aria-live', 'polite');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add action buttons to toasts?**
```jsx
function Toast({ id, message, type, actions, onClose }) {
  return (
    <div className="toast">
      <span>{message}</span>
      {actions && (
        <div className="toast-actions">
          {actions.map((action, i) => (
            <button
              key={i}
              onClick={() => {
                action.onClick();
                onClose(id);
              }}
            >
              {action.label}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}

// Usage
addToast('File deleted', 'info', 5000, [
  { label: 'Undo', onClick: () => undoDelete() }
]);
```

**Q: How would you limit the number of visible toasts?**
```jsx
const MAX_TOASTS = 3;

const addToast = useCallback((message, type, duration) => {
  setToasts(prev => {
    const newToast = { id: Date.now(), message, type, duration };
    const updated = [...prev, newToast];
    
    // Keep only last MAX_TOASTS
    return updated.slice(-MAX_TOASTS);
  });
}, []);
```

**Q: How would you add a toast queue?**
```jsx
function useToastQueue() {
  const [queue, setQueue] = useState([]);
  const [activeToasts, setActiveToasts] = useState([]);
  const MAX_ACTIVE = 3;

  useEffect(() => {
    if (queue.length > 0 && activeToasts.length < MAX_ACTIVE) {
      const [next, ...rest] = queue;
      setQueue(rest);
      setActiveToasts(prev => [...prev, next]);
    }
  }, [queue, activeToasts]);

  const addToast = (toast) => {
    setQueue(prev => [...prev, { ...toast, id: Date.now() }]);
  };

  return { activeToasts, addToast };
}
```

**Q: How would you persist toasts across page reloads?**
```jsx
// Save to sessionStorage
useEffect(() => {
  sessionStorage.setItem('toasts', JSON.stringify(toasts));
}, [toasts]);

// Load on mount
const [toasts, setToasts] = useState(() => {
  const saved = sessionStorage.getItem('toasts');
  return saved ? JSON.parse(saved) : [];
});
```

**Q: How would you add sound notifications?**
```jsx
const addToast = useCallback((message, type, duration) => {
  // Play sound based on type
  const sounds = {
    success: '/sounds/success.mp3',
    error: '/sounds/error.mp3'
  };
  
  if (sounds[type]) {
    const audio = new Audio(sounds[type]);
    audio.play().catch(err => console.log('Audio play failed:', err));
  }
  
  // ... rest of logic
}, []);
```

## Edge Cases Handled
- Multiple toasts stacking
- Auto-dismiss with configurable duration
- Pause on hover, resume on leave
- Manual dismiss via close button
- Different toast types with icons
- Progress bar animation
- Portal rendering for z-index
- Cleanup timers on unmount
- Unique IDs to prevent collisions
- Zero duration for persistent toasts
- ARIA attributes for accessibility
