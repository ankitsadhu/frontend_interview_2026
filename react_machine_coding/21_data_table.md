# Data Table with Sort, Filter, Pagination

## Problem
Create a data table with column sorting, filtering, pagination, row selection, and bulk actions.

## Key Concepts
- `useReducer` for complex state
- `useMemo` for sorted/filtered data
- Performance optimization

## Solution

```jsx
import { useState, useMemo } from 'react';

function DataTable({ data, columns, pageSize = 10 }) {
  const [sortConfig, setSortConfig] = useState({ key: null, direction: 'asc' });
  const [filterText, setFilterText] = useState('');
  const [currentPage, setCurrentPage] = useState(1);
  const [selectedRows, setSelectedRows] = useState(new Set());

  // Sort and filter data
  const processedData = useMemo(() => {
    let result = [...data];

    // Filter
    if (filterText) {
      result = result.filter(row =>
        Object.values(row).some(value =>
          String(value).toLowerCase().includes(filterText.toLowerCase())
        )
      );
    }

    // Sort
    if (sortConfig.key) {
      result.sort((a, b) => {
        const aVal = a[sortConfig.key];
        const bVal = b[sortConfig.key];
        
        if (aVal < bVal) return sortConfig.direction === 'asc' ? -1 : 1;
        if (aVal > bVal) return sortConfig.direction === 'asc' ? 1 : -1;
        return 0;
      });
    }

    return result;
  }, [data, sortConfig, filterText]);

  // Pagination
  const totalPages = Math.ceil(processedData.length / pageSize);
  const paginatedData = useMemo(() => {
    const start = (currentPage - 1) * pageSize;
    return processedData.slice(start, start + pageSize);
  }, [processedData, currentPage, pageSize]);

  const handleSort = (key) => {
    setSortConfig(prev => ({
      key,
      direction: prev.key === key && prev.direction === 'asc' ? 'desc' : 'asc'
    }));
  };

  const toggleRowSelection = (id) => {
    setSelectedRows(prev => {
      const newSet = new Set(prev);
      if (newSet.has(id)) {
        newSet.delete(id);
      } else {
        newSet.add(id);
      }
      return newSet;
    });
  };

  const toggleSelectAll = () => {
    if (selectedRows.size === paginatedData.length) {
      setSelectedRows(new Set());
    } else {
      setSelectedRows(new Set(paginatedData.map(row => row.id)));
    }
  };

  const handleBulkDelete = () => {
    console.log('Delete rows:', Array.from(selectedRows));
    setSelectedRows(new Set());
  };

  return (
    <div className="data-table">
      {/* Controls */}
      <div className="controls">
        <input
          type="text"
          placeholder="Search..."
          value={filterText}
          onChange={(e) => {
            setFilterText(e.target.value);
            setCurrentPage(1);
          }}
        />
        {selectedRows.size > 0 && (
          <button onClick={handleBulkDelete}>
            Delete Selected ({selectedRows.size})
          </button>
        )}
      </div>

      {/* Table */}
      <table>
        <thead>
          <tr>
            <th>
              <input
                type="checkbox"
                checked={selectedRows.size === paginatedData.length}
                onChange={toggleSelectAll}
              />
            </th>
            {columns.map(col => (
              <th key={col.key} onClick={() => handleSort(col.key)}>
                {col.label}
                {sortConfig.key === col.key && (
                  <span>{sortConfig.direction === 'asc' ? ' ↑' : ' ↓'}</span>
                )}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {paginatedData.map(row => (
            <tr key={row.id}>
              <td>
                <input
                  type="checkbox"
                  checked={selectedRows.has(row.id)}
                  onChange={() => toggleRowSelection(row.id)}
                />
              </td>
              {columns.map(col => (
                <td key={col.key}>{row[col.key]}</td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>

      {/* Pagination */}
      <div className="pagination">
        <button
          onClick={() => setCurrentPage(p => Math.max(1, p - 1))}
          disabled={currentPage === 1}
        >
          Previous
        </button>
        <span>
          Page {currentPage} of {totalPages}
        </span>
        <button
          onClick={() => setCurrentPage(p => Math.min(totalPages, p + 1))}
          disabled={currentPage === totalPages}
        >
          Next
        </button>
      </div>
    </div>
  );
}

export default DataTable;
```

