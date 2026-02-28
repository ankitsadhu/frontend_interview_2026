# Debounced Search with API

## Problem
Create a search input that debounces user input, fetches results from an API, and displays them with loading/error states.

## Key Concepts
- `useState` for input and results
- `useEffect` with debouncing
- `AbortController` for canceling requests
- Error handling and loading states

## MANG-Level Expectations
- **Performance**: Debouncing to reduce API calls, cancel in-flight requests
- **UX**: Loading states, error handling, empty states, keyboard navigation
- **Robustness**: Handle race conditions, network errors, edge cases
- **Optimization**: Cache results, retry logic, request deduplication

## Solution

```jsx
import { useState, useEffect, useRef } from 'react';

/**
 * TRICKY PART #1: Debouncing with cleanup
 * - Delay API call until user stops typing
 * - MANG expectation: Explain why debouncing is needed
 * - Reduces API calls, improves performance, better UX
 */
function DebouncedSearch({ 
  apiEndpoint, 
  debounceMs = 300,
  minChars = 2,
  placeholder = 'Search...'
}) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [selectedIndex, setSelectedIndex] = useState(-1);

  /**
   * TRICKY PART #2: AbortController for request cancellation
   * - Cancel previous request when new one starts
   * - MANG will ask: "How do you handle race conditions?"
   * - AbortController prevents stale results from appearing
   */
  const abortControllerRef = useRef(null);

  /**
   * TRICKY PART #3: Debounced API call with cleanup
   * - setTimeout to delay execution
   * - Cleanup function cancels timer and aborts request
   * - CRITICAL: Prevents memory leaks and race conditions
   */
  useEffect(() => {
    // Don't search if query is too short
    if (query.length < minChars) {
      setResults([]);
      setError(null);
      return;
    }

    // Debounce timer
    const timer = setTimeout(async () => {
      // Cancel previous request if exists
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }

      // Create new AbortController for this request
      abortControllerRef.current = new AbortController();

      setLoading(true);
      setError(null);

      try {
        const response = await fetch(
          `${apiEndpoint}?q=${encodeURIComponent(query)}`,
          { signal: abortControllerRef.current.signal }
        );

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const data = await response.json();
        setResults(data.results || data);
        setSelectedIndex(-1);
      } catch (err) {
        // Don't set error if request was aborted
        if (err.name !== 'AbortError') {
          setError(err.message);
          setResults([]);
        }
      } finally {
        setLoading(false);
      }
    }, debounceMs);

    // Cleanup function
    return () => {
      clearTimeout(timer);
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }
    };
  }, [query, apiEndpoint, debounceMs, minChars]);

  /**
   * TRICKY PART #4: Keyboard navigation
   * - Arrow keys to navigate results
   * - Enter to select
   * - Escape to clear
   * - MANG expectation: Full keyboard accessibility
   */
  const handleKeyDown = (e) => {
    if (results.length === 0) return;

    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setSelectedIndex(prev => 
          prev < results.length - 1 ? prev + 1 : prev
        );
        break;
      case 'ArrowUp':
        e.preventDefault();
        setSelectedIndex(prev => prev > 0 ? prev - 1 : -1);
        break;
      case 'Enter':
        e.preventDefault();
        if (selectedIndex >= 0) {
          handleSelect(results[selectedIndex]);
        }
        break;
      case 'Escape':
        setQuery('');
        setResults([]);
        setSelectedIndex(-1);
        break;
    }
  };

  const handleSelect = (result) => {
    console.log('Selected:', result);
    setQuery(result.name || result.title || '');
    setResults([]);
    setSelectedIndex(-1);
  };

  const handleInputChange = (e) => {
    setQuery(e.target.value);
  };

  const handleClear = () => {
    setQuery('');
    setResults([]);
    setError(null);
    setSelectedIndex(-1);
  };

  return (
    <div className="debounced-search">
      <div className="search-input-wrapper">
        <input
          type="text"
          value={query}
          onChange={handleInputChange}
          onKeyDown={handleKeyDown}
          placeholder={placeholder}
          className="search-input"
          aria-label="Search"
          aria-autocomplete="list"
          aria-controls="search-results"
          aria-activedescendant={
            selectedIndex >= 0 ? `result-${selectedIndex}` : undefined
          }
        />
        {query && (
          <button
            className="search-clear"
            onClick={handleClear}
            aria-label="Clear search"
          >
            ×
          </button>
        )}
        {loading && <div className="search-spinner" aria-label="Loading" />}
      </div>

      {/* Results dropdown */}
      {query.length >= minChars && (
        <div
          id="search-results"
          className="search-results"
          role="listbox"
        >
          {error && (
            <div className="search-error" role="alert">
              Error: {error}
            </div>
          )}

          {!loading && !error && results.length === 0 && (
            <div className="search-empty">No results found</div>
          )}

          {!error && results.length > 0 && (
            <ul className="search-results-list">
              {results.map((result, index) => (
                <li
                  key={result.id || index}
                  id={`result-${index}`}
                  className={`search-result-item ${
                    index === selectedIndex ? 'selected' : ''
                  }`}
                  onClick={() => handleSelect(result)}
                  onMouseEnter={() => setSelectedIndex(index)}
                  role="option"
                  aria-selected={index === selectedIndex}
                >
                  {result.name || result.title}
                </li>
              ))}
            </ul>
          )}
        </div>
      )}

      {query.length > 0 && query.length < minChars && (
        <div className="search-hint">
          Type at least {minChars} characters to search
        </div>
      )}
    </div>
  );
}

export default DebouncedSearch;
```

