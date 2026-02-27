# Searchable Dropdown / Autocomplete

## Problem
Create an autocomplete component with filtering, keyboard navigation, and match highlighting.

## Key Concepts
- `useState` for input and selection
- `useRef` for DOM access
- Debouncing for performance
- `onKeyDown` for keyboard navigation

## MANG-Level Expectations
- **Performance**: Debouncing to avoid excessive filtering/API calls
- **Accessibility**: Full keyboard navigation (Arrow keys, Enter, Escape, Tab)
- **Edge Cases**: Empty states, special characters in search, case-insensitive matching
- **UX Polish**: Highlight matching text, scroll highlighted item into view
- **Extensibility**: Support for async data fetching, custom rendering

## Solution

```jsx
import { useState, useRef, useEffect } from 'react';

/**
 * TRICKY PART #1: Debouncing
 * - Interviewers expect you to implement debouncing to avoid excessive filtering
 * - Use setTimeout + cleanup to cancel previous timers
 * - Alternative: Could extract to custom useDebounce hook for reusability
 */

function Autocomplete({ 
  options = [], 
  placeholder = 'Search...', 
  onSelect,
  debounceMs = 300 
}) {
  const [inputValue, setInputValue] = useState('');
  const [filteredOptions, setFilteredOptions] = useState([]);
  const [isOpen, setIsOpen] = useState(false);
  const [highlightedIndex, setHighlightedIndex] = useState(-1);
  const inputRef = useRef(null);
  const listRef = useRef(null);

  // Debounced filtering
  useEffect(() => {
    const timer = setTimeout(() => {
      if (inputValue.trim()) {
        // TRICKY: Case-insensitive search with trim
        // MANG expectation: Handle special regex characters if using regex
        const filtered = options.filter(option =>
          option.toLowerCase().includes(inputValue.toLowerCase())
        );
        setFilteredOptions(filtered);
        setIsOpen(filtered.length > 0);
      } else {
        setFilteredOptions([]);
        setIsOpen(false);
      }
    }, debounceMs);

    // CRITICAL: Cleanup to prevent memory leaks and stale updates
    return () => clearTimeout(timer);
  }, [inputValue, options, debounceMs]);

  const handleSelect = (option) => {
    setInputValue(option);
    setIsOpen(false);
    setHighlightedIndex(-1);
    onSelect?.(option);
  };

  /**
   * TRICKY PART #2: Keyboard Navigation
   * - Must prevent default behavior for arrow keys (page scroll)
   * - Handle boundary conditions (first/last item)
   * - MANG expectation: Also handle Tab, Home, End keys
   */
  const handleKeyDown = (e) => {
    if (!isOpen) return;

    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault(); // CRITICAL: Prevent page scroll
        setHighlightedIndex(prev => 
          prev < filteredOptions.length - 1 ? prev + 1 : prev
        );
        break;
      case 'ArrowUp':
        e.preventDefault();
        setHighlightedIndex(prev => prev > 0 ? prev - 1 : 0);
        break;
      case 'Enter':
        e.preventDefault();
        if (highlightedIndex >= 0) {
          handleSelect(filteredOptions[highlightedIndex]);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        setHighlightedIndex(-1);
        break;
      // EXTENSION: Add Home/End for MANG-level completeness
      // case 'Home': setHighlightedIndex(0); break;
      // case 'End': setHighlightedIndex(filteredOptions.length - 1); break;
    }
  };

  /**
   * TRICKY PART #3: Auto-scroll highlighted item
   * - Use scrollIntoView with 'nearest' to avoid jarring jumps
   * - MANG expectation: Smooth UX, no layout shifts
   */
  useEffect(() => {
    if (highlightedIndex >= 0 && listRef.current) {
      const highlightedElement = listRef.current.children[highlightedIndex];
      highlightedElement?.scrollIntoView({ block: 'nearest' });
    }
  }, [highlightedIndex]);

  /**
   * TRICKY PART #4: Highlight matching text
   * - Must escape special regex characters in production
   * - MANG expectation: Handle edge cases like empty query, special chars
   * - Could be asked to optimize for large strings
   */
  const highlightMatch = (text, query) => {
    if (!query) return text;
    
    // POTENTIAL ISSUE: Special regex chars need escaping
    // Production: query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
    const parts = text.split(new RegExp(`(${query})`, 'gi'));
    return parts.map((part, i) => 
      part.toLowerCase() === query.toLowerCase() 
        ? <mark key={i}>{part}</mark> 
        : part
    );
  };

  return (
    <div className="autocomplete">
      <input
        ref={inputRef}
        type="text"
        value={inputValue}
        onChange={(e) => setInputValue(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder={placeholder}
        className="autocomplete-input"
      />
      
      {isOpen && (
        <ul ref={listRef} className="autocomplete-list">
          {filteredOptions.map((option, index) => (
            <li
              key={option}
              className={`autocomplete-item ${
                index === highlightedIndex ? 'highlighted' : ''
              }`}
              onClick={() => handleSelect(option)}
              onMouseEnter={() => setHighlightedIndex(index)}
            >
              {highlightMatch(option, inputValue)}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

export default Autocomplete;
```