## Usage

```jsx
const data = [
  { id: 1, name: 'John', age: 30, email: 'john@example.com' },
  { id: 2, name: 'Jane', age: 25, email: 'jane@example.com' },
  // ... more rows
];

const columns = [
  { key: 'name', label: 'Name' },
  { key: 'age', label: 'Age' },
  { key: 'email', label: 'Email' }
];

<DataTable data={data} columns={columns} pageSize={10} />
```

## CSS

```css
.data-table {
  padding: 1rem;
}

.controls {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}

.controls input {
  padding: 0.5rem;
  flex: 1;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  padding: 0.75rem;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

th {
  cursor: pointer;
  user-select: none;
}

th:hover {
  background: #f5f5f5;
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
  margin-top: 1rem;
}
```


## MANG-Level Expectations
- **Performance**: useMemo for expensive computations, virtualization for large datasets
- **State Management**: useReducer for complex state, proper immutability
- **Features**: Multi-column sort, advanced filters, column resize, inline editing
- **Accessibility**: Keyboard navigation, ARIA attributes, screen reader support
- **Extensibility**: Plugin architecture, custom cell renderers

## Enhanced Solution with Comments

```jsx
/**
 * TRICKY PART #1: useMemo for performance
 * - Sorting and filtering can be expensive for large datasets
 * - MANG expectation: Explain when to use useMemo vs when it's premature optimization
 * - Rule of thumb: Use for computations > 1000 items or complex operations
 */
const processedData = useMemo(() => {
  let result = [...data]; // IMPORTANT: Create new array, don't mutate

  // Filter - O(n) operation
  if (filterText) {
    result = result.filter(row =>
      Object.values(row).some(value =>
        String(value).toLowerCase().includes(filterText.toLowerCase())
      )
    );
  }

  // Sort - O(n log n) operation
  if (sortConfig.key) {
    result.sort((a, b) => {
      const aVal = a[sortConfig.key];
      const bVal = b[sortConfig.key];
      
      // TRICKY: Handle different data types
      // MANG might ask: "What about dates, numbers as strings?"
      if (typeof aVal === 'number' && typeof bVal === 'number') {
        return sortConfig.direction === 'asc' ? aVal - bVal : bVal - aVal;
      }
      
      if (aVal < bVal) return sortConfig.direction === 'asc' ? -1 : 1;
      if (aVal > bVal) return sortConfig.direction === 'asc' ? 1 : -1;
      return 0;
    });
  }

  return result;
}, [data, sortConfig, filterText]);

/**
 * TRICKY PART #2: Pagination with useMemo
 * - Depends on processedData, so needs separate useMemo
 * - MANG expectation: Understand dependency chains
 */
const paginatedData = useMemo(() => {
  const start = (currentPage - 1) * pageSize;
  return processedData.slice(start, start + pageSize);
}, [processedData, currentPage, pageSize]);
```

## Test Cases

