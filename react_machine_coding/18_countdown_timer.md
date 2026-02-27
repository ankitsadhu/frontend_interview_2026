# Countdown Timer

## Problem
Create a countdown timer with start, pause, resume, reset, and MM:SS format.

## Key Concepts
- `useState` for time state
- `useRef` for interval ID
- `useEffect` cleanup

## Solution

```jsx
import { useState, useRef, useEffect } from 'react';

function CountdownTimer({ initialMinutes = 5 }) {
  const [timeLeft, setTimeLeft] = useState(initialMinutes * 60);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning && timeLeft > 0) {
      intervalRef.current = setInterval(() => {
        setTimeLeft(prev => {
          if (prev <= 1) {
            setIsRunning(false);
            return 0;
          }
          return prev - 1;
        });
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }

    return () => clearInterval(intervalRef.current);
  }, [isRunning, timeLeft]);

  const formatTime = (seconds) => {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${String(mins).padStart(2, '0')}:${String(secs).padStart(2, '0')}`;
  };

  const handleStart = () => setIsRunning(true);
  const handlePause = () => setIsRunning(false);
  const handleReset = () => {
    setIsRunning(false);
    setTimeLeft(initialMinutes * 60);
  };

  return (
    <div className="countdown-timer">
      <div className="display">{formatTime(timeLeft)}</div>
      <div className="controls">
        {!isRunning ? (
          <button onClick={handleStart} disabled={timeLeft === 0}>
            Start
          </button>
        ) : (
          <button onClick={handlePause}>Pause</button>
        )}
        <button onClick={handleReset}>Reset</button>
      </div>
      {timeLeft === 0 && <div className="alert">Time's up!</div>}
    </div>
  );
}

export default CountdownTimer;
```

## CSS

```css
.countdown-timer {
  text-align: center;
  padding: 2rem;
}

.display {
  font-size: 4rem;
  font-weight: bold;
  margin-bottom: 1rem;
  font-family: monospace;
}

.controls {
  display: flex;
  gap: 1rem;
  justify-content: center;
}

.controls button {
  padding: 0.5rem 1.5rem;
  font-size: 1rem;
  cursor: pointer;
}

.alert {
  margin-top: 1rem;
  color: #d32f2f;
  font-size: 1.5rem;
  font-weight: bold;
}
```

## Edge Cases Handled
- Proper cleanup with useRef
- Auto-stop at zero
- Disabled start when time is up
- Formatted time display


## MANG-Level Expectations
- **Timing Accuracy**: Use useRef for interval ID, proper cleanup
- **State Management**: Handle start/pause/reset correctly
- **UX**: Format time display, visual feedback, sound on completion
- **Performance**: Avoid memory leaks, cleanup intervals
- **Extensibility**: Custom durations, lap times, multiple timers

## Enhanced Solution with Comments

```jsx
/**
 * TRICKY PART #1: useRef for interval ID
 * - MANG expectation: Explain why useRef instead of useState
 * - Answer: Changing interval ID shouldn't trigger re-render
 * - Also prevents stale closure issues
 */
const intervalRef = useRef(null);

/**
 * TRICKY PART #2: Cleanup in useEffect
 * - CRITICAL: Always clear interval on unmount or when dependencies change
 * - Memory leak if not cleaned up properly
 */
useEffect(() => {
  if (isRunning && timeLeft > 0) {
    intervalRef.current = setInterval(() => {
      setTimeLeft(prev => {
        if (prev <= 1) {
          setIsRunning(false);
          return 0;
        }
        return prev - 1;
      });
    }, 1000);
  } else {
    clearInterval(intervalRef.current);
  }

  return () => clearInterval(intervalRef.current); // CRITICAL cleanup
}, [isRunning, timeLeft]);

/**
 * TRICKY PART #3: Functional state update
 * - Use prev => prev - 1 instead of timeLeft - 1
 * - Avoids stale closure issues
 * - MANG will ask: "Why functional update?"
 */
```

## Test Cases

```jsx
import { render, screen, fireEvent, act } from '@testing-library/react';
import CountdownTimer from './CountdownTimer';

jest.useFakeTimers();

