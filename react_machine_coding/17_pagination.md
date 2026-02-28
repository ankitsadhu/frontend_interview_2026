# Pagination Component

## Problem
Create a pagination component with page numbers, prev/next buttons, and ellipsis for large page ranges.

## Key Concepts
- `useState` for current page
- Derived state for page range calculation
- `useMemo` for performance optimization
- Smart ellipsis logic

## MANG-Level Expectations
- **Smart Pagination**: Ellipsis for large ranges, configurable siblings
- **UX**: Disabled states, keyboard navigation, jump to page
- **Accessibility**: ARIA attributes, keyboard support, screen reader friendly
- **Performance**: Memoization, avoid unnecessary re-renders
- **Extensibility**: Custom page sizes, total count display, URL sync

## Solution

```jsx
import { useState, useMemo } from 'react';

/**
 * TRICKY PART #1: Page range calculation with ellipsis
 * - Show first, last, current, and siblings
 * - Add ellipsis when there are gaps
 * - MANG expectation: Explain the algorithm
 */
function Pagination({
  totalItems,
  itemsPerPage = 10,
  currentPage = 1,
  onPageChange,
  siblingCount = 1,
  showFirstLast = true
}) {
  const totalPages = Math.ceil(totalItems / itemsPerPage);

  /**
   * TRICKY PART #2: Generate page numbers with ellipsis
   * - Complex logic to determine which pages to show
   * - MANG will ask: "How do you decide where to put ellipsis?"
   * - Show current page, siblings, first, last, and ellipsis for gaps
   */
  const pageNumbers = useMemo(() => {
    // Total page numbers to show (excluding ellipsis)
    const totalPageNumbers = siblingCount * 2 + 5; // siblings + current + first + last + 2 ellipsis

    // If total pages less than total numbers to show, return all pages
    if (totalPages <= totalPageNumbers) {
      return Array.from({ length: totalPages }, (_, i) => i + 1);
    }

    const leftSiblingIndex = Math.max(currentPage - siblingCount, 1);
    const rightSiblingIndex = Math.min(currentPage + siblingCount, totalPages);

    const shouldShowLeftEllipsis = leftSiblingIndex > 2;
    const shouldShowRightEllipsis = rightSiblingIndex < totalPages - 1;

    const firstPageIndex = 1;
    const lastPageIndex = totalPages;

    // No ellipsis on either side
    if (!shouldShowLeftEllipsis && !shouldShowRightEllipsis) {
      return Array.from({ length: totalPages }, (_, i) => i + 1);
    }

    // Only right ellipsis
    if (!shouldShowLeftEllipsis && shouldShowRightEllipsis) {
      const leftRange = Array.from(
        { length: 3 + 2 * siblingCount },
        (_, i) => i + 1
      );
      return [...leftRange, '...', totalPages];
    }

    // Only left ellipsis
    if (shouldShowLeftEllipsis && !shouldShowRightEllipsis) {
      const rightRange = Array.from(
        { length: 3 + 2 * siblingCount },
        (_, i) => totalPages - (3 + 2 * siblingCount) + i + 1
      );
      return [firstPageIndex, '...', ...rightRange];
    }

    // Both ellipsis
    const middleRange = Array.from(
      { length: rightSiblingIndex - leftSiblingIndex + 1 },
      (_, i) => leftSiblingIndex + i
    );
    return [firstPageIndex, '...', ...middleRange, '...', lastPageIndex];
  }, [totalPages, currentPage, siblingCount]);

  const handlePageChange = (page) => {
    if (page < 1 || page > totalPages || page === currentPage) return;
    onPageChange?.(page);
  };

  const goToNextPage = () => handlePageChange(currentPage + 1);
  const goToPrevPage = () => handlePageChange(currentPage - 1);
  const goToFirstPage = () => handlePageChange(1);
  const goToLastPage = () => handlePageChange(totalPages);

  /**
   * TRICKY PART #3: Keyboard navigation
   * - Arrow keys for prev/next
   * - Home/End for first/last
   * - MANG expectation: Full keyboard accessibility
   */
  const handleKeyDown = (e, page) => {
    switch (e.key) {
      case 'ArrowLeft':
        e.preventDefault();
        goToPrevPage();
        break;
      case 'ArrowRight':
        e.preventDefault();
        goToNextPage();
        break;
      case 'Home':
        e.preventDefault();
        goToFirstPage();
        break;
      case 'End':
        e.preventDefault();
        goToLastPage();
        break;
      case 'Enter':
      case ' ':
        if (typeof page === 'number') {
          e.preventDefault();
          handlePageChange(page);
        }
        break;
    }
  };

  if (totalPages <= 1) return null;

  const startItem = (currentPage - 1) * itemsPerPage + 1;
  const endItem = Math.min(currentPage * itemsPerPage, totalItems);

  return (
    <nav className="pagination" role="navigation" aria-label="Pagination">
      {/* Items info */}
      <div className="pagination-info">
        Showing {startItem}-{endItem} of {totalItems}
      </div>

      <ul className="pagination-list">
        {/* First page button */}
        {showFirstLast && (
          <li>
            <button
              className="pagination-button"
              onClick={goToFirstPage}
              disabled={currentPage === 1}
              aria-label="Go to first page"
            >
              «
            </button>
          </li>
        )}

        {/* Previous button */}
        <li>
          <button
            className="pagination-button"
            onClick={goToPrevPage}
            disabled={currentPage === 1}
            aria-label="Go to previous page"
          >
            ‹
          </button>
        </li>

        {/* Page numbers */}
        {pageNumbers.map((pageNumber, index) => {
          if (pageNumber === '...') {
            return (
              <li key={`ellipsis-${index}`}>
                <span className="pagination-ellipsis">...</span>
              </li>
            );
          }

          return (
            <li key={pageNumber}>
              <button
                className={`pagination-button ${
                  currentPage === pageNumber ? 'active' : ''
                }`}
                onClick={() => handlePageChange(pageNumber)}
                onKeyDown={(e) => handleKeyDown(e, pageNumber)}
                aria-label={`Go to page ${pageNumber}`}
                aria-current={currentPage === pageNumber ? 'page' : undefined}
              >
                {pageNumber}
              </button>
            </li>
          );
        })}

        {/* Next button */}
        <li>
          <button
            className="pagination-button"
            onClick={goToNextPage}
            disabled={currentPage === totalPages}
            aria-label="Go to next page"
          >
            ›
          </button>
        </li>

        {/* Last page button */}
        {showFirstLast && (
          <li>
            <button
              className="pagination-button"
              onClick={goToLastPage}
              disabled={currentPage === totalPages}
              aria-label="Go to last page"
            >
              »
            </button>
          </li>
        )}
      </ul>
    </nav>
  );
}

export default Pagination;
```