## CSS

```css
.autocomplete {
  position: relative;
  width: 300px;
}

.autocomplete-input {
  width: 100%;
  padding: 0.5rem;
  font-size: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.autocomplete-list {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  max-height: 200px;
  overflow-y: auto;
  background: white;
  border: 1px solid #ccc;
  border-top: none;
  list-style: none;
  margin: 0;
  padding: 0;
  z-index: 1000;
}

.autocomplete-item {
  padding: 0.5rem;
  cursor: pointer;
}

.autocomplete-item:hover,
.autocomplete-item.highlighted {
  background: #f0f0f0;
}

.autocomplete-item mark {
  background: #ffeb3b;
  font-weight: bold;
}
```

## Usage

```jsx
const fruits = [
  'Apple', 'Banana', 'Cherry', 'Date', 'Elderberry',
  'Fig', 'Grape', 'Honeydew', 'Kiwi', 'Lemon'
];

<Autocomplete 
  options={fruits}
  placeholder="Search fruits..."
  onSelect={(fruit) => console.log('Selected:', fruit)}
  debounceMs={300}
/>
```

## Test Cases

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Autocomplete from './Autocomplete';

describe('Autocomplete', () => {
  const mockOptions = ['Apple', 'Banana', 'Cherry', 'Date'];
  const mockOnSelect = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders input field', () => {
    render(<Autocomplete options={mockOptions} />);
    expect(screen.getByPlaceholderText('Search...')).toBeInTheDocument();
  });

  test('filters options based on input', async () => {
    render(<Autocomplete options={mockOptions} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'app' } });
    
    await waitFor(() => {
      expect(screen.getByText(/Apple/)).toBeInTheDocument();
      expect(screen.queryByText(/Banana/)).not.toBeInTheDocument();
    });
  });

  test('debounces search input', async () => {
    jest.useFakeTimers();
    render(<Autocomplete options={mockOptions} debounceMs={300} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'a' } });
    expect(screen.queryByText(/Apple/)).not.toBeInTheDocument();
    
    jest.advanceTimersByTime(300);
    
    await waitFor(() => {
      expect(screen.getByText(/Apple/)).toBeInTheDocument();
    });
    
    jest.useRealTimers();
  });

  test('keyboard navigation with arrow keys', async () => {
    render(<Autocomplete options={mockOptions} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'a' } });
    
    await waitFor(() => {
      expect(screen.getByText(/Apple/)).toBeInTheDocument();
    });
    
    fireEvent.keyDown(input, { key: 'ArrowDown' });
    expect(screen.getByText(/Apple/).parentElement).toHaveClass('highlighted');
    
    fireEvent.keyDown(input, { key: 'ArrowDown' });
    expect(screen.getByText(/Banana/).parentElement).toHaveClass('highlighted');
    
    fireEvent.keyDown(input, { key: 'ArrowUp' });
    expect(screen.getByText(/Apple/).parentElement).toHaveClass('highlighted');
  });

  test('selects option on Enter key', async () => {
    render(<Autocomplete options={mockOptions} onSelect={mockOnSelect} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'app' } });
    await waitFor(() => screen.getByText(/Apple/));
    
    fireEvent.keyDown(input, { key: 'ArrowDown' });
    fireEvent.keyDown(input, { key: 'Enter' });
    
    expect(mockOnSelect).toHaveBeenCalledWith('Apple');
    expect(input.value).toBe('Apple');
  });

  test('closes dropdown on Escape key', async () => {
    render(<Autocomplete options={mockOptions} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'a' } });
    await waitFor(() => screen.getByText(/Apple/));
    
    fireEvent.keyDown(input, { key: 'Escape' });
    
    expect(screen.queryByText(/Apple/)).not.toBeInTheDocument();
  });

  test('selects option on click', async () => {
    render(<Autocomplete options={mockOptions} onSelect={mockOnSelect} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'ban' } });
    await waitFor(() => screen.getByText(/Banana/));
    
    fireEvent.click(screen.getByText(/Banana/));
    
    expect(mockOnSelect).toHaveBeenCalledWith('Banana');
    expect(input.value).toBe('Banana');
  });

  test('highlights matching text', async () => {
    render(<Autocomplete options={mockOptions} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'app' } });
    
    await waitFor(() => {
      const mark = screen.getByText('App').closest('mark');
      expect(mark).toBeInTheDocument();
    });
  });

  test('handles empty search', async () => {
    render(<Autocomplete options={mockOptions} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'xyz' } });
    
    await waitFor(() => {
      expect(screen.queryByRole('list')).not.toBeInTheDocument();
    });
  });

  test('case-insensitive search', async () => {
    render(<Autocomplete options={mockOptions} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');
    
    fireEvent.change(input, { target: { value: 'APPLE' } });
    
    await waitFor(() => {
      expect(screen.getByText(/Apple/)).toBeInTheDocument();
    });
  });
});
```

## MANG Interview Follow-ups

**Q: How would you handle async API calls instead of local filtering?**
```jsx
// Add loading state and AbortController for cleanup
const [loading, setLoading] = useState(false);

