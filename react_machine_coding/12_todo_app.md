# Todo App with CRUD and Filters

## Problem
Build a todo app with CRUD operations, filter (all/active/completed), and localStorage persistence.

## Key Concepts
- `useState` or `useReducer` for state
- `useEffect` for localStorage sync
- Filter logic with derived state

## MANG-Level Expectations
- **State Management**: Proper immutable updates, consider useReducer for complex state
- **Performance**: Avoid unnecessary re-renders, use derived state for filters
- **Persistence**: localStorage with error handling, sync across tabs
- **UX**: Optimistic updates, inline editing, keyboard shortcuts
- **Extensibility**: Easy to add features like drag-to-reorder, categories, due dates

## Solution

```jsx
import { useState, useEffect } from 'react';

/**
 * TRICKY PART #1: Lazy initialization with localStorage
 * - Use function form of useState to avoid parsing on every render
 * - MANG expectation: Handle JSON.parse errors gracefully
 */
function TodoApp() {
  const [todos, setTodos] = useState(() => {
    try {
      const saved = localStorage.getItem('todos');
      return saved ? JSON.parse(saved) : [];
    } catch (error) {
      console.error('Failed to parse todos from localStorage:', error);
      return [];
    }
  });
  const [input, setInput] = useState('');
  const [filter, setFilter] = useState('all'); // all, active, completed
  const [editId, setEditId] = useState(null);
  const [editText, setEditText] = useState('');

  /**
   * TRICKY PART #2: localStorage sync
   * - Runs on every todos change
   * - MANG expectation: Handle quota exceeded errors
   * - EXTENSION: Sync across tabs with storage event listener
   */
  useEffect(() => {
    try {
      localStorage.setItem('todos', JSON.stringify(todos));
    } catch (error) {
      console.error('Failed to save todos to localStorage:', error);
      // Could show user notification here
    }
  }, [todos]);

  /**
   * EXTENSION for MANG: Sync across browser tabs
   * useEffect(() => {
   *   const handleStorageChange = (e) => {
   *     if (e.key === 'todos' && e.newValue) {
   *       setTodos(JSON.parse(e.newValue));
   *     }
   *   };
   *   window.addEventListener('storage', handleStorageChange);
   *   return () => window.removeEventListener('storage', handleStorageChange);
   * }, []);
   */

  /**
   * TRICKY PART #3: Immutable state updates
   * - Always create new arrays/objects, never mutate
   * - MANG expectation: Use proper unique IDs (Date.now() can collide)
   * - Production: Use UUID or crypto.randomUUID()
   */
  const addTodo = () => {
    if (!input.trim()) return;
    
    // POTENTIAL ISSUE: Date.now() can create duplicate IDs if called rapidly
    // Better: crypto.randomUUID() or a proper ID generator
    setTodos([
      ...todos,
      { id: Date.now(), text: input, completed: false }
    ]);
    setInput('');
  };

  const toggleTodo = (id) => {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };

  const deleteTodo = (id) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };

  const startEdit = (todo) => {
    setEditId(todo.id);
    setEditText(todo.text);
  };

  const saveEdit = () => {
    if (!editText.trim()) return;
    
    setTodos(todos.map(todo =>
      todo.id === editId ? { ...todo, text: editText } : todo
    ));
    setEditId(null);
    setEditText('');
  };

  const cancelEdit = () => {
    setEditId(null);
    setEditText('');
  };

  /**
   * TRICKY PART #4: Derived state for filtering
   * - Computed on every render, but that's fine for small lists
   * - MANG expectation: For large lists, consider useMemo
   * - Could be asked about performance optimization
   */
  const filteredTodos = todos.filter(todo => {
    if (filter === 'active') return !todo.completed;
    if (filter === 'completed') return todo.completed;
    return true;
  });

  // EXTENSION for MANG: Memoize for large lists
  // const filteredTodos = useMemo(() => {
  //   return todos.filter(todo => {
  //     if (filter === 'active') return !todo.completed;
  //     if (filter === 'completed') return todo.completed;
  //     return true;
  //   });
  // }, [todos, filter]);

  const stats = {
    total: todos.length,
    active: todos.filter(t => !t.completed).length,
    completed: todos.filter(t => t.completed).length
  };

  return (
    <div className="todo-app">
      <h1>Todo App</h1>
      
      {/* Add Todo */}
      <div className="add-todo">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && addTodo()}
          placeholder="What needs to be done?"
        />
        <button onClick={addTodo}>Add</button>
      </div>

      {/* Filters */}
      <div className="filters">
        <button 
          className={filter === 'all' ? 'active' : ''}
          onClick={() => setFilter('all')}
        >
          All ({stats.total})
        </button>
        <button 
          className={filter === 'active' ? 'active' : ''}
          onClick={() => setFilter('active')}
        >
          Active ({stats.active})
        </button>
        <button 
          className={filter === 'completed' ? 'active' : ''}
          onClick={() => setFilter('completed')}
        >
          Completed ({stats.completed})
        </button>
      </div>

      {/* Todo List */}
      <ul className="todo-list">
        {filteredTodos.length === 0 ? (
          <li className="empty">No todos to show</li>
        ) : (
          filteredTodos.map(todo => (
            <li key={todo.id} className={todo.completed ? 'completed' : ''}>
              {editId === todo.id ? (
                <div className="edit-mode">
                  <input
                    type="text"
                    value={editText}
                    onChange={(e) => setEditText(e.target.value)}
                    onKeyPress={(e) => e.key === 'Enter' && saveEdit()}
                    autoFocus
                  />
                  <button onClick={saveEdit}>Save</button>
                  <button onClick={cancelEdit}>Cancel</button>
                </div>
              ) : (
                <div className="view-mode">
                  <input
                    type="checkbox"
                    checked={todo.completed}
                    onChange={() => toggleTodo(todo.id)}
                  />
                  <span className="todo-text">{todo.text}</span>
                  <button onClick={() => startEdit(todo)}>Edit</button>
                  <button onClick={() => deleteTodo(todo.id)}>Delete</button>
                </div>
              )}
            </li>
          ))
        )}
      </ul>
    </div>
  );
}

export default TodoApp;
```

