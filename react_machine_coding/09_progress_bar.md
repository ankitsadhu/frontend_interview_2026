# Progress Bar Component

## Problem
Create an animated progress bar with percentage label and smooth transitions.

## Key Concepts
- `useState` for progress value
- `useEffect` for animations
- CSS transitions for smooth fill
- `requestAnimationFrame` for custom animations

## MANG-Level Expectations
- **Animation**: Smooth transitions, easing functions, custom animations
- **Variants**: Linear, circular, stepped, indeterminate
- **UX**: Percentage label, color coding, accessibility
- **Performance**: Efficient animations, avoid layout thrashing
- **Extensibility**: Custom colors, sizes, labels, segments

## Solution

```jsx
import { useState, useEffect, useRef } from 'react';

/**
 * TRICKY PART #1: Animated progress
 * - Animate from current to target value
 * - MANG expectation: Smooth animation with easing
 * - Use CSS transition or requestAnimationFrame
 */
function ProgressBar({
  value = 0,
  max = 100,
  showLabel = true,
  animated = true,
  color,
  height = 20,
  striped = false,
  indeterminate = false
}) {
  const [displayValue, setDisplayValue] = useState(0);
  const animationRef = useRef(null);

  /**
   * TRICKY PART #2: Smooth animation with RAF
   * - Use requestAnimationFrame for 60fps animation
   * - MANG will ask: "Why RAF instead of CSS transition?"
   * - RAF gives more control, can add easing, callbacks
   */
  useEffect(() => {
    if (!animated || indeterminate) {
      setDisplayValue(value);
      return;
    }

    const startValue = displayValue;
    const endValue = Math.min(Math.max(value, 0), max);
    const duration = 500; // ms
    const startTime = performance.now();

    const animate = (currentTime) => {
      const elapsed = currentTime - startTime;
      const progress = Math.min(elapsed / duration, 1);

      // Easing function (ease-out)
      const easeOut = 1 - Math.pow(1 - progress, 3);
      const currentValue = startValue + (endValue - startValue) * easeOut;

      setDisplayValue(currentValue);

      if (progress < 1) {
        animationRef.current = requestAnimationFrame(animate);
      }
    };

    animationRef.current = requestAnimationFrame(animate);

    return () => {
      if (animationRef.current) {
        cancelAnimationFrame(animationRef.current);
      }
    };
  }, [value, max, animated, indeterminate]);

  const percentage = Math.round((displayValue / max) * 100);
  const clampedPercentage = Math.min(Math.max(percentage, 0), 100);

  /**
   * TRICKY PART #3: Color coding based on value
   * - Different colors for different ranges
   * - MANG expectation: Customizable thresholds
   */
  const getColor = () => {
    if (color) return color;
    
    if (percentage < 30) return '#d32f2f'; // Red
    if (percentage < 70) return '#f57c00'; // Orange
    return '#388e3c'; // Green
  };

  return (
    <div
      className="progress-bar-container"
      role="progressbar"
      aria-valuenow={clampedPercentage}
      aria-valuemin={0}
      aria-valuemax={100}
      aria-label={`Progress: ${clampedPercentage}%`}
    >
      <div
        className="progress-bar-track"
        style={{ height: `${height}px` }}
      >
        <div
          className={`progress-bar-fill ${striped ? 'striped' : ''} ${
            indeterminate ? 'indeterminate' : ''
          }`}
          style={{
            width: indeterminate ? '100%' : `${clampedPercentage}%`,
            backgroundColor: getColor(),
            transition: animated && !indeterminate ? 'width 0.3s ease' : 'none'
          }}
        >
          {showLabel && !indeterminate && (
            <span className="progress-bar-label">{clampedPercentage}%</span>
          )}
        </div>
      </div>
    </div>
  );
}

/**
 * Circular Progress Bar
 * TRICKY PART #4: SVG circle animation
 * - Use stroke-dasharray and stroke-dashoffset
 * - MANG will ask: "How do you animate SVG circles?"
 */
function CircularProgress({ value = 0, max = 100, size = 120, strokeWidth = 8 }) {
  const [displayValue, setDisplayValue] = useState(0);
  
  useEffect(() => {
    const timer = setTimeout(() => setDisplayValue(value), 100);
    return () => clearTimeout(timer);
  }, [value]);

  const percentage = Math.min(Math.max((displayValue / max) * 100, 0), 100);
  const radius = (size - strokeWidth) / 2;
  const circumference = 2 * Math.PI * radius;
  const offset = circumference - (percentage / 100) * circumference;

  return (
    <div className="circular-progress" style={{ width: size, height: size }}>
      <svg width={size} height={size}>
        {/* Background circle */}
        <circle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          fill="none"
          stroke="#e0e0e0"
          strokeWidth={strokeWidth}
        />
        {/* Progress circle */}
        <circle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          fill="none"
          stroke="#007bff"
          strokeWidth={strokeWidth}
          strokeDasharray={circumference}
          strokeDashoffset={offset}
          strokeLinecap="round"
          transform={`rotate(-90 ${size / 2} ${size / 2})`}
          style={{
            transition: 'stroke-dashoffset 0.5s ease'
          }}
        />
      </svg>
      <div className="circular-progress-label">
        {Math.round(percentage)}%
      </div>
    </div>
  );
}

export { ProgressBar, CircularProgress };
export default ProgressBar;
```