## CSS

```css
.pagination {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
}

.pagination-info {
  font-size: 0.875rem;
  color: #666;
}

.pagination-list {
  display: flex;
  list-style: none;
  gap: 0.25rem;
  margin: 0;
  padding: 0;
}

.pagination-button {
  min-width: 40px;
  height: 40px;
  padding: 0.5rem;
  border: 1px solid #ddd;
  background: white;
  cursor: pointer;
  font-size: 0.875rem;
  border-radius: 4px;
  transition: all 0.2s;
}

.pagination-button:hover:not(:disabled) {
  background: #f5f5f5;
  border-color: #999;
}

.pagination-button:focus {
  outline: 2px solid #007bff;
  outline-offset: 2px;
}

.pagination-button.active {
  background: #007bff;
  color: white;
  border-color: #007bff;
  font-weight: bold;
}

.pagination-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.pagination-ellipsis {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 40px;
  height: 40px;
  color: #666;
}

/* Responsive */
@media (max-width: 640px) {
  .pagination-button {
    min-width: 36px;
    height: 36px;
    font-size: 0.75rem;
  }
}
```

## Usage

```jsx
function App() {
  const [currentPage, setCurrentPage] = useState(1);
  const itemsPerPage = 10;
  const totalItems = 250;

  return (
    <div>
      {/* Your data display */}
      <DataList
        items={getData(currentPage, itemsPerPage)}
      />

      {/* Pagination */}
      <Pagination
        totalItems={totalItems}
        itemsPerPage={itemsPerPage}
        currentPage={currentPage}
        onPageChange={setCurrentPage}
        siblingCount={1}
        showFirstLast={true}
      />
    </div>
  );
}
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import Pagination from './Pagination';

describe('Pagination', () => {
  const mockOnPageChange = jest.fn();

  beforeEach(() => {
    mockOnPageChange.mockClear();
  });

  test('renders pagination with correct page numbers', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={1}
        onPageChange={mockOnPageChange}
      />
    );

    expect(screen.getByText('1')).toBeInTheDocument();
    expect(screen.getByText('5')).toBeInTheDocument();
  });

  test('shows items info', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={2}
        onPageChange={mockOnPageChange}
      />
    );

    expect(screen.getByText('Showing 11-20 of 50')).toBeInTheDocument();
  });

  test('calls onPageChange when clicking page number', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={1}
        onPageChange={mockOnPageChange}
      />
    );

    fireEvent.click(screen.getByText('3'));

    expect(mockOnPageChange).toHaveBeenCalledWith(3);
  });

  test('disables previous button on first page', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={1}
        onPageChange={mockOnPageChange}
      />
    );

    const prevButton = screen.getByLabelText('Go to previous page');
    expect(prevButton).toBeDisabled();
  });

  test('disables next button on last page', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={5}
        onPageChange={mockOnPageChange}
      />
    );

    const nextButton = screen.getByLabelText('Go to next page');
    expect(nextButton).toBeDisabled();
  });

  test('navigates to next page', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={2}
        onPageChange={mockOnPageChange}
      />
    );

    fireEvent.click(screen.getByLabelText('Go to next page'));

    expect(mockOnPageChange).toHaveBeenCalledWith(3);
  });

  test('navigates to previous page', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={3}
        onPageChange={mockOnPageChange}
      />
    );

    fireEvent.click(screen.getByLabelText('Go to previous page'));

    expect(mockOnPageChange).toHaveBeenCalledWith(2);
  });

  test('shows ellipsis for large page ranges', () => {
    render(
      <Pagination
        totalItems={200}
        itemsPerPage={10}
        currentPage={10}
        onPageChange={mockOnPageChange}
        siblingCount={1}
      />
    );

    const ellipsis = screen.getAllByText('...');
    expect(ellipsis.length).toBeGreaterThan(0);
  });

  test('applies active class to current page', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={3}
        onPageChange={mockOnPageChange}
      />
    );

    const activePage = screen.getByText('3').closest('button');
    expect(activePage).toHaveClass('active');
  });

  test('navigates to first page', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={3}
        onPageChange={mockOnPageChange}
        showFirstLast={true}
      />
    );

    fireEvent.click(screen.getByLabelText('Go to first page'));

    expect(mockOnPageChange).toHaveBeenCalledWith(1);
  });

  test('navigates to last page', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={2}
        onPageChange={mockOnPageChange}
        showFirstLast={true}
      />
    );

    fireEvent.click(screen.getByLabelText('Go to last page'));

    expect(mockOnPageChange).toHaveBeenCalledWith(5);
  });

  test('does not render when only one page', () => {
    const { container } = render(
      <Pagination
        totalItems={5}
        itemsPerPage={10}
        currentPage={1}
        onPageChange={mockOnPageChange}
      />
    );

    expect(container.firstChild).toBeNull();
  });

  test('handles keyboard navigation', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={3}
        onPageChange={mockOnPageChange}
      />
    );

    const pageButton = screen.getByText('3');

    fireEvent.keyDown(pageButton, { key: 'ArrowRight' });
    expect(mockOnPageChange).toHaveBeenCalledWith(4);

    fireEvent.keyDown(pageButton, { key: 'ArrowLeft' });
    expect(mockOnPageChange).toHaveBeenCalledWith(2);
  });

  test('has correct ARIA attributes', () => {
    render(
      <Pagination
        totalItems={50}
        itemsPerPage={10}
        currentPage={3}
        onPageChange={mockOnPageChange}
      />
    );

    const nav = screen.getByRole('navigation');
    expect(nav).toHaveAttribute('aria-label', 'Pagination');

    const currentPage = screen.getByText('3').closest('button');
    expect(currentPage).toHaveAttribute('aria-current', 'page');
  });

  test('respects siblingCount prop', () => {
    render(
      <Pagination
        totalItems={200}
        itemsPerPage={10}
        currentPage={10}
        onPageChange={mockOnPageChange}
        siblingCount={2}
      />
    );

    // With siblingCount=2, should show pages 8, 9, 10, 11, 12
    expect(screen.getByText('8')).toBeInTheDocument();
    expect(screen.getByText('12')).toBeInTheDocument();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add a page size selector?**
```jsx
function PaginationWithPageSize({ totalItems, onPageChange, onPageSizeChange }) {
  const [currentPage, setCurrentPage] = useState(1);
  const [pageSize, setPageSize] = useState(10);

  const handlePageSizeChange = (newSize) => {
    setPageSize(newSize);
    setCurrentPage(1); // Reset to first page
    onPageSizeChange?.(newSize);
  };

  return (
    <div className="pagination-container">
      <select
        value={pageSize}
        onChange={(e) => handlePageSizeChange(Number(e.target.value))}
      >
        <option value={10}>10 per page</option>
        <option value={25}>25 per page</option>
        <option value={50}>50 per page</option>
        <option value={100}>100 per page</option>
      </select>

      <Pagination
        totalItems={totalItems}
        itemsPerPage={pageSize}
        currentPage={currentPage}
        onPageChange={setCurrentPage}
      />
    </div>
  );
}
```

**Q: How would you add jump to page input?**
```jsx
function PaginationWithJump({ totalPages, currentPage, onPageChange }) {
  const [jumpValue, setJumpValue] = useState('');

  const handleJump = (e) => {
    e.preventDefault();
    const page = parseInt(jumpValue);
    if (page >= 1 && page <= totalPages) {
      onPageChange(page);
      setJumpValue('');
    }
  };

  return (
    <div className="pagination-with-jump">
      <Pagination {...props} />
      
      <form onSubmit={handleJump} className="jump-to-page">
        <label>
          Jump to page:
          <input
            type="number"
            min={1}
            max={totalPages}
            value={jumpValue}
            onChange={(e) => setJumpValue(e.target.value)}
          />
        </label>
        <button type="submit">Go</button>
      </form>
    </div>
  );
}
```

**Q: How would you sync pagination with URL?**
```jsx
import { useHistory, useLocation } from 'react-router-dom';

