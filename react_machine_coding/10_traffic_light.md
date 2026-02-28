# 10. Traffic Light

## Problem Description
Build an auto-cycling Traffic Light component.
The light should automatically transition from Red (e.g., 4000ms) → Yellow (500ms) → Green (3000ms) and repeat. You should use `useEffect` and handle memory management properly to prevent memory leaks from intervals or timeouts.

**Key Concepts Tested:** `useState`, `useEffect`, `setInterval`/`setTimeout`, React lifecycle, cleanup functions.

## React Component Implementation Outline

```tsx
import React, { useState, useEffect } from 'react';
import './TrafficLight.css';

type LightColor = 'red' | 'yellow' | 'green';

interface LightConfig {
  color: LightColor;
  duration: number;
}

const LIGHTS: Record<LightColor, LightConfig> = {
  red: { color: 'red', duration: 4000 },
  yellow: { color: 'yellow', duration: 500 },
  green: { color: 'green', duration: 3000 },
};

export const TrafficLight: React.FC = () => {
  const [activeLight, setActiveLight] = useState<LightColor>('red');

  useEffect(() => {
    const timer = setTimeout(() => {
      setActiveLight((prev) => {
        if (prev === 'red') return 'green';
        if (prev === 'green') return 'yellow';
        return 'red';
      });
    }, LIGHTS[activeLight].duration);

    // Cleanup prevents memory leaks if component unmounts mid-timer
    return () => clearTimeout(timer); 
  }, [activeLight]);

  return (
    <div className="traffic-light-container">
      {Object.keys(LIGHTS).map((key) => {
        const color = key as LightColor;
        return (
          <div
            key={color}
            className={`light ${color} ${activeLight === color ? 'active' : 'inactive'}`}
            aria-label={`${color} light`}
          />
        );
      })}
    </div>
  );
};
```

## Vitest Unit Test Cases

```tsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { TrafficLight } from './TrafficLight';

describe('TrafficLight Component', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.runOnlyPendingTimers();
    vi.useRealTimers();
  });

  it('renders initial state correctly (Red)', () => {
    render(<TrafficLight />);
    expect(screen.getByLabelText('red light')).toHaveClass('active');
    expect(screen.getByLabelText('yellow light')).toHaveClass('inactive');
    expect(screen.getByLabelText('green light')).toHaveClass('inactive');
  });

  it('cycles from red to green after correct duration', () => {
    render(<TrafficLight />);
    
    // Advance time by 4000ms (red duration)
    vi.advanceTimersByTime(4000);
    expect(screen.getByLabelText('green light')).toHaveClass('active');
    expect(screen.getByLabelText('red light')).toHaveClass('inactive');
  });

  it('cycles from green to yellow, then to red', () => {
    render(<TrafficLight />);
    
    // Red (4000ms) -> Green (3000ms) -> Yellow (500ms) -> Red
    vi.advanceTimersByTime(4000); // Now Green
    expect(screen.getByLabelText('green light')).toHaveClass('active');
    
    vi.advanceTimersByTime(3000); // Now Yellow
    expect(screen.getByLabelText('yellow light')).toHaveClass('active');
    
    vi.advanceTimersByTime(500); // Back to Red
    expect(screen.getByLabelText('red light')).toHaveClass('active');
  });
});
```

## MANG-Level Cross Questions

1. **Intervals vs Timeouts:** You used `setTimeout` triggered on state change instead of a single `setInterval`. Why? What potential bugs occur if you use `setInterval` with heavily-updating state?
2. **Configuration Driven UI:** Suppose we want to add an override that forces the light to blink Yellow indefinitely when there's an emergency. How would you architect this state machine to break the usual cycle and enter an emergency mode?
3. **Data Fetching Consistency:** If the durations need to be fetched via an API on mount, how do you handle the component state while the data is loading?
4. **State Machine Scalability:** In a real-world scenario (e.g., an intersection with pedestrian crossings and left-turn signals), the transitioning logic gets incredibly complex. How would you refactor this using XState or a powerful `useReducer` to enforce strict transitions?
5. **Component Unmount:** What specifically happens at the memory/React level if the component is unmounted exactly when the timeout fires, assuming the cleanup function *wasn't* implemented? How does React 18 strict mode handle this in development?
