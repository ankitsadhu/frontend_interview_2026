# Undo/Redo with History Stack

## Problem
Implement undo/redo functionality with keyboard shortcuts (Ctrl+Z / Ctrl+Y).

## Key Concepts
- `useReducer` with history stack
- Keyboard event handling
- State management pattern

## Solution

```jsx
import { useReducer, useEffect } from 'react';

function useUndoRedo(initialState) {
  const [state, dispatch] = useReducer(
    (state, action) => {
      switch (action.type) {
        case 'SET':
          return {
            past: [...state.past, state.present],
            present: action.payload,
            future: []
          };
        case 'UNDO':
          if (state.past.length === 0) return state;
          return {
            past: state.past.slice(0, -1),
            present: state.past[state.past.length - 1],
            future: [state.present, ...state.future]
          };
        case 'REDO':
          if (state.future.length === 0) return state;
          return {
            past: [...state.past, state.present],
            present: state.future[0],
            future: state.future.slice(1)
          };
        default:
          return state;
      }
    },
    {
      past: [],
      present: initialState,
      future: []
    }
  );

  const canUndo = state.past.length > 0;
  const canRedo = state.future.length > 0;

  const set = (newState) => dispatch({ type: 'SET', payload: newState });
  const undo = () => dispatch({ type: 'UNDO' });
  const redo = () => dispatch({ type: 'REDO' });

  return { state: state.present, set, undo, redo, canUndo, canRedo };
}

function DrawingApp() {
  const { state, set, undo, redo, canUndo, canRedo } = useUndoRedo('');

  useEffect(() => {
    const handleKeyDown = (e) => {
      if (e.ctrlKey || e.metaKey) {
        if (e.key === 'z') {
          e.preventDefault();
          undo();
        } else if (e.key === 'y') {
          e.preventDefault();
          redo();
        }
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [undo, redo]);

  return (
    <div className="drawing-app">
      <div className="toolbar">
        <button onClick={undo} disabled={!canUndo}>
          ↶ Undo (Ctrl+Z)
        </button>
        <button onClick={redo} disabled={!canRedo}>
          ↷ Redo (Ctrl+Y)
        </button>
      </div>

      <textarea
        value={state}
        onChange={(e) => set(e.target.value)}
        placeholder="Type something..."
        rows={10}
      />
    </div>
  );
}

export default DrawingApp;
```

## CSS

```css
.drawing-app {
  max-width: 600px;
  margin: 2rem auto;
  padding: 1rem;
}

.toolbar {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.toolbar button {
  padding: 0.5rem 1rem;
  cursor: pointer;
}

.toolbar button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

textarea {
  width: 100%;
  padding: 0.5rem;
  font-size: 1rem;
  font-family: monospace;
}
```

## Edge Cases Handled
- Keyboard shortcuts (Ctrl+Z, Ctrl+Y)
- Disabled buttons when no history
- Clear future on new action
- Works with any state type


## MANG-Level Expectations
- **State Management**: useReducer with history stack pattern
- **Keyboard Shortcuts**: Global event listeners, cleanup
- **Performance**: Limit history size, efficient state updates
- **Extensibility**: Works with any state type, serializable
- **UX**: Visual feedback, disabled states, history navigation

## Enhanced Solution with Comments