## CSS

```css
.debounced-search {
  position: relative;
  width: 100%;
  max-width: 500px;
}

.search-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-input {
  width: 100%;
  padding: 0.75rem 2.5rem 0.75rem 1rem;
  font-size: 1rem;
  border: 2px solid #ddd;
  border-radius: 8px;
  outline: none;
  transition: border-color 0.2s;
}

.search-input:focus {
  border-color: #007bff;
}

.search-clear {
  position: absolute;
  right: 0.5rem;
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  color: #666;
  padding: 0.25rem;
}

.search-clear:hover {
  color: #000;
}

.search-spinner {
  position: absolute;
  right: 0.75rem;
  width: 20px;
  height: 20px;
  border: 2px solid #f3f3f3;
  border-top: 2px solid #007bff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.search-results {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  margin-top: 0.5rem;
  background: white;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  max-height: 300px;
  overflow-y: auto;
  z-index: 1000;
}

.search-results-list {
  list-style: none;
  margin: 0;
  padding: 0;
}

.search-result-item {
  padding: 0.75rem 1rem;
  cursor: pointer;
  transition: background 0.2s;
}

.search-result-item:hover,
.search-result-item.selected {
  background: #f0f0f0;
}

.search-error {
  padding: 1rem;
  color: #d32f2f;
  text-align: center;
}

.search-empty {
  padding: 1rem;
  color: #666;
  text-align: center;
}

.search-hint {
  margin-top: 0.5rem;
  font-size: 0.875rem;
  color: #666;
}
```

## Usage

```jsx
// Mock API endpoint
const searchAPI = 'https://api.example.com/search';

<DebouncedSearch
  apiEndpoint={searchAPI}
  debounceMs={300}
  minChars={2}
  placeholder="Search products..."
/>
```

## Test Cases

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import DebouncedSearch from './DebouncedSearch';

// Mock fetch
global.fetch = jest.fn();

jest.useFakeTimers();