useEffect(() => {
  const controller = new AbortController();
  
  const fetchResults = async () => {
    if (!inputValue.trim()) return;
    
    setLoading(true);
    try {
      const response = await fetch(`/api/search?q=${inputValue}`, {
        signal: controller.signal
      });
      const data = await response.json();
      setFilteredOptions(data);
    } catch (err) {
      if (err.name !== 'AbortError') {
        console.error(err);
      }
    } finally {
      setLoading(false);
    }
  };
  
  const timer = setTimeout(fetchResults, debounceMs);
  
  return () => {
    clearTimeout(timer);
    controller.abort(); // Cancel in-flight requests
  };
}, [inputValue, debounceMs]);
```

**Q: How would you handle 10,000+ options efficiently?**
- Use virtualization (react-window or react-virtual)
- Implement server-side filtering
- Add pagination to results
- Consider fuzzy search libraries (fuse.js)

**Q: How would you make this accessible?**
- Add ARIA attributes: `role="combobox"`, `aria-expanded`, `aria-activedescendant`
- Announce results count to screen readers
- Ensure focus management
- Add `aria-label` for icon buttons

**Q: How would you test the debouncing logic?**
- Use `jest.useFakeTimers()` and `jest.advanceTimersByTime()`
- Test that filtering doesn't happen immediately
- Test cleanup on unmount

## Edge Cases Handled
- Debounced search for performance
- Keyboard navigation (Arrow keys, Enter, Escape)
- Highlighted match text
- Auto-scroll highlighted item into view
- Case-insensitive matching
- Empty search results
- Special characters in search (needs escaping in production)
- Cleanup on unmount to prevent memory leaks