```jsx
/**
 * TRICKY PART #1: History stack pattern
 * - Three arrays: past, present, future
 * - MANG expectation: Explain the data structure choice
 * - Alternative: Could use single array with pointer
 */
const [state, dispatch] = useReducer(
  (state, action) => {
    switch (action.type) {
      case 'SET':
        // IMPORTANT: Clear future on new action
        return {
          past: [...state.past, state.present],
          present: action.payload,
          future: [] // Clear redo stack
        };
      case 'UNDO':
        if (state.past.length === 0) return state;
        return {
          past: state.past.slice(0, -1), // Remove last
          present: state.past[state.past.length - 1],
          future: [state.present, ...state.future] // Add to future
        };
      case 'REDO':
        if (state.future.length === 0) return state;
        return {
          past: [...state.past, state.present],
          present: state.future[0],
          future: state.future.slice(1)
        };
      default:
        return state;
    }
  },
  {
    past: [],
    present: initialState,
    future: []
  }
);

/**
 * TRICKY PART #2: Keyboard event listeners
 * - Must handle both Ctrl (Windows) and Cmd (Mac)
 * - MANG will ask: "What about cleanup?"
 * - CRITICAL: Remove listener on unmount
 */
useEffect(() => {
  const handleKeyDown = (e) => {
    if (e.ctrlKey || e.metaKey) { // Both Ctrl and Cmd
      if (e.key === 'z') {
        e.preventDefault(); // Prevent browser undo
        undo();
      } else if (e.key === 'y') {
        e.preventDefault();
        redo();
      }
    }
  };

  window.addEventListener('keydown', handleKeyDown);
  return () => window.removeEventListener('keydown', handleKeyDown);
}, [undo, redo]);
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import { renderHook, act } from '@testing-library/react-hooks';
import DrawingApp, { useUndoRedo } from './DrawingApp';

describe('useUndoRedo', () => {
  test('initializes with initial state', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    expect(result.current.state).toBe('initial');
  });

  test('sets new state', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('new state');
    });
    
    expect(result.current.state).toBe('new state');
  });

  test('undoes to previous state', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('state 1');
      result.current.set('state 2');
      result.current.undo();
    });
    
    expect(result.current.state).toBe('state 1');
  });

  test('redoes to next state', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('state 1');
      result.current.undo();
      result.current.redo();
    });
    
    expect(result.current.state).toBe('state 1');
  });

  test('clears future on new action', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('state 1');
      result.current.set('state 2');
      result.current.undo();
      result.current.set('state 3');
    });
    
    expect(result.current.canRedo).toBe(false);
  });

  test('canUndo is false initially', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    expect(result.current.canUndo).toBe(false);
  });

  test('canUndo is true after action', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('new state');
    });
    
    expect(result.current.canUndo).toBe(true);
  });

  test('canRedo is false initially', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    expect(result.current.canRedo).toBe(false);
  });

  test('canRedo is true after undo', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('state 1');
      result.current.undo();
    });
    
    expect(result.current.canRedo).toBe(true);
  });

  test('handles multiple undos', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('state 1');
      result.current.set('state 2');
      result.current.set('state 3');
      result.current.undo();
      result.current.undo();
    });
    
    expect(result.current.state).toBe('state 1');
  });

  test('does not undo beyond initial state', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.undo();
      result.current.undo();
    });
    
    expect(result.current.state).toBe('initial');
  });

  test('does not redo beyond last state', () => {
    const { result } = renderHook(() => useUndoRedo('initial'));
    
    act(() => {
      result.current.set('state 1');
      result.current.redo();
      result.current.redo();
    });
    
    expect(result.current.state).toBe('state 1');
  });
});

describe('DrawingApp keyboard shortcuts', () => {
  test('undoes on Ctrl+Z', () => {
    render(<DrawingApp />);
    const textarea = screen.getByPlaceholderText('Type something...');
    
    fireEvent.change(textarea, { target: { value: 'Hello' } });
    fireEvent.change(textarea, { target: { value: 'Hello World' } });
    
    fireEvent.keyDown(window, { key: 'z', ctrlKey: true });
    
    expect(textarea.value).toBe('Hello');
  });

  test('redoes on Ctrl+Y', () => {
    render(<DrawingApp />);
    const textarea = screen.getByPlaceholderText('Type something...');
    
    fireEvent.change(textarea, { target: { value: 'Hello' } });
    fireEvent.keyDown(window, { key: 'z', ctrlKey: true });
    fireEvent.keyDown(window, { key: 'y', ctrlKey: true });
    
    expect(textarea.value).toBe('Hello');
  });

  test('works with Cmd key on Mac', () => {
    render(<DrawingApp />);
    const textarea = screen.getByPlaceholderText('Type something...');
    
    fireEvent.change(textarea, { target: { value: 'Hello' } });
    fireEvent.keyDown(window, { key: 'z', metaKey: true });
    
    expect(textarea.value).toBe('');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you limit history size to prevent memory issues?**
```jsx
const MAX_HISTORY = 50;

case 'SET':
  return {
    past: [...state.past, state.present].slice(-MAX_HISTORY),
    present: action.payload,
    future: []
  };

// Or use a circular buffer for better performance
```

**Q: How would you add history navigation (jump to specific state)?**
```jsx
const [history, setHistory] = useState([initialState]);
const [currentIndex, setCurrentIndex] = useState(0);

const jumpTo = (index) => {
  setCurrentIndex(index);
};