describe('DebouncedSearch', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  afterEach(() => {
    jest.clearAllTimers();
  });

  test('renders search input', () => {
    render(<DebouncedSearch apiEndpoint="/api/search" />);
    expect(screen.getByPlaceholderText('Search...')).toBeInTheDocument();
  });

  test('debounces API calls', async () => {
    fetch.mockResolvedValue({
      ok: true,
      json: async () => ({ results: [{ id: 1, name: 'Result 1' }] })
    });

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={300} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'te' } });
    expect(fetch).not.toHaveBeenCalled();

    fireEvent.change(input, { target: { value: 'tes' } });
    expect(fetch).not.toHaveBeenCalled();

    jest.advanceTimersByTime(300);

    await waitFor(() => {
      expect(fetch).toHaveBeenCalledTimes(1);
    });
  });

  test('shows loading spinner while fetching', async () => {
    fetch.mockImplementation(() => new Promise(() => {})); // Never resolves

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'test' } });

    await waitFor(() => {
      expect(screen.getByLabelText('Loading')).toBeInTheDocument();
    });
  });

  test('displays search results', async () => {
    fetch.mockResolvedValue({
      ok: true,
      json: async () => ({
        results: [
          { id: 1, name: 'Result 1' },
          { id: 2, name: 'Result 2' }
        ]
      })
    });

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'test' } });

    await waitFor(() => {
      expect(screen.getByText('Result 1')).toBeInTheDocument();
      expect(screen.getByText('Result 2')).toBeInTheDocument();
    });
  });

  test('handles API errors', async () => {
    fetch.mockRejectedValue(new Error('Network error'));

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'test' } });

    await waitFor(() => {
      expect(screen.getByText(/Error: Network error/)).toBeInTheDocument();
    });
  });

  test('shows empty state when no results', async () => {
    fetch.mockResolvedValue({
      ok: true,
      json: async () => ({ results: [] })
    });

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'xyz' } });

    await waitFor(() => {
      expect(screen.getByText('No results found')).toBeInTheDocument();
    });
  });

  test('does not search with less than minimum characters', () => {
    render(<DebouncedSearch apiEndpoint="/api/search" minChars={3} debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'te' } });

    expect(screen.getByText(/Type at least 3 characters/)).toBeInTheDocument();
    expect(fetch).not.toHaveBeenCalled();
  });

  test('clears search on clear button click', async () => {
    fetch.mockResolvedValue({
      ok: true,
      json: async () => ({ results: [{ id: 1, name: 'Result 1' }] })
    });

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'test' } });

    await waitFor(() => screen.getByText('Result 1'));

    fireEvent.click(screen.getByLabelText('Clear search'));

    expect(input.value).toBe('');
    expect(screen.queryByText('Result 1')).not.toBeInTheDocument();
  });

  test('navigates results with keyboard', async () => {
    fetch.mockResolvedValue({
      ok: true,
      json: async () => ({
        results: [
          { id: 1, name: 'Result 1' },
          { id: 2, name: 'Result 2' }
        ]
      })
    });

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'test' } });

    await waitFor(() => screen.getByText('Result 1'));

    fireEvent.keyDown(input, { key: 'ArrowDown' });
    expect(screen.getByText('Result 1').closest('li')).toHaveClass('selected');

    fireEvent.keyDown(input, { key: 'ArrowDown' });
    expect(screen.getByText('Result 2').closest('li')).toHaveClass('selected');

    fireEvent.keyDown(input, { key: 'ArrowUp' });
    expect(screen.getByText('Result 1').closest('li')).toHaveClass('selected');
  });

  test('selects result on Enter key', async () => {
    const consoleSpy = jest.spyOn(console, 'log').mockImplementation();

    fetch.mockResolvedValue({
      ok: true,
      json: async () => ({ results: [{ id: 1, name: 'Result 1' }] })
    });

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'test' } });
    await waitFor(() => screen.getByText('Result 1'));

    fireEvent.keyDown(input, { key: 'ArrowDown' });
    fireEvent.keyDown(input, { key: 'Enter' });

    expect(consoleSpy).toHaveBeenCalledWith('Selected:', { id: 1, name: 'Result 1' });

    consoleSpy.mockRestore();
  });

  test('aborts previous request on new search', async () => {
    const abortSpy = jest.spyOn(AbortController.prototype, 'abort');

    fetch.mockImplementation(() => new Promise(resolve => {
      setTimeout(() => resolve({
        ok: true,
        json: async () => ({ results: [] })
      }), 100);
    }));

    render(<DebouncedSearch apiEndpoint="/api/search" debounceMs={0} />);
    const input = screen.getByPlaceholderText('Search...');

    fireEvent.change(input, { target: { value: 'test1' } });
    fireEvent.change(input, { target: { value: 'test2' } });

    await waitFor(() => {
      expect(abortSpy).toHaveBeenCalled();
    });

    abortSpy.mockRestore();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add result caching?**
```jsx
const cacheRef = useRef(new Map());

useEffect(() => {
  // Check cache first
  if (cacheRef.current.has(query)) {
    setResults(cacheRef.current.get(query));
    return;
  }

  // ... fetch logic

  // Cache results
  cacheRef.current.set(query, data.results);
  
  // Limit cache size
  if (cacheRef.current.size > 50) {
    const firstKey = cacheRef.current.keys().next().value;
    cacheRef.current.delete(firstKey);
  }
}, [query]);
```

**Q: How would you add retry logic for failed requests?**
```jsx
const [retryCount, setRetryCount] = useState(0);
const MAX_RETRIES = 3;

const fetchWithRetry = async (url, options, retriesLeft = MAX_RETRIES) => {
  try {
    const response = await fetch(url, options);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response;
  } catch (error) {
    if (retriesLeft > 0 && error.name !== 'AbortError') {
      await new Promise(resolve => setTimeout(resolve, 1000));
      return fetchWithRetry(url, options, retriesLeft - 1);
    }
    throw error;
  }
};
```

**Q: How would you add search history?**
```jsx
const [searchHistory, setSearchHistory] = useState(() => {
  const saved = localStorage.getItem('searchHistory');
  return saved ? JSON.parse(saved) : [];
});

const addToHistory = (query) => {
  const updated = [query, ...searchHistory.filter(q => q !== query)].slice(0, 10);
  setSearchHistory(updated);
  localStorage.setItem('searchHistory', JSON.stringify(updated));
};

// Show history when input is focused and empty
{query === '' && searchHistory.length > 0 && (
  <div className="search-history">
    {searchHistory.map((item, i) => (
      <div key={i} onClick={() => setQuery(item)}>
        {item}
      </div>
    ))}
  </div>
)}
```

**Q: How would you add highlighting of search terms in results?**
```jsx
const highlightMatch = (text, query) => {
  if (!query) return text;
  
  const parts = text.split(new RegExp(`(${query})`, 'gi'));
  return parts.map((part, i) =>
    part.toLowerCase() === query.toLowerCase() ? (
      <mark key={i}>{part}</mark>
    ) : (
      part
    )
  );
};

<li>{highlightMatch(result.name, query)}</li>
```

## Edge Cases Handled
- Debouncing to reduce API calls
- AbortController to cancel in-flight requests
- Race condition prevention
- Minimum character requirement
- Loading states
- Error handling
- Empty results
- Keyboard navigation (arrows, enter, escape)
- Clear button
- ARIA attributes for accessibility
- Request cleanup on unmount
- Network error handling
- HTTP error status handling