```jsx
import { render, screen, fireEvent, within } from '@testing-library/react';
import DataTable from './DataTable';

describe('DataTable', () => {
  const mockData = [
    { id: 1, name: 'Alice', age: 30, email: 'alice@example.com' },
    { id: 2, name: 'Bob', age: 25, email: 'bob@example.com' },
    { id: 3, name: 'Charlie', age: 35, email: 'charlie@example.com' }
  ];

  const mockColumns = [
    { key: 'name', label: 'Name' },
    { key: 'age', label: 'Age' },
    { key: 'email', label: 'Email' }
  ];

  test('renders table with data', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    expect(screen.getByText('Alice')).toBeInTheDocument();
    expect(screen.getByText('Bob')).toBeInTheDocument();
    expect(screen.getByText('Charlie')).toBeInTheDocument();
  });

  test('sorts by column ascending', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    const nameHeader = screen.getByText('Name');
    fireEvent.click(nameHeader);

    const rows = screen.getAllByRole('row');
    expect(within(rows[1]).getByText('Alice')).toBeInTheDocument();
    expect(within(rows[2]).getByText('Bob')).toBeInTheDocument();
    expect(within(rows[3]).getByText('Charlie')).toBeInTheDocument();
  });

  test('sorts by column descending on second click', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    const nameHeader = screen.getByText('Name');
    fireEvent.click(nameHeader);
    fireEvent.click(nameHeader);

    const rows = screen.getAllByRole('row');
    expect(within(rows[1]).getByText('Charlie')).toBeInTheDocument();
    expect(within(rows[2]).getByText('Bob')).toBeInTheDocument();
    expect(within(rows[3]).getByText('Alice')).toBeInTheDocument();
  });

  test('filters data by search text', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    const searchInput = screen.getByPlaceholderText('Search...');
    fireEvent.change(searchInput, { target: { value: 'alice' } });

    expect(screen.getByText('Alice')).toBeInTheDocument();
    expect(screen.queryByText('Bob')).not.toBeInTheDocument();
    expect(screen.queryByText('Charlie')).not.toBeInTheDocument();
  });

  test('resets to page 1 when filtering', () => {
    const largeData = Array.from({ length: 30 }, (_, i) => ({
      id: i,
      name: `Person ${i}`,
      age: 20 + i,
      email: `person${i}@example.com`
    }));

    render(<DataTable data={largeData} columns={mockColumns} pageSize={10} />);
    
    // Go to page 2
    fireEvent.click(screen.getByText('Next'));
    expect(screen.getByText(/Page 2/)).toBeInTheDocument();

    // Filter should reset to page 1
    const searchInput = screen.getByPlaceholderText('Search...');
    fireEvent.change(searchInput, { target: { value: 'Person 5' } });

    expect(screen.getByText(/Page 1/)).toBeInTheDocument();
  });

  test('selects individual rows', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    const checkboxes = screen.getAllByRole('checkbox');
    fireEvent.click(checkboxes[1]); // First data row (index 0 is select all)

    expect(checkboxes[1]).toBeChecked();
  });

  test('selects all rows', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    const checkboxes = screen.getAllByRole('checkbox');
    const selectAllCheckbox = checkboxes[0];
    
    fireEvent.click(selectAllCheckbox);

    checkboxes.slice(1).forEach(checkbox => {
      expect(checkbox).toBeChecked();
    });
  });

  test('deselects all when clicking select all again', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    const selectAllCheckbox = screen.getAllByRole('checkbox')[0];
    
    fireEvent.click(selectAllCheckbox);
    fireEvent.click(selectAllCheckbox);

    const checkboxes = screen.getAllByRole('checkbox');
    checkboxes.forEach(checkbox => {
      expect(checkbox).not.toBeChecked();
    });
  });

  test('shows bulk delete button when rows selected', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    expect(screen.queryByText(/Delete Selected/)).not.toBeInTheDocument();

    const checkboxes = screen.getAllByRole('checkbox');
    fireEvent.click(checkboxes[1]);

    expect(screen.getByText(/Delete Selected \(1\)/)).toBeInTheDocument();
  });

  test('paginates data correctly', () => {
    const largeData = Array.from({ length: 25 }, (_, i) => ({
      id: i,
      name: `Person ${i}`,
      age: 20,
      email: `person${i}@example.com`
    }));

    render(<DataTable data={largeData} columns={mockColumns} pageSize={10} />);
    
    expect(screen.getByText('Person 0')).toBeInTheDocument();
    expect(screen.queryByText('Person 10')).not.toBeInTheDocument();

    fireEvent.click(screen.getByText('Next'));

    expect(screen.queryByText('Person 0')).not.toBeInTheDocument();
    expect(screen.getByText('Person 10')).toBeInTheDocument();
  });

  test('disables previous button on first page', () => {
    render(<DataTable data={mockData} columns={mockColumns} />);
    
    const prevButton = screen.getByText('Previous');
    expect(prevButton).toBeDisabled();
  });

  test('disables next button on last page', () => {
    render(<DataTable data={mockData} columns={mockColumns} pageSize={10} />);
    
    const nextButton = screen.getByText('Next');
    expect(nextButton).toBeDisabled();
  });

  test('shows correct page numbers', () => {
    const largeData = Array.from({ length: 25 }, (_, i) => ({
      id: i,
      name: `Person ${i}`,
      age: 20,
      email: `person${i}@example.com`
    }));

    render(<DataTable data={largeData} columns={mockColumns} pageSize={10} />);
    
    expect(screen.getByText(/Page 1 of 3/)).toBeInTheDocument();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you handle 100,000 rows?**
```jsx
// 1. Server-side pagination, sorting, filtering
// 2. Virtual scrolling (react-window, react-virtual)
// 3. Lazy loading with infinite scroll
// 4. Web Workers for heavy computations
// 5. IndexedDB for client-side caching

