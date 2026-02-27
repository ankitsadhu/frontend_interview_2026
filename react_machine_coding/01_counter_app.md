# Counter App

## Problem
Create a counter with increment, decrement, reset buttons, and min/max limits.

## Key Concepts
- `useState` for state management
- Event handlers
- Conditional button disable

## Solution

```jsx
import { useState } from 'react';

function Counter({ min = 0, max = 10, initialValue = 0 }) {
  const [count, setCount] = useState(initialValue);

  const increment = () => setCount(prev => Math.min(prev + 1, max));
  const decrement = () => setCount(prev => Math.max(prev - 1, min));
  const reset = () => setCount(initialValue);

  return (
    <div className="counter">
      <h2>Counter: {count}</h2>
      <div className="controls">
        <button onClick={decrement} disabled={count <= min}>
          -
        </button>
        <button onClick={reset}>Reset</button>
        <button onClick={increment} disabled={count >= max}>
          +
        </button>
      </div>
      <p>Range: {min} to {max}</p>
    </div>
  );
}

export default Counter;
```

## CSS

```css
.counter {
  text-align: center;
  padding: 2rem;
}

.controls {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin: 1rem 0;
}

button {
  padding: 0.5rem 1rem;
  font-size: 1rem;
  cursor: pointer;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

## Edge Cases Handled
- Min/max boundary enforcement
- Disabled buttons at limits
- Configurable initial value and range


## MANG-Level Expectations
- **State Management**: Proper use of useState with functional updates
- **Boundary Handling**: Min/max enforcement, disabled states
- **UX**: Visual feedback, keyboard support
- **Extensibility**: Step size, custom increment values

## Enhanced Solution with Comments

```jsx
/**
 * TRICKY PART: Math.min/Math.max for boundary enforcement
 * - MANG expectation: Handle edge cases cleanly
 * - Alternative: Could use if statements, but this is more elegant
 */
const increment = () => setCount(prev => Math.min(prev + 1, max));
const decrement = () => setCount(prev => Math.max(prev - 1, min));

/**
 * EXTENSION: Add step size
 * const increment = () => setCount(prev => Math.min(prev + step, max));
 */
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from './Counter';

describe('Counter', () => {
  test('renders with initial value', () => {
    render(<Counter initialValue={5} />);
    expect(screen.getByText(/Counter: 5/)).toBeInTheDocument();
  });

  test('increments counter', () => {
    render(<Counter initialValue={0} max={10} />);
    const incrementBtn = screen.getByText('+');
    
    fireEvent.click(incrementBtn);
    expect(screen.getByText(/Counter: 1/)).toBeInTheDocument();
  });

  test('decrements counter', () => {
    render(<Counter initialValue={5} min={0} />);
    const decrementBtn = screen.getByText('-');
    
    fireEvent.click(decrementBtn);
    expect(screen.getByText(/Counter: 4/)).toBeInTheDocument();
  });

  test('resets to initial value', () => {
    render(<Counter initialValue={5} />);
    
    fireEvent.click(screen.getByText('+'));
    fireEvent.click(screen.getByText('Reset'));
    
    expect(screen.getByText(/Counter: 5/)).toBeInTheDocument();
  });

  test('disables increment at max', () => {
    render(<Counter initialValue={10} max={10} />);
    expect(screen.getByText('+')).toBeDisabled();
  });

  test('disables decrement at min', () => {
    render(<Counter initialValue={0} min={0} />);
    expect(screen.getByText('-')).toBeDisabled();
  });

  test('does not exceed max on increment', () => {
    render(<Counter initialValue={9} max={10} />);
    const incrementBtn = screen.getByText('+');
    
    fireEvent.click(incrementBtn);
    fireEvent.click(incrementBtn);
    
    expect(screen.getByText(/Counter: 10/)).toBeInTheDocument();
  });

  test('does not go below min on decrement', () => {
    render(<Counter initialValue={1} min={0} />);
    const decrementBtn = screen.getByText('-');
    
    fireEvent.click(decrementBtn);
    fireEvent.click(decrementBtn);
    
    expect(screen.getByText(/Counter: 0/)).toBeInTheDocument();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add keyboard support?**
```jsx
useEffect(() => {
  const handleKeyDown = (e) => {
    if (e.key === 'ArrowUp') increment();
    if (e.key === 'ArrowDown') decrement();
    if (e.key === 'r') reset();
  };
  
  window.addEventListener('keydown', handleKeyDown);
  return () => window.removeEventListener('keydown', handleKeyDown);
}, []);
```

**Q: How would you add step size?**
```jsx
function Counter({ min = 0, max = 10, initialValue = 0, step = 1 }) {
  const increment = () => setCount(prev => Math.min(prev + step, max));
  const decrement = () => setCount(prev => Math.max(prev - step, min));
}
```

**Q: How would you add animation?**
```jsx
// Use CSS transitions or react-spring
const [isIncrementing, setIsIncrementing] = useState(false);

const increment = () => {
  setIsIncrementing(true);
  setCount(prev => Math.min(prev + 1, max));
  setTimeout(() => setIsIncrementing(false), 300);
};

<h2 className={isIncrementing ? 'pulse' : ''}>Counter: {count}</h2>
```
