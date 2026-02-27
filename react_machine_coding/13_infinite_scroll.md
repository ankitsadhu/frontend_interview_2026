# Infinite Scroll List

## Problem
Create an infinite scroll list that fetches data on scroll with loading indicator and "no more items" state.

## Key Concepts
- `useEffect` for data fetching
- `IntersectionObserver` for scroll detection
- Pagination state management

## MANG-Level Expectations
- **Performance**: Use IntersectionObserver (not scroll events), prevent duplicate requests
- **Error Handling**: Retry mechanism, error states, network failures
- **UX**: Loading states, skeleton screens, smooth transitions
- **Memory**: Cleanup observers, abort in-flight requests
- **Extensibility**: Support for bi-directional scroll, pull-to-refresh

## Solution

```jsx
import { useState, useEffect, useRef, useCallback } from 'react';

/**
 * TRICKY PART #1: Prevent duplicate requests
 * - Use loading flag to prevent multiple simultaneous fetches
 * - MANG expectation: Handle race conditions properly
 * - useCallback to stabilize function reference
 */

function InfiniteScrollList({ fetchItems, pageSize = 20 }) {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const [error, setError] = useState(null);
  
  const observerTarget = useRef(null);

  // Fetch data
  const loadMore = useCallback(async () => {
    // CRITICAL: Prevent duplicate requests
    if (loading || !hasMore) return;

    setLoading(true);
    setError(null);

    try {
      const newItems = await fetchItems(page, pageSize);
      
      // TRICKY: Check for end of data
      if (newItems.length === 0) {
        setHasMore(false);
      } else {
        // IMPORTANT: Use functional update to avoid stale closure
        setItems(prev => [...prev, ...newItems]);
        setPage(prev => prev + 1);
      }
    } catch (err) {
      setError(err.message);
      // MANG expectation: Don't increment page on error
    } finally {
      setLoading(false);
    }
  }, [page, loading, hasMore, fetchItems, pageSize]);

  /**
   * TRICKY PART #2: IntersectionObserver setup
   * - More performant than scroll events
   * - MANG expectation: Proper cleanup to avoid memory leaks
   * - threshold: 1.0 means trigger when fully visible
   */
  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        // IMPORTANT: Check all conditions before loading
        if (entries[0].isIntersecting && hasMore && !loading) {
          loadMore();
        }
      },
      { threshold: 1.0 } // Trigger when target is fully visible
    );

    const currentTarget = observerTarget.current;
    if (currentTarget) {
      observer.observe(currentTarget);
    }

    // CRITICAL: Cleanup to prevent memory leaks
    return () => {
      if (currentTarget) {
        observer.unobserve(currentTarget);
      }
    };
  }, [loadMore, hasMore, loading]);

  /**
   * TRICKY PART #3: Initial load
   * - Empty dependency array for mount-only effect
   * - eslint-disable needed but be ready to explain why
   * - MANG might ask: "Why disable the lint rule?"
   * - Answer: We only want this to run once on mount
   */
  useEffect(() => {
    loadMore();
  }, []); // eslint-disable-line react-hooks/exhaustive-deps

  return (
    <div className="infinite-scroll">
      <div className="items-container">
        {items.map((item, index) => (
          <div key={item.id || index} className="item">
            {item.title || item.name || JSON.stringify(item)}
          </div>
        ))}
      </div>

      {/* Loading indicator */}
      {loading && (
        <div className="loading">
          <div className="spinner"></div>
          <p>Loading...</p>
        </div>
      )}

      {/* Error state */}
      {error && (
        <div className="error">
          <p>Error: {error}</p>
          <button onClick={loadMore}>Retry</button>
        </div>
      )}

      {/* No more items */}
      {!hasMore && items.length > 0 && (
        <div className="end-message">
          No more items to load
        </div>
      )}

      {/* Empty state */}
      {!loading && items.length === 0 && (
        <div className="empty">
          No items found
        </div>
      )}

      {/* Observer target */}
      <div ref={observerTarget} className="observer-target" />
    </div>
  );
}

export default InfiniteScrollList;
```