## CSS

```css
.todo-app {
  max-width: 600px;
  margin: 2rem auto;
  padding: 1rem;
}

.add-todo {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.add-todo input {
  flex: 1;
  padding: 0.5rem;
  font-size: 1rem;
}

.filters {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.filters button {
  padding: 0.5rem 1rem;
  cursor: pointer;
}

.filters button.active {
  background: #007bff;
  color: white;
}

.todo-list {
  list-style: none;
  padding: 0;
}

.todo-list li {
  padding: 0.75rem;
  border-bottom: 1px solid #eee;
}

.todo-list li.completed .todo-text {
  text-decoration: line-through;
  opacity: 0.6;
}

.view-mode, .edit-mode {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.todo-text {
  flex: 1;
}

.empty {
  text-align: center;
  color: #999;
  font-style: italic;
}
```

## Edge Cases Handled
- Empty input validation
- localStorage persistence
- Edit mode with save/cancel
- Filter counts
- Empty state message
- Enter key to add/save


## Test Cases

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import TodoApp from './TodoApp';

// Mock localStorage
const localStorageMock = (() => {
  let store = {};
  return {
    getItem: (key) => store[key] || null,
    setItem: (key, value) => { store[key] = value.toString(); },
    clear: () => { store = {}; }
  };
})();

Object.defineProperty(window, 'localStorage', { value: localStorageMock });