const currentState = history[currentIndex];

// Render timeline
<div className="timeline">
  {history.map((state, index) => (
    <button
      key={index}
      onClick={() => jumpTo(index)}
      className={index === currentIndex ? 'active' : ''}
    >
      {index}
    </button>
  ))}
</div>
```

**Q: How would you add state diffing (show what changed)?**
```jsx
const getDiff = (oldState, newState) => {
  if (typeof oldState === 'string' && typeof newState === 'string') {
    // Use diff library like diff-match-patch
    const dmp = new DiffMatchPatch();
    const diffs = dmp.diff_main(oldState, newState);
    return diffs;
  }
  
  // For objects, use deep-diff or similar
  return deepDiff(oldState, newState);
};

// Display diff
<div className="diff">
  {getDiff(state.past[state.past.length - 1], state.present).map((change, i) => (
    <span key={i} className={change.type}>
      {change.value}
    </span>
  ))}
</div>
```

**Q: How would you persist history across page reloads?**
```jsx
// Save to localStorage
useEffect(() => {
  localStorage.setItem('history', JSON.stringify({
    past: state.past,
    present: state.present,
    future: state.future
  }));
}, [state]);

// Load on mount
const [state, dispatch] = useReducer(reducer, null, () => {
  const saved = localStorage.getItem('history');
  if (saved) {
    return JSON.parse(saved);
  }
  return {
    past: [],
    present: initialState,
    future: []
  };
});
```

**Q: How would you handle complex state (nested objects)?**
```jsx
// Use immer for immutable updates
import produce from 'immer';

const set = (updater) => {
  dispatch({
    type: 'SET',
    payload: produce(state.present, updater)
  });
};

// Usage
set(draft => {
  draft.user.name = 'John';
  draft.settings.theme = 'dark';
});

// Or use immer's patches for efficient history
import { produceWithPatches } from 'immer';

const [nextState, patches, inversePatches] = produceWithPatches(
  state.present,
  draft => {
    draft.user.name = 'John';
  }
);

// Store patches instead of full state
past: [...state.past, { patches: inversePatches }]
```

**Q: How would you add branching history (like Git)?**
```jsx
// Tree structure instead of linear history
const [historyTree, setHistoryTree] = useState({
  id: 'root',
  state: initialState,
  children: []
});

const [currentNode, setCurrentNode] = useState(historyTree);

const addState = (newState) => {
  const newNode = {
    id: Date.now(),
    state: newState,
    parent: currentNode,
    children: []
  };
  
  currentNode.children.push(newNode);
  setCurrentNode(newNode);
};
```

## Performance Considerations

1. **History Size Limit**
   - Unbounded history can cause memory issues
   - Limit to 50-100 states
   - Use circular buffer for efficiency

2. **State Serialization**
   - Deep cloning can be expensive
   - Use structural sharing (immer)
   - Consider storing diffs instead of full states

3. **Keyboard Event Listeners**
   - Single global listener is efficient
   - Always cleanup on unmount
   - Use event delegation if multiple instances

4. **Re-render Optimization**
   - useReducer is already optimized
   - Memoize canUndo/canRedo if expensive

## Tricky Parts Explained

**1. Why Clear Future on New Action?**
```jsx
// When user makes new change after undo, we lose the "future"
// This is standard undo/redo behavior (like text editors)
case 'SET':
  return {
    past: [...state.past, state.present],
    present: action.payload,
    future: [] // Clear - can't redo after new action
  };
```

**2. Why useReducer Instead of useState?**
```jsx
// useReducer is better for complex state transitions
// Keeps logic centralized and testable
// Easier to add new actions (UNDO_TO_INDEX, CLEAR_HISTORY, etc.)
```

**3. Keyboard Shortcut Dependencies**
```jsx
// TRICKY: undo/redo in dependencies can cause issues
// They're stable references from useCallback
// But if not, could cause listener to re-register frequently
useEffect(() => {
  // ...
}, [undo, redo]); // These should be stable
```

## Edge Cases Handled
- Keyboard shortcuts (Ctrl+Z, Ctrl+Y)
- Both Ctrl (Windows) and Cmd (Mac)
- Disabled buttons when no history
- Clear future on new action
- Works with any state type
- Proper cleanup of event listeners
- Prevent browser default undo behavior
- Boundary checks (can't undo/redo beyond limits)