describe('CountdownTimer', () => {
  afterEach(() => {
    jest.clearAllTimers();
  });

  test('renders with initial time', () => {
    render(<CountdownTimer initialMinutes={5} />);
    expect(screen.getByText('05:00')).toBeInTheDocument();
  });

  test('starts countdown on start button', () => {
    render(<CountdownTimer initialMinutes={1} />);
    
    fireEvent.click(screen.getByText('Start'));
    
    act(() => {
      jest.advanceTimersByTime(1000);
    });
    
    expect(screen.getByText('00:59')).toBeInTheDocument();
  });

  test('pauses countdown', () => {
    render(<CountdownTimer initialMinutes={1} />);
    
    fireEvent.click(screen.getByText('Start'));
    
    act(() => {
      jest.advanceTimersByTime(2000);
    });
    
    fireEvent.click(screen.getByText('Pause'));
    
    const timeBeforePause = screen.getByText('00:58').textContent;
    
    act(() => {
      jest.advanceTimersByTime(5000);
    });
    
    expect(screen.getByText(timeBeforePause)).toBeInTheDocument();
  });

  test('resumes countdown after pause', () => {
    render(<CountdownTimer initialMinutes={1} />);
    
    fireEvent.click(screen.getByText('Start'));
    act(() => jest.advanceTimersByTime(1000));
    
    fireEvent.click(screen.getByText('Pause'));
    fireEvent.click(screen.getByText('Start'));
    
    act(() => jest.advanceTimersByTime(1000));
    
    expect(screen.getByText('00:58')).toBeInTheDocument();
  });

  test('resets to initial time', () => {
    render(<CountdownTimer initialMinutes={5} />);
    
    fireEvent.click(screen.getByText('Start'));
    act(() => jest.advanceTimersByTime(10000));
    
    fireEvent.click(screen.getByText('Reset'));
    
    expect(screen.getByText('05:00')).toBeInTheDocument();
  });

  test('stops at zero', () => {
    render(<CountdownTimer initialMinutes={0} />);
    
    // Set to 2 seconds manually for testing
    const { rerender } = render(<CountdownTimer initialMinutes={0} />);
    
    // Simulate countdown to zero
    fireEvent.click(screen.getByText('Start'));
    
    act(() => {
      jest.advanceTimersByTime(3000);
    });
    
    expect(screen.getByText('00:00')).toBeInTheDocument();
  });

  test('shows alert when time is up', () => {
    render(<CountdownTimer initialMinutes={0} />);
    expect(screen.getByText("Time's up!")).toBeInTheDocument();
  });

  test('disables start button when time is zero', () => {
    render(<CountdownTimer initialMinutes={0} />);
    expect(screen.getByText('Start')).toBeDisabled();
  });

  test('formats time correctly', () => {
    render(<CountdownTimer initialMinutes={1} />);
    expect(screen.getByText('01:00')).toBeInTheDocument();
    
    fireEvent.click(screen.getByText('Start'));
    act(() => jest.advanceTimersByTime(30000));
    
    expect(screen.getByText('00:30')).toBeInTheDocument();
  });

  test('cleans up interval on unmount', () => {
    const { unmount } = render(<CountdownTimer initialMinutes={5} />);
    
    fireEvent.click(screen.getByText('Start'));
    
    const clearIntervalSpy = jest.spyOn(global, 'clearInterval');
    unmount();
    
    expect(clearIntervalSpy).toHaveBeenCalled();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add lap/split times?**
```jsx
const [laps, setLaps] = useState([]);

const handleLap = () => {
  setLaps(prev => [...prev, timeLeft]);
};

<button onClick={handleLap} disabled={!isRunning}>
  Lap
</button>

<ul>
  {laps.map((lap, i) => (
    <li key={i}>Lap {i + 1}: {formatTime(lap)}</li>
  ))}
</ul>
```

**Q: How would you add sound/notification on completion?**
```jsx
useEffect(() => {
  if (timeLeft === 0 && isRunning) {
    // Play sound
    const audio = new Audio('/notification.mp3');
    audio.play();
    
    // Browser notification
    if ('Notification' in window && Notification.permission === 'granted') {
      new Notification('Timer Complete!', {
        body: 'Your countdown has finished',
        icon: '/timer-icon.png'
      });
    }
  }
}, [timeLeft, isRunning]);
```

**Q: How would you improve timing accuracy?**
```jsx
// Problem: setInterval can drift over time
// Solution: Use Date.now() to calculate actual elapsed time

const startTimeRef = useRef(null);
const targetTimeRef = useRef(null);

const start = () => {
  startTimeRef.current = Date.now();
  targetTimeRef.current = Date.now() + (timeLeft * 1000);
  setIsRunning(true);
};

useEffect(() => {
  if (!isRunning) return;
  
  const interval = setInterval(() => {
    const now = Date.now();
    const remaining = Math.max(0, Math.ceil((targetTimeRef.current - now) / 1000));
    
    setTimeLeft(remaining);
    
    if (remaining === 0) {
      setIsRunning(false);
    }
  }, 100); // Check more frequently for accuracy
  
  return () => clearInterval(interval);
}, [isRunning]);
```

**Q: How would you add custom time input?**
```jsx
const [inputMinutes, setInputMinutes] = useState(5);
const [inputSeconds, setInputSeconds] = useState(0);

<input
  type="number"
  min="0"
  max="59"
  value={inputMinutes}
  onChange={(e) => setInputMinutes(Number(e.target.value))}
/>
<input
  type="number"
  min="0"
  max="59"
  value={inputSeconds}
  onChange={(e) => setInputSeconds(Number(e.target.value))}
/>
<button onClick={() => setTimeLeft(inputMinutes * 60 + inputSeconds)}>
  Set Time
</button>
```

**Q: How would you handle multiple timers?**
```jsx
// Use array of timer objects
const [timers, setTimers] = useState([]);

const addTimer = () => {
  setTimers(prev => [...prev, {
    id: Date.now(),
    timeLeft: 300,
    isRunning: false
  }]);
};

// Render each timer with its own state
{timers.map(timer => (
  <CountdownTimer
    key={timer.id}
    initialTime={timer.timeLeft}
    onComplete={() => handleTimerComplete(timer.id)}
  />
))}
```

## Performance Considerations

1. **Interval Cleanup**
   - Always clear intervals in useEffect cleanup
   - Use useRef to store interval ID
   - Prevents memory leaks

2. **Timing Accuracy**
   - setInterval can drift (browser throttling, heavy tasks)
   - Use Date.now() for critical timing
   - Consider requestAnimationFrame for smoother updates

3. **Re-render Optimization**
   - Timer updates every second (acceptable)
   - For sub-second precision, consider throttling display updates
   - Use React.memo if timer is part of larger component

## Edge Cases Handled
- Proper cleanup with useRef
- Auto-stop at zero
- Disabled start when time is up
- Formatted time display (MM:SS)
- Functional state updates to avoid stale closures
- Pause/resume functionality
- Reset to initial value
