# Star Rating Component

## Problem
Create a star rating component with click to rate, hover preview, and read-only mode.

## Key Concepts
- `useState` for rating state
- `onMouseEnter/Leave` for hover effects
- Props for configuration

## Solution

```jsx
import { useState } from 'react';

function StarRating({ 
  totalStars = 5, 
  initialRating = 0, 
  readOnly = false,
  onChange 
}) {
  const [rating, setRating] = useState(initialRating);
  const [hoverRating, setHoverRating] = useState(0);

  const handleClick = (value) => {
    if (readOnly) return;
    setRating(value);
    onChange?.(value);
  };

  const handleMouseEnter = (value) => {
    if (readOnly) return;
    setHoverRating(value);
  };

  const handleMouseLeave = () => {
    if (readOnly) return;
    setHoverRating(0);
  };

  return (
    <div className="star-rating">
      {[...Array(totalStars)].map((_, index) => {
        const starValue = index + 1;
        const isFilled = starValue <= (hoverRating || rating);
        
        return (
          <span
            key={starValue}
            className={`star ${isFilled ? 'filled' : ''} ${readOnly ? 'read-only' : ''}`}
            onClick={() => handleClick(starValue)}
            onMouseEnter={() => handleMouseEnter(starValue)}
            onMouseLeave={handleMouseLeave}
          >
            ★
          </span>
        );
      })}
      <span className="rating-text">
        {rating} / {totalStars}
      </span>
    </div>
  );
}

export default StarRating;
```

## CSS

```css
.star-rating {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.star {
  font-size: 2rem;
  color: #ddd;
  cursor: pointer;
  transition: color 0.2s;
}

.star.filled {
  color: #ffc107;
}

.star:not(.read-only):hover {
  transform: scale(1.1);
}

.star.read-only {
  cursor: default;
}

.rating-text {
  margin-left: 0.5rem;
  font-size: 0.9rem;
  color: #666;
}
```

## Usage

```jsx
// Interactive rating
<StarRating 
  totalStars={5} 
  initialRating={3}
  onChange={(rating) => console.log('New rating:', rating)}
/>

// Read-only display
<StarRating 
  totalStars={5} 
  initialRating={4.5}
  readOnly={true}
/>
```

## Edge Cases Handled
- Read-only mode prevents interaction
- Hover preview only in interactive mode
- Configurable number of stars
- Optional onChange callback


## MANG-Level Expectations
- **Accessibility**: Keyboard navigation, ARIA labels, screen reader support
- **UX**: Hover effects, half-star support, size variants
- **Flexibility**: Custom icons, colors, tooltips
- **Performance**: Avoid unnecessary re-renders

## Enhanced Solution with Comments

```jsx
/**
 * TRICKY PART: Hover vs actual rating
 * - Show hoverRating while hovering, fall back to rating
 * - MANG expectation: Clean conditional logic
 */
const isFilled = starValue <= (hoverRating || rating);

/**
 * EXTENSION: Half-star support
 * - Track decimal ratings (e.g., 3.5)
 * - Render half-filled stars with CSS or SVG
 */
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import StarRating from './StarRating';

describe('StarRating', () => {
  test('renders correct number of stars', () => {
    render(<StarRating totalStars={5} />);
    const stars = screen.getAllByText('★');
    expect(stars).toHaveLength(5);
  });

  test('displays initial rating', () => {
    render(<StarRating totalStars={5} initialRating={3} />);
    expect(screen.getByText('3 / 5')).toBeInTheDocument();
  });

  test('updates rating on click', () => {
    const mockOnChange = jest.fn();
    render(<StarRating totalStars={5} onChange={mockOnChange} />);
    
    const stars = screen.getAllByText('★');
    fireEvent.click(stars[2]); // Click 3rd star
    
    expect(mockOnChange).toHaveBeenCalledWith(3);
    expect(screen.getByText('3 / 5')).toBeInTheDocument();
  });

  test('shows hover preview', () => {
    render(<StarRating totalStars={5} initialRating={2} />);
    
    const stars = screen.getAllByText('★');
    fireEvent.mouseEnter(stars[3]); // Hover 4th star
    
    // Check if 4 stars are filled (visual test would be better)
    expect(stars[3].classList.contains('filled')).toBe(true);
  });

  test('resets hover on mouse leave', () => {
    render(<StarRating totalStars={5} initialRating={2} />);
    
    const stars = screen.getAllByText('★');
    fireEvent.mouseEnter(stars[3]);
    fireEvent.mouseLeave(stars[3]);
    
    // Should show original rating
    expect(screen.getByText('2 / 5')).toBeInTheDocument();
  });

  test('read-only mode prevents interaction', () => {
    const mockOnChange = jest.fn();
    render(<StarRating totalStars={5} initialRating={3} readOnly onChange={mockOnChange} />);
    
    const stars = screen.getAllByText('★');
    fireEvent.click(stars[4]);
    
    expect(mockOnChange).not.toHaveBeenCalled();
    expect(screen.getByText('3 / 5')).toBeInTheDocument();
  });

  test('read-only mode prevents hover', () => {
    render(<StarRating totalStars={5} initialRating={2} readOnly />);
    
    const stars = screen.getAllByText('★');
    fireEvent.mouseEnter(stars[4]);
    
    // Should still show original rating
    expect(screen.getByText('2 / 5')).toBeInTheDocument();
  });

  test('handles zero rating', () => {
    render(<StarRating totalStars={5} initialRating={0} />);
    expect(screen.getByText('0 / 5')).toBeInTheDocument();
  });

  test('handles full rating', () => {
    render(<StarRating totalStars={5} initialRating={5} />);
    expect(screen.getByText('5 / 5')).toBeInTheDocument();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add half-star support?**
```jsx
const [rating, setRating] = useState(3.5);

const handleClick = (starValue, e) => {
  const rect = e.target.getBoundingClientRect();
  const isHalf = e.clientX - rect.left < rect.width / 2;
  const newRating = isHalf ? starValue - 0.5 : starValue;
  setRating(newRating);
};

// Render half-filled star with CSS clip-path or SVG
<span className={`star ${getStarClass(starValue, rating)}`}>
  ★
</span>

const getStarClass = (starValue, rating) => {
  if (starValue <= rating) return 'filled';
  if (starValue - 0.5 === rating) return 'half-filled';
  return '';
};
```

**Q: How would you make this accessible?**
```jsx
<div role="radiogroup" aria-label="Rating">
  {[...Array(totalStars)].map((_, index) => (
    <button
      key={index}
      role="radio"
      aria-checked={index + 1 === rating}
      aria-label={`${index + 1} star${index + 1 > 1 ? 's' : ''}`}
      onClick={() => handleClick(index + 1)}
    >
      ★
    </button>
  ))}
</div>

// Announce rating to screen readers
<div role="status" aria-live="polite" className="sr-only">
  Rating: {rating} out of {totalStars} stars
</div>
```

**Q: How would you add custom icons?**
```jsx
function StarRating({ 
  icon = '★',
  filledIcon = '★',
  emptyIcon = '☆'
}) {
  return (
    <span className="star">
      {isFilled ? filledIcon : emptyIcon}
    </span>
  );
}
```

**Q: How would you add tooltips?**
```jsx
const labels = ['Poor', 'Fair', 'Good', 'Very Good', 'Excellent'];

<span title={labels[starValue - 1]}>
  ★
</span>
```