## Mock API Function

```jsx
// Example usage with mock API
const mockFetchItems = async (page, pageSize) => {
  // Simulate API delay
  await new Promise(resolve => setTimeout(resolve, 1000));
  
  // Simulate pagination
  const start = (page - 1) * pageSize;
  const end = start + pageSize;
  
  // Return empty array after page 5 to simulate end
  if (page > 5) return [];
  
  return Array.from({ length: pageSize }, (_, i) => ({
    id: start + i,
    title: `Item ${start + i + 1}`,
    description: `Description for item ${start + i + 1}`
  }));
};

// Usage
<InfiniteScrollList 
  fetchItems={mockFetchItems}
  pageSize={20}
/>
```

## CSS

```css
.infinite-scroll {
  max-width: 800px;
  margin: 0 auto;
  padding: 1rem;
}

.items-container {
  display: grid;
  gap: 1rem;
}

.item {
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
}

.loading, .error, .end-message, .empty {
  text-align: center;
  padding: 2rem;
  color: #666;
}

.spinner {
  width: 40px;
  height: 40px;
  margin: 0 auto 1rem;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.error {
  color: #d32f2f;
}

.error button {
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  cursor: pointer;
}

.observer-target {
  height: 20px;
}
```

## Edge Cases Handled
- Loading state with spinner
- Error handling with retry
- "No more items" message
- Empty state
- Prevents duplicate requests
- Cleanup on unmount
- IntersectionObserver for performance


## Test Cases

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import InfiniteScrollList from './InfiniteScrollList';

// Mock IntersectionObserver
const mockIntersectionObserver = jest.fn();
mockIntersectionObserver.mockReturnValue({
  observe: () => null,
  unobserve: () => null,
  disconnect: () => null
});
window.IntersectionObserver = mockIntersectionObserver;

