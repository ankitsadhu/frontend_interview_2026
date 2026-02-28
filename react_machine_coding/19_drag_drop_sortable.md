# Drag & Drop Sortable List

## Problem
Create a sortable list where items can be reordered via drag and drop with visual drop indicators.

## Key Concepts
- Drag and Drop API (`onDragStart`, `onDragOver`, `onDrop`)
- `dataTransfer` for passing data
- State management for list reordering
- Visual feedback during drag

## MANG-Level Expectations
- **UX**: Smooth animations, visual feedback, drop indicators
- **Accessibility**: Keyboard support for reordering, ARIA attributes
- **Performance**: Avoid unnecessary re-renders, optimize for large lists
- **Features**: Multi-select drag, nested lists, drag handles
- **Robustness**: Handle edge cases, touch support

## Solution

```jsx
import { useState } from 'react';

/**
 * TRICKY PART #1: Drag and Drop state management
 * - Track dragged item and drop target
 * - MANG expectation: Explain the drag/drop flow
 * - onDragStart → onDragOver → onDrop → reorder
 */
function SortableList({ initialItems, onReorder }) {
  const [items, setItems] = useState(initialItems);
  const [draggedIndex, setDraggedIndex] = useState(null);
  const [dragOverIndex, setDragOverIndex] = useState(null);

  /**
   * TRICKY PART #2: onDragStart
   * - Store the index of dragged item
   * - Set dataTransfer for compatibility
   * - MANG will ask: "Why use dataTransfer?"
   * - Required by HTML5 Drag and Drop API
   */
  const handleDragStart = (e, index) => {
    setDraggedIndex(index);
    e.dataTransfer.effectAllowed = 'move';
    // Store data for compatibility with some browsers
    e.dataTransfer.setData('text/html', e.currentTarget);
    
    // Optional: Set custom drag image
    // e.dataTransfer.setDragImage(e.currentTarget, 0, 0);
  };

  /**
   * TRICKY PART #3: onDragOver
   * - Must call preventDefault() to allow drop
   * - CRITICAL: Without this, onDrop won't fire
   * - Track which item is being hovered over
   */
  const handleDragOver = (e, index) => {
    e.preventDefault(); // CRITICAL for drop to work
    e.dataTransfer.dropEffect = 'move';
    
    if (index !== draggedIndex) {
      setDragOverIndex(index);
    }
  };

  const handleDragEnter = (e, index) => {
    e.preventDefault();
    if (index !== draggedIndex) {
      setDragOverIndex(index);
    }
  };

  const handleDragLeave = () => {
    setDragOverIndex(null);
  };

  /**
   * TRICKY PART #4: onDrop - Reorder logic
   * - Calculate new position
   * - Reorder array immutably
   * - MANG expectation: Efficient array manipulation
   */
  const handleDrop = (e, dropIndex) => {
    e.preventDefault();
    
    if (draggedIndex === null || draggedIndex === dropIndex) {
      setDraggedIndex(null);
      setDragOverIndex(null);
      return;
    }

    // Create new array with reordered items
    const newItems = [...items];
    const [draggedItem] = newItems.splice(draggedIndex, 1);
    newItems.splice(dropIndex, 0, draggedItem);

    setItems(newItems);
    onReorder?.(newItems);
    
    // Reset state
    setDraggedIndex(null);
    setDragOverIndex(null);
  };

  const handleDragEnd = () => {
    setDraggedIndex(null);
    setDragOverIndex(null);
  };

  /**
   * TRICKY PART #5: Keyboard accessibility
   * - Allow reordering with keyboard
   * - MANG will ask: "How do you make this accessible?"
   * - Alt+Up/Down to move items
   */
  const handleKeyDown = (e, index) => {
    if (e.altKey) {
      if (e.key === 'ArrowUp' && index > 0) {
        e.preventDefault();
        const newItems = [...items];
        [newItems[index - 1], newItems[index]] = [newItems[index], newItems[index - 1]];
        setItems(newItems);
        onReorder?.(newItems);
      } else if (e.key === 'ArrowDown' && index < items.length - 1) {
        e.preventDefault();
        const newItems = [...items];
        [newItems[index], newItems[index + 1]] = [newItems[index + 1], newItems[index]];
        setItems(newItems);
        onReorder?.(newItems);
      }
    }
  };

  return (
    <div className="sortable-list" role="list">
      {items.map((item, index) => (
        <div
          key={item.id}
          draggable
          onDragStart={(e) => handleDragStart(e, index)}
          onDragOver={(e) => handleDragOver(e, index)}
          onDragEnter={(e) => handleDragEnter(e, index)}
          onDragLeave={handleDragLeave}
          onDrop={(e) => handleDrop(e, index)}
          onDragEnd={handleDragEnd}
          onKeyDown={(e) => handleKeyDown(e, index)}
          tabIndex={0}
          className={`sortable-item ${
            draggedIndex === index ? 'dragging' : ''
          } ${
            dragOverIndex === index ? 'drag-over' : ''
          }`}
          role="listitem"
          aria-grabbed={draggedIndex === index}
        >
          <div className="drag-handle" aria-label="Drag to reorder">
            ⋮⋮
          </div>
          <div className="item-content">
            {item.content || item.name || item.title}
          </div>
        </div>
      ))}
    </div>
  );
}

export default SortableList;
```