import { useVirtual } from 'react-virtual';

const rowVirtualizer = useVirtual({
  size: filteredData.length,
  parentRef: tableRef,
  estimateSize: useCallback(() => 50, []),
  overscan: 5
});
```

**Q: How would you add column resizing?**
```jsx
const [columnWidths, setColumnWidths] = useState({});

const handleMouseDown = (columnKey) => (e) => {
  const startX = e.clientX;
  const startWidth = e.target.parentElement.offsetWidth;

  const handleMouseMove = (e) => {
    const newWidth = startWidth + (e.clientX - startX);
    setColumnWidths(prev => ({ ...prev, [columnKey]: newWidth }));
  };

  const handleMouseUp = () => {
    document.removeEventListener('mousemove', handleMouseMove);
    document.removeEventListener('mouseup', handleMouseUp);
  };

  document.addEventListener('mousemove', handleMouseMove);
  document.addEventListener('mouseup', handleMouseUp);
};
```

**Q: How would you add inline editing?**
```jsx
const [editingCell, setEditingCell] = useState(null);

const handleCellClick = (rowId, columnKey) => {
  setEditingCell({ rowId, columnKey });
};

const handleCellSave = (rowId, columnKey, newValue) => {
  // Update data
  onDataChange?.(data.map(row => 
    row.id === rowId ? { ...row, [columnKey]: newValue } : row
  ));
  setEditingCell(null);
};
```

**Q: How would you implement multi-column sort?**
```jsx
const [sortConfigs, setSortConfigs] = useState([]);

const handleSort = (key, shiftKey) => {
  if (shiftKey) {
    // Add to sort stack
    setSortConfigs(prev => [...prev, { key, direction: 'asc' }]);
  } else {
    // Replace sort stack
    setSortConfigs([{ key, direction: 'asc' }]);
  }
};

// Apply multiple sorts
result.sort((a, b) => {
  for (const config of sortConfigs) {
    const comparison = compare(a[config.key], b[config.key], config.direction);
    if (comparison !== 0) return comparison;
  }
  return 0;
});
```

**Q: How would you make this accessible?**
```jsx
<table role="grid" aria-label="Data table">
  <thead>
    <tr role="row">
      <th
        role="columnheader"
        aria-sort={sortConfig.key === col.key ? sortConfig.direction : 'none'}
        tabIndex={0}
        onKeyPress={(e) => e.key === 'Enter' && handleSort(col.key)}
      >
        {col.label}
      </th>
    </tr>
  </thead>
</table>

// Announce changes to screen readers
<div role="status" aria-live="polite" className="sr-only">
  Showing {paginatedData.length} of {processedData.length} results
</div>
```

## Performance Optimization Tips

1. **useMemo Dependencies**
   - Only include what actually affects the computation
   - Avoid object/array literals in dependencies

2. **Virtualization Threshold**
   - < 100 rows: No virtualization needed
   - 100-1000 rows: Consider useMemo
   - > 1000 rows: Use virtualization

3. **Debounce Search**
   - Add 300ms debounce to filter input
   - Prevents excessive re-renders

4. **Pagination Strategy**
   - Client-side: Good for < 10,000 rows
   - Server-side: Required for > 10,000 rows

## Edge Cases Handled
- Empty data set
- Single page of data (disable pagination)
- Filter with no results
- Sort stability (equal values maintain order)
- Select all only selects current page
- Reset page on filter change
- Immutable state updates
- Type-aware sorting (numbers vs strings)