describe('InfiniteScrollList', () => {
  const mockFetchItems = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders initial items', async () => {
    mockFetchItems.mockResolvedValueOnce([
      { id: 1, title: 'Item 1' },
      { id: 2, title: 'Item 2' }
    ]);

    render(<InfiniteScrollList fetchItems={mockFetchItems} pageSize={20} />);

    await waitFor(() => {
      expect(screen.getByText('Item 1')).toBeInTheDocument();
      expect(screen.getByText('Item 2')).toBeInTheDocument();
    });
  });

  test('shows loading indicator', async () => {
    mockFetchItems.mockImplementation(() => new Promise(() => {})); // Never resolves

    render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  test('handles fetch error', async () => {
    mockFetchItems.mockRejectedValueOnce(new Error('Network error'));

    render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    await waitFor(() => {
      expect(screen.getByText(/Error: Network error/)).toBeInTheDocument();
    });
  });

  test('shows retry button on error', async () => {
    mockFetchItems.mockRejectedValueOnce(new Error('Failed'));

    render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    await waitFor(() => {
      expect(screen.getByText('Retry')).toBeInTheDocument();
    });
  });

  test('retries fetch on retry button click', async () => {
    mockFetchItems
      .mockRejectedValueOnce(new Error('Failed'))
      .mockResolvedValueOnce([{ id: 1, title: 'Item 1' }]);

    render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    await waitFor(() => screen.getByText('Retry'));
    
    userEvent.click(screen.getByText('Retry'));

    await waitFor(() => {
      expect(screen.getByText('Item 1')).toBeInTheDocument();
    });
  });

  test('shows "no more items" when data exhausted', async () => {
    mockFetchItems
      .mockResolvedValueOnce([{ id: 1, title: 'Item 1' }])
      .mockResolvedValueOnce([]); // Empty array signals end

    const { rerender } = render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    await waitFor(() => screen.getByText('Item 1'));

    // Simulate intersection (trigger load more)
    const observerCallback = mockIntersectionObserver.mock.calls[0][0];
    observerCallback([{ isIntersecting: true }]);

    await waitFor(() => {
      expect(screen.getByText('No more items to load')).toBeInTheDocument();
    });
  });

  test('shows empty state when no items', async () => {
    mockFetchItems.mockResolvedValueOnce([]);

    render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    await waitFor(() => {
      expect(screen.getByText('No items found')).toBeInTheDocument();
    });
  });

  test('prevents duplicate requests', async () => {
    mockFetchItems.mockImplementation(() => 
      new Promise(resolve => setTimeout(() => resolve([{ id: 1, title: 'Item' }]), 100))
    );

    render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    // Try to trigger multiple loads
    const observerCallback = mockIntersectionObserver.mock.calls[0][0];
    observerCallback([{ isIntersecting: true }]);
    observerCallback([{ isIntersecting: true }]);
    observerCallback([{ isIntersecting: true }]);

    await waitFor(() => screen.getByText('Item'));

    // Should only call once (initial load)
    expect(mockFetchItems).toHaveBeenCalledTimes(1);
  });

  test('cleans up observer on unmount', () => {
    const unobserveMock = jest.fn();
    mockIntersectionObserver.mockReturnValue({
      observe: jest.fn(),
      unobserve: unobserveMock,
      disconnect: jest.fn()
    });

    const { unmount } = render(<InfiniteScrollList fetchItems={mockFetchItems} />);

    unmount();

    expect(unobserveMock).toHaveBeenCalled();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you handle AbortController for cleanup?**
```jsx
const loadMore = useCallback(async () => {
  if (loading || !hasMore) return;

  const controller = new AbortController();
  setLoading(true);

  try {
    const response = await fetch(`/api/items?page=${page}`, {
      signal: controller.signal
    });
    const newItems = await response.json();
    // ... rest of logic
  } catch (err) {
    if (err.name !== 'AbortError') {
      setError(err.message);
    }
  }

  return () => controller.abort();
}, [page, loading, hasMore]);
```

**Q: How would you implement bi-directional infinite scroll?**
```jsx
// Track both top and bottom observers
// Maintain page ranges (e.g., pages 5-10 loaded)
// Prepend items for upward scroll
// More complex state management needed
```

**Q: How would you optimize for 100,000 items?**
```jsx
// 1. Use virtualization (react-window, react-virtual)
// 2. Implement windowing - only keep N pages in memory
// 3. Remove old items as new ones load
// 4. Use pagination with URL state for deep linking
```

**Q: How would you test IntersectionObserver?**
```jsx
// Mock IntersectionObserver in tests
const mockObserver = jest.fn();
mockObserver.mockReturnValue({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn()
});
window.IntersectionObserver = mockObserver;

// Trigger callback manually in tests
const callback = mockObserver.mock.calls[0][0];
callback([{ isIntersecting: true }]);
```

**Q: What about SEO and initial server-side rendering?**
```jsx
// 1. Server-render first page of items
// 2. Hydrate with initial data
// 3. Use Next.js getServerSideProps or getStaticProps
// 4. Consider pagination URLs for crawlability
```

## Performance Considerations

1. **IntersectionObserver vs Scroll Events**
   - IntersectionObserver is more performant (runs off main thread)
   - Scroll events need throttling/debouncing
   - Better battery life on mobile

2. **Preventing Duplicate Requests**
   - Use loading flag
   - Check hasMore before fetching
   - Consider request deduplication library

3. **Memory Management**
   - For very long lists, implement windowing
   - Remove items outside viewport
   - Use virtualization libraries

## Edge Cases Handled
- Loading state with spinner
- Error handling with retry
- "No more items" message
- Empty state
- Prevents duplicate requests
- Cleanup on unmount
- IntersectionObserver for performance
- Race condition prevention
- Functional state updates to avoid stale closures