## CSS

```css
.sortable-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 1rem;
}

.sortable-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1rem;
  background: white;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  cursor: move;
  transition: all 0.2s ease;
  user-select: none;
}

.sortable-item:hover {
  border-color: #007bff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.sortable-item:focus {
  outline: 2px solid #007bff;
  outline-offset: 2px;
}

.sortable-item.dragging {
  opacity: 0.5;
  transform: scale(0.95);
  cursor: grabbing;
}

.sortable-item.drag-over {
  border-color: #28a745;
  border-style: dashed;
  background: #f0f8ff;
}

/* Drop indicator line */
.sortable-item.drag-over::before {
  content: '';
  position: absolute;
  top: -4px;
  left: 0;
  right: 0;
  height: 4px;
  background: #28a745;
  border-radius: 2px;
}

.drag-handle {
  font-size: 1.25rem;
  color: #999;
  cursor: grab;
  line-height: 1;
}

.drag-handle:active {
  cursor: grabbing;
}

.item-content {
  flex: 1;
  font-size: 1rem;
}

/* Animation for reordering */
@keyframes slideIn {
  from {
    transform: translateY(-10px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.sortable-item {
  animation: slideIn 0.2s ease-out;
}
```

## Usage

```jsx
const initialItems = [
  { id: 1, name: 'Item 1' },
  { id: 2, name: 'Item 2' },
  { id: 3, name: 'Item 3' },
  { id: 4, name: 'Item 4' }
];

function App() {
  const handleReorder = (newItems) => {
    console.log('New order:', newItems);
    // Save to backend, update state, etc.
  };

  return (
    <SortableList
      initialItems={initialItems}
      onReorder={handleReorder}
    />
  );
}
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import SortableList from './SortableList';

const mockItems = [
  { id: 1, name: 'Item 1' },
  { id: 2, name: 'Item 2' },
  { id: 3, name: 'Item 3' }
];

describe('SortableList', () => {
  test('renders all items', () => {
    render(<SortableList initialItems={mockItems} />);
    
    expect(screen.getByText('Item 1')).toBeInTheDocument();
    expect(screen.getByText('Item 2')).toBeInTheDocument();
    expect(screen.getByText('Item 3')).toBeInTheDocument();
  });

  test('items are draggable', () => {
    render(<SortableList initialItems={mockItems} />);
    
    const items = screen.getAllByRole('listitem');
    expect(items[0]).toHaveAttribute('draggable', 'true');
  });

  test('adds dragging class on drag start', () => {
    render(<SortableList initialItems={mockItems} />);
    
    const item = screen.getByText('Item 1').closest('.sortable-item');
    
    fireEvent.dragStart(item);
    
    expect(item).toHaveClass('dragging');
  });

  test('adds drag-over class on drag over', () => {
    render(<SortableList initialItems={mockItems} />);
    
    const item1 = screen.getByText('Item 1').closest('.sortable-item');
    const item2 = screen.getByText('Item 2').closest('.sortable-item');
    
    fireEvent.dragStart(item1);
    fireEvent.dragOver(item2);
    
    expect(item2).toHaveClass('drag-over');
  });

  test('reorders items on drop', () => {
    const handleReorder = jest.fn();
    render(<SortableList initialItems={mockItems} onReorder={handleReorder} />);
    
    const item1 = screen.getByText('Item 1').closest('.sortable-item');
    const item3 = screen.getByText('Item 3').closest('.sortable-item');
    
    fireEvent.dragStart(item1);
    fireEvent.dragOver(item3);
    fireEvent.drop(item3);
    
    expect(handleReorder).toHaveBeenCalled();
    const newOrder = handleReorder.mock.calls[0][0];
    expect(newOrder[0].name).toBe('Item 2');
    expect(newOrder[1].name).toBe('Item 3');
    expect(newOrder[2].name).toBe('Item 1');
  });

  test('removes dragging class on drag end', () => {
    render(<SortableList initialItems={mockItems} />);
    
    const item = screen.getByText('Item 1').closest('.sortable-item');
    
    fireEvent.dragStart(item);
    expect(item).toHaveClass('dragging');
    
    fireEvent.dragEnd(item);
    expect(item).not.toHaveClass('dragging');
  });

  test('does not reorder when dropping on same position', () => {
    const handleReorder = jest.fn();
    render(<SortableList initialItems={mockItems} onReorder={handleReorder} />);
    
    const item1 = screen.getByText('Item 1').closest('.sortable-item');
    
    fireEvent.dragStart(item1);
    fireEvent.drop(item1);
    
    expect(handleReorder).not.toHaveBeenCalled();
  });

  test('moves item up with Alt+ArrowUp', () => {
    const handleReorder = jest.fn();
    render(<SortableList initialItems={mockItems} onReorder={handleReorder} />);
    
    const item2 = screen.getByText('Item 2').closest('.sortable-item');
    
    fireEvent.keyDown(item2, { key: 'ArrowUp', altKey: true });
    
    expect(handleReorder).toHaveBeenCalled();
    const newOrder = handleReorder.mock.calls[0][0];
    expect(newOrder[0].name).toBe('Item 2');
    expect(newOrder[1].name).toBe('Item 1');
  });

  test('moves item down with Alt+ArrowDown', () => {
    const handleReorder = jest.fn();
    render(<SortableList initialItems={mockItems} onReorder={handleReorder} />);
    
    const item2 = screen.getByText('Item 2').closest('.sortable-item');
    
    fireEvent.keyDown(item2, { key: 'ArrowDown', altKey: true });
    
    expect(handleReorder).toHaveBeenCalled();
    const newOrder = handleReorder.mock.calls[0][0];
    expect(newOrder[1].name).toBe('Item 3');
    expect(newOrder[2].name).toBe('Item 2');
  });

  test('does not move first item up', () => {
    const handleReorder = jest.fn();
    render(<SortableList initialItems={mockItems} onReorder={handleReorder} />);
    
    const item1 = screen.getByText('Item 1').closest('.sortable-item');
    
    fireEvent.keyDown(item1, { key: 'ArrowUp', altKey: true });
    
    expect(handleReorder).not.toHaveBeenCalled();
  });

  test('does not move last item down', () => {
    const handleReorder = jest.fn();
    render(<SortableList initialItems={mockItems} onReorder={handleReorder} />);
    
    const item3 = screen.getByText('Item 3').closest('.sortable-item');
    
    fireEvent.keyDown(item3, { key: 'ArrowDown', altKey: true });
    
    expect(handleReorder).not.toHaveBeenCalled();
  });

  test('has correct ARIA attributes', () => {
    render(<SortableList initialItems={mockItems} />);
    
    const list = screen.getByRole('list');
    expect(list).toBeInTheDocument();
    
    const items = screen.getAllByRole('listitem');
    expect(items).toHaveLength(3);
    
    fireEvent.dragStart(items[0]);
    expect(items[0]).toHaveAttribute('aria-grabbed', 'true');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add touch support for mobile?**
```jsx
const [touchStartY, setTouchStartY] = useState(0);
const [touchCurrentY, setTouchCurrentY] = useState(0);