## CSS

```css
.progress-bar-container {
  width: 100%;
}

.progress-bar-track {
  width: 100%;
  background: #e0e0e0;
  border-radius: 10px;
  overflow: hidden;
  position: relative;
}

.progress-bar-fill {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
  position: relative;
  overflow: hidden;
}

.progress-bar-label {
  color: white;
  font-size: 0.75rem;
  font-weight: bold;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
  z-index: 1;
}

/* Striped pattern */
.progress-bar-fill.striped {
  background-image: linear-gradient(
    45deg,
    rgba(255, 255, 255, 0.15) 25%,
    transparent 25%,
    transparent 50%,
    rgba(255, 255, 255, 0.15) 50%,
    rgba(255, 255, 255, 0.15) 75%,
    transparent 75%,
    transparent
  );
  background-size: 20px 20px;
  animation: stripeAnimation 1s linear infinite;
}

@keyframes stripeAnimation {
  0% {
    background-position: 0 0;
  }
  100% {
    background-position: 20px 0;
  }
}

/* Indeterminate animation */
.progress-bar-fill.indeterminate {
  background: linear-gradient(
    90deg,
    transparent,
    currentColor,
    transparent
  );
  animation: indeterminateAnimation 1.5s ease-in-out infinite;
}

@keyframes indeterminateAnimation {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

/* Circular Progress */
.circular-progress {
  position: relative;
  display: inline-block;
}

.circular-progress-label {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 1.25rem;
  font-weight: bold;
  color: #333;
}

/* Pulse animation for loading */
@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

.progress-bar-fill.pulse {
  animation: pulse 1.5s ease-in-out infinite;
}
```

## Usage

```jsx
// Basic linear progress
<ProgressBar value={75} />

// Custom color and height
<ProgressBar value={50} color="#9c27b0" height={30} />

// Striped animated
<ProgressBar value={60} striped animated />

// Indeterminate (loading)
<ProgressBar indeterminate />

// Circular progress
<CircularProgress value={75} size={150} strokeWidth={10} />

// Animated progress example
function AnimatedProgressExample() {
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setProgress(prev => {
        if (prev >= 100) return 0;
        return prev + 10;
      });
    }, 500);

    return () => clearInterval(timer);
  }, []);

  return <ProgressBar value={progress} />;
}
```

## Test Cases

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import { ProgressBar, CircularProgress } from './ProgressBar';

describe('ProgressBar', () => {
  test('renders progress bar', () => {
    render(<ProgressBar value={50} />);
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });

  test('displays correct percentage', () => {
    render(<ProgressBar value={75} max={100} />);
    expect(screen.getByText('75%')).toBeInTheDocument();
  });

  test('clamps value to max', () => {
    render(<ProgressBar value={150} max={100} />);
    expect(screen.getByText('100%')).toBeInTheDocument();
  });

  test('clamps negative values to 0', () => {
    render(<ProgressBar value={-10} />);
    expect(screen.getByText('0%')).toBeInTheDocument();
  });

  test('hides label when showLabel is false', () => {
    render(<ProgressBar value={50} showLabel={false} />);
    expect(screen.queryByText('50%')).not.toBeInTheDocument();
  });

  test('applies custom color', () => {
    render(<ProgressBar value={50} color="#ff0000" />);
    const fill = document.querySelector('.progress-bar-fill');
    expect(fill).toHaveStyle({ backgroundColor: '#ff0000' });
  });

  test('applies custom height', () => {
    render(<ProgressBar value={50} height={40} />);
    const track = document.querySelector('.progress-bar-track');
    expect(track).toHaveStyle({ height: '40px' });
  });

  test('adds striped class when striped prop is true', () => {
    render(<ProgressBar value={50} striped />);
    const fill = document.querySelector('.progress-bar-fill');
    expect(fill).toHaveClass('striped');
  });

  test('adds indeterminate class when indeterminate', () => {
    render(<ProgressBar indeterminate />);
    const fill = document.querySelector('.progress-bar-fill');
    expect(fill).toHaveClass('indeterminate');
  });

  test('has correct ARIA attributes', () => {
    render(<ProgressBar value={60} />);
    const progressBar = screen.getByRole('progressbar');
    
    expect(progressBar).toHaveAttribute('aria-valuenow', '60');
    expect(progressBar).toHaveAttribute('aria-valuemin', '0');
    expect(progressBar).toHaveAttribute('aria-valuemax', '100');
    expect(progressBar).toHaveAttribute('aria-label', 'Progress: 60%');
  });

  test('animates value change', async () => {
    const { rerender } = render(<ProgressBar value={0} animated />);
    
    rerender(<ProgressBar value={100} animated />);
    
    await waitFor(() => {
      expect(screen.getByText('100%')).toBeInTheDocument();
    }, { timeout: 1000 });
  });

  test('uses default color based on percentage', () => {
    const { rerender } = render(<ProgressBar value={20} />);
    let fill = document.querySelector('.progress-bar-fill');
    expect(fill).toHaveStyle({ backgroundColor: '#d32f2f' }); // Red

    rerender(<ProgressBar value={50} />);
    fill = document.querySelector('.progress-bar-fill');
    expect(fill).toHaveStyle({ backgroundColor: '#f57c00' }); // Orange

    rerender(<ProgressBar value={80} />);
    fill = document.querySelector('.progress-bar-fill');
    expect(fill).toHaveStyle({ backgroundColor: '#388e3c' }); // Green
  });
});