function PaginationWithURL({ totalItems, itemsPerPage }) {
  const history = useHistory();
  const location = useLocation();

  const getCurrentPage = () => {
    const params = new URLSearchParams(location.search);
    return parseInt(params.get('page') || '1');
  };

  const [currentPage, setCurrentPage] = useState(getCurrentPage);

  const handlePageChange = (page) => {
    setCurrentPage(page);
    history.push(`?page=${page}`);
  };

  // Sync with URL changes (back/forward)
  useEffect(() => {
    setCurrentPage(getCurrentPage());
  }, [location.search]);

  return (
    <Pagination
      totalItems={totalItems}
      itemsPerPage={itemsPerPage}
      currentPage={currentPage}
      onPageChange={handlePageChange}
    />
  );
}
```

**Q: How would you add infinite scroll as alternative?**
```jsx
function InfinitePagination({ totalItems, itemsPerPage, onLoadMore }) {
  const [page, setPage] = useState(1);
  const [items, setItems] = useState([]);
  const observerRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && items.length < totalItems) {
          setPage(prev => prev + 1);
          onLoadMore(page + 1);
        }
      },
      { threshold: 1.0 }
    );

    if (observerRef.current) {
      observer.observe(observerRef.current);
    }

    return () => observer.disconnect();
  }, [items.length, totalItems, page, onLoadMore]);

  return (
    <div>
      {/* Render items */}
      <div ref={observerRef} style={{ height: '20px' }} />
    </div>
  );
}
```

**Q: How would you optimize for large datasets?**
```jsx
// 1. Server-side pagination
const fetchPage = async (page, pageSize) => {
  const response = await fetch(`/api/items?page=${page}&size=${pageSize}`);
  return response.json();
};

// 2. Memoize page numbers calculation
const pageNumbers = useMemo(() => {
  // ... calculation
}, [totalPages, currentPage, siblingCount]);

// 3. Virtual scrolling for very large lists
import { FixedSizeList } from 'react-window';

// 4. Debounce rapid page changes
const debouncedPageChange = useMemo(
  () => debounce(onPageChange, 300),
  [onPageChange]
);
```

## Performance Considerations

1. **Memoization**
   - Use useMemo for page numbers calculation
   - Avoid recalculating on every render
   - Memoize component if used in lists

2. **Server-side Pagination**
   - Only fetch current page data
   - Use cursor-based pagination for large datasets
   - Cache pages for faster navigation

3. **Rendering Optimization**
   - Don't render if only one page
   - Use React.memo for page buttons
   - Debounce rapid page changes

## Edge Cases Handled
- Smart ellipsis placement
- Configurable sibling count
- First/last page buttons
- Disabled states at boundaries
- Keyboard navigation (arrows, home, end)
- ARIA attributes for accessibility
- Items count display
- Single page (no render)
- Large page ranges
- Responsive design
- useMemo for performance