describe('TodoApp', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  test('renders todo app', () => {
    render(<TodoApp />);
    expect(screen.getByPlaceholderText(/what needs to be done/i)).toBeInTheDocument();
  });

  test('adds a new todo', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);
    const addButton = screen.getByText('Add');

    fireEvent.change(input, { target: { value: 'New Todo' } });
    fireEvent.click(addButton);

    expect(screen.getByText('New Todo')).toBeInTheDocument();
    expect(input.value).toBe('');
  });

  test('does not add empty todo', () => {
    render(<TodoApp />);
    const addButton = screen.getByText('Add');

    fireEvent.click(addButton);

    expect(screen.queryByRole('listitem')).not.toBeInTheDocument();
  });

  test('toggles todo completion', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);

    fireEvent.change(input, { target: { value: 'Test Todo' } });
    fireEvent.click(screen.getByText('Add'));

    const checkbox = screen.getByRole('checkbox');
    fireEvent.click(checkbox);

    expect(checkbox).toBeChecked();
    expect(screen.getByText('Test Todo').closest('li')).toHaveClass('completed');
  });

  test('deletes a todo', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);

    fireEvent.change(input, { target: { value: 'Delete Me' } });
    fireEvent.click(screen.getByText('Add'));

    const deleteButton = screen.getByText('Delete');
    fireEvent.click(deleteButton);

    expect(screen.queryByText('Delete Me')).not.toBeInTheDocument();
  });

  test('edits a todo', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);

    fireEvent.change(input, { target: { value: 'Original' } });
    fireEvent.click(screen.getByText('Add'));

    const editButton = screen.getByText('Edit');
    fireEvent.click(editButton);

    const editInput = screen.getByDisplayValue('Original');
    fireEvent.change(editInput, { target: { value: 'Updated' } });
    fireEvent.click(screen.getByText('Save'));

    expect(screen.getByText('Updated')).toBeInTheDocument();
    expect(screen.queryByText('Original')).not.toBeInTheDocument();
  });

  test('cancels edit', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);

    fireEvent.change(input, { target: { value: 'Original' } });
    fireEvent.click(screen.getByText('Add'));

    fireEvent.click(screen.getByText('Edit'));
    const editInput = screen.getByDisplayValue('Original');
    fireEvent.change(editInput, { target: { value: 'Changed' } });
    fireEvent.click(screen.getByText('Cancel'));

    expect(screen.getByText('Original')).toBeInTheDocument();
    expect(screen.queryByText('Changed')).not.toBeInTheDocument();
  });

  test('filters todos correctly', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);

    // Add active todo
    fireEvent.change(input, { target: { value: 'Active Todo' } });
    fireEvent.click(screen.getByText('Add'));

    // Add completed todo
    fireEvent.change(input, { target: { value: 'Completed Todo' } });
    fireEvent.click(screen.getByText('Add'));
    const checkboxes = screen.getAllByRole('checkbox');
    fireEvent.click(checkboxes[1]);

    // Test Active filter
    fireEvent.click(screen.getByText(/Active \(1\)/));
    expect(screen.getByText('Active Todo')).toBeInTheDocument();
    expect(screen.queryByText('Completed Todo')).not.toBeInTheDocument();

    // Test Completed filter
    fireEvent.click(screen.getByText(/Completed \(1\)/));
    expect(screen.getByText('Completed Todo')).toBeInTheDocument();
    expect(screen.queryByText('Active Todo')).not.toBeInTheDocument();

    // Test All filter
    fireEvent.click(screen.getByText(/All \(2\)/));
    expect(screen.getByText('Active Todo')).toBeInTheDocument();
    expect(screen.getByText('Completed Todo')).toBeInTheDocument();
  });

  test('persists todos to localStorage', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);

    fireEvent.change(input, { target: { value: 'Persistent Todo' } });
    fireEvent.click(screen.getByText('Add'));

    const saved = JSON.parse(localStorage.getItem('todos'));
    expect(saved).toHaveLength(1);
    expect(saved[0].text).toBe('Persistent Todo');
  });

  test('loads todos from localStorage on mount', () => {
    const initialTodos = [
      { id: 1, text: 'Existing Todo', completed: false }
    ];
    localStorage.setItem('todos', JSON.stringify(initialTodos));

    render(<TodoApp />);

    expect(screen.getByText('Existing Todo')).toBeInTheDocument();
  });

  test('shows empty state message', () => {
    render(<TodoApp />);
    fireEvent.click(screen.getByText(/Active/));
    expect(screen.getByText('No todos to show')).toBeInTheDocument();
  });

  test('adds todo on Enter key', () => {
    render(<TodoApp />);
    const input = screen.getByPlaceholderText(/what needs to be done/i);

    fireEvent.change(input, { target: { value: 'Enter Todo' } });
    fireEvent.keyPress(input, { key: 'Enter', code: 13, charCode: 13 });

    expect(screen.getByText('Enter Todo')).toBeInTheDocument();
  });
});
```

## MANG Interview Follow-ups

**Q: How would you optimize this for 10,000 todos?**
```jsx
// 1. Use virtualization (react-window)
// 2. Memoize filtered todos
// 3. Use useReducer for complex state updates
// 4. Implement pagination or infinite scroll
// 5. Debounce search/filter operations

const filteredTodos = useMemo(() => {
  return todos.filter(todo => {
    if (filter === 'active') return !todo.completed;
    if (filter === 'completed') return todo.completed;
    return true;
  });
}, [todos, filter]);
```

**Q: How would you handle concurrent edits from multiple tabs?**
```jsx
// Use storage event listener
useEffect(() => {
  const handleStorageChange = (e) => {
    if (e.key === 'todos' && e.newValue) {
      const newTodos = JSON.parse(e.newValue);
      // Merge strategy: last write wins, or use timestamps
      setTodos(newTodos);
    }
  };
  
  window.addEventListener('storage', handleStorageChange);
  return () => window.removeEventListener('storage', handleStorageChange);
}, []);
```

**Q: How would you add undo/redo functionality?**
```jsx
// Use history stack pattern (see undo_redo.md)
// Or use immer with patches for efficient history
```

**Q: How would you test localStorage integration?**
```jsx
// Mock localStorage in tests
const localStorageMock = {
  getItem: jest.fn(),
  setItem: jest.fn(),
  clear: jest.fn()
};
global.localStorage = localStorageMock;
```

**Q: What if localStorage quota is exceeded?**
```jsx
try {
  localStorage.setItem('todos', JSON.stringify(todos));
} catch (error) {
  if (error.name === 'QuotaExceededError') {
    // Show user notification
    // Implement cleanup strategy (remove old completed todos)
    // Or move to IndexedDB for larger storage
  }
}
```

## Edge Cases Handled
- Empty input validation
- localStorage persistence with error handling
- Edit mode with save/cancel
- Filter counts update dynamically
- Empty state message per filter
- Enter key to add/save
- Immutable state updates
- Lazy initialization to avoid unnecessary parsing
- JSON parse error handling