describe('CircularProgress', () => {
  test('renders circular progress', () => {
    render(<CircularProgress value={50} />);
    expect(screen.getByText('50%')).toBeInTheDocument();
  });

  test('renders SVG circles', () => {
    const { container } = render(<CircularProgress value={50} />);
    const circles = container.querySelectorAll('circle');
    expect(circles).toHaveLength(2); // Background + progress
  });

  test('applies custom size', () => {
    const { container } = render(<CircularProgress value={50} size={200} />);
    const svg = container.querySelector('svg');
    expect(svg).toHaveAttribute('width', '200');
    expect(svg).toHaveAttribute('height', '200');
  });

  test('calculates correct stroke-dashoffset', () => {
    const { container } = render(<CircularProgress value={50} size={120} strokeWidth={8} />);
    const progressCircle = container.querySelectorAll('circle')[1];
    
    // For 50%, offset should be half of circumference
    const radius = (120 - 8) / 2;
    const circumference = 2 * Math.PI * radius;
    const expectedOffset = circumference / 2;
    
    expect(progressCircle).toHaveAttribute('stroke-dashoffset', expectedOffset.toString());
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add segments/steps?**
```jsx
function SegmentedProgress({ value, segments = 5 }) {
  const segmentWidth = 100 / segments;
  const activeSegments = Math.ceil((value / 100) * segments);

  return (
    <div className="segmented-progress">
      {Array.from({ length: segments }, (_, i) => (
        <div
          key={i}
          className={`segment ${i < activeSegments ? 'active' : ''}`}
          style={{ width: `${segmentWidth}%` }}
        />
      ))}
    </div>
  );
}
```

**Q: How would you add a buffer indicator (like YouTube)?**
```jsx
function BufferedProgress({ value, buffered }) {
  return (
    <div className="progress-bar-track">
      <div
        className="progress-bar-buffer"
        style={{ width: `${buffered}%` }}
      />
      <div
        className="progress-bar-fill"
        style={{ width: `${value}%` }}
      />
    </div>
  );
}

// CSS
.progress-bar-buffer {
  position: absolute;
  height: 100%;
  background: rgba(255, 255, 255, 0.3);
}
```

**Q: How would you add milestone markers?**
```jsx
function ProgressWithMilestones({ value, milestones = [25, 50, 75] }) {
  return (
    <div className="progress-container">
      <ProgressBar value={value} />
      <div className="milestones">
        {milestones.map(milestone => (
          <div
            key={milestone}
            className={`milestone ${value >= milestone ? 'reached' : ''}`}
            style={{ left: `${milestone}%` }}
          >
            {milestone}%
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Q: How would you add custom easing functions?**
```jsx
const easingFunctions = {
  linear: t => t,
  easeIn: t => t * t,
  easeOut: t => 1 - Math.pow(1 - t, 3),
  easeInOut: t => t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2
};

function ProgressBar({ value, easing = 'easeOut' }) {
  const animate = (currentTime) => {
    const progress = Math.min(elapsed / duration, 1);
    const easedProgress = easingFunctions[easing](progress);
    const currentValue = startValue + (endValue - startValue) * easedProgress;
    setDisplayValue(currentValue);
  };
}
```

**Q: How would you add a gradient fill?**
```jsx
<div
  className="progress-bar-fill"
  style={{
    background: `linear-gradient(90deg, #667eea 0%, #764ba2 100%)`,
    width: `${percentage}%`
  }}
/>
```

## Performance Considerations

1. **Animation Performance**
   - Use CSS transitions for simple animations
   - Use RAF for complex animations with easing
   - Cancel RAF on unmount to prevent memory leaks

2. **Avoid Layout Thrashing**
   - Batch DOM reads and writes
   - Use transform instead of width when possible
   - Debounce rapid value updates

3. **Large Numbers of Progress Bars**
   - Use CSS animations instead of JS
   - Virtualize if rendering 100+ bars
   - Memoize components

## Edge Cases Handled
- Value clamping (0 to max)
- Smooth animation with RAF
- Custom easing functions
- Color coding based on value
- Striped pattern animation
- Indeterminate loading state
- Circular progress with SVG
- ARIA attributes for accessibility
- Custom colors, heights, sizes
- Label visibility toggle
- Animation cleanup on unmount