const handleTouchStart = (e, index) => {
  setDraggedIndex(index);
  setTouchStartY(e.touches[0].clientY);
};

const handleTouchMove = (e) => {
  if (draggedIndex === null) return;
  
  setTouchCurrentY(e.touches[0].clientY);
  
  // Calculate which item is under touch point
  const elements = document.elementsFromPoint(
    e.touches[0].clientX,
    e.touches[0].clientY
  );
  
  const targetItem = elements.find(el => 
    el.classList.contains('sortable-item')
  );
  
  if (targetItem) {
    const index = Array.from(targetItem.parentNode.children).indexOf(targetItem);
    setDragOverIndex(index);
  }
};

const handleTouchEnd = () => {
  if (draggedIndex !== null && dragOverIndex !== null) {
    // Perform reorder
    const newItems = [...items];
    const [draggedItem] = newItems.splice(draggedIndex, 1);
    newItems.splice(dragOverIndex, 0, draggedItem);
    setItems(newItems);
  }
  
  setDraggedIndex(null);
  setDragOverIndex(null);
};
```

**Q: How would you add drag handles (only drag from specific area)?**
```jsx
// Already implemented in the solution!
// The drag handle is the ⋮⋮ icon

// To make only the handle draggable:
<div
  className="sortable-item"
  draggable={false} // Make item not draggable
>
  <div
    className="drag-handle"
    draggable={true} // Only handle is draggable
    onDragStart={(e) => handleDragStart(e, index)}
  >
    ⋮⋮
  </div>
  <div className="item-content">{item.name}</div>
</div>
```

**Q: How would you add multi-select drag?**
```jsx
const [selectedItems, setSelectedItems] = useState(new Set());

const handleItemClick = (e, index) => {
  if (e.ctrlKey || e.metaKey) {
    setSelectedItems(prev => {
      const newSet = new Set(prev);
      if (newSet.has(index)) {
        newSet.delete(index);
      } else {
        newSet.add(index);
      }
      return newSet;
    });
  }
};

const handleDrop = (e, dropIndex) => {
  if (selectedItems.size > 0) {
    // Move all selected items
    const newItems = [...items];
    const itemsToMove = Array.from(selectedItems)
      .sort((a, b) => a - b)
      .map(i => newItems[i]);
    
    // Remove selected items
    const remaining = newItems.filter((_, i) => !selectedItems.has(i));
    
    // Insert at drop position
    remaining.splice(dropIndex, 0, ...itemsToMove);
    
    setItems(remaining);
    setSelectedItems(new Set());
  }
};
```

**Q: How would you add nested sortable lists?**
```jsx
function NestedSortableList({ items, level = 0 }) {
  return (
    <div className="sortable-list" style={{ marginLeft: `${level * 20}px` }}>
      {items.map((item, index) => (
        <div key={item.id}>
          <div className="sortable-item" draggable>
            {item.name}
          </div>
          {item.children && (
            <NestedSortableList items={item.children} level={level + 1} />
          )}
        </div>
      ))}
    </div>
  );
}
```

**Q: How would you optimize for large lists (1000+ items)?**
```jsx
// 1. Use react-window for virtualization
import { FixedSizeList } from 'react-window';

// 2. Memoize items
const SortableItem = React.memo(({ item, index, onDragStart, onDrop }) => (
  <div
    draggable
    onDragStart={(e) => onDragStart(e, index)}
    onDrop={(e) => onDrop(e, index)}
  >
    {item.name}
  </div>
));

// 3. Use useCallback for handlers
const handleDragStart = useCallback((e, index) => {
  setDraggedIndex(index);
}, []);

// 4. Debounce drag over events
const debouncedDragOver = useMemo(
  () => debounce((index) => setDragOverIndex(index), 50),
  []
);
```

**Q: How would you persist the order to backend?**
```jsx
const handleReorder = async (newItems) => {
  setItems(newItems);
  
  // Optimistic update
  try {
    await fetch('/api/items/reorder', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        items: newItems.map((item, index) => ({
          id: item.id,
          position: index
        }))
      })
    });
  } catch (error) {
    // Revert on error
    setItems(initialItems);
    console.error('Failed to save order:', error);
  }
};
```

## Performance Considerations

1. **Avoid Re-renders**
   - Use React.memo for list items
   - useCallback for event handlers
   - Only update dragged and drop target items

2. **Large Lists**
   - Implement virtualization (react-window)
   - Debounce dragOver events
   - Use CSS transforms instead of re-rendering

3. **Touch Devices**
   - Add touch event handlers
   - Provide visual feedback
   - Handle scroll during drag

## Edge Cases Handled
- Drag and drop with visual feedback
- Immutable array reordering
- Keyboard accessibility (Alt+Arrow keys)
- Drag handle for better UX
- Drop indicator styling
- Prevent drop on same position
- ARIA attributes for screen readers
- Boundary checks (first/last item)
- dataTransfer for browser compatibility
- preventDefault in dragOver (critical!)
- State cleanup on drag end
