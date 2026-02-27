# File Explorer / Tree View

## Problem
Create a recursive file/folder tree with expand/collapse and breadcrumb navigation.

## Key Concepts
- Recursion for nested rendering
- `useState` for expanded nodes
- Tree data structure

## Solution

```jsx
import { useState } from 'react';

function FileExplorer({ data }) {
  const [expanded, setExpanded] = useState(new Set());
  const [selected, setSelected] = useState(null);

  const toggleExpand = (id) => {
    setExpanded(prev => {
      const newSet = new Set(prev);
      if (newSet.has(id)) {
        newSet.delete(id);
      } else {
        newSet.add(id);
      }
      return newSet;
    });
  };

  const TreeNode = ({ node, level = 0 }) => {
    const isExpanded = expanded.has(node.id);
    const isFolder = node.type === 'folder';
    const isSelected = selected === node.id;

    return (
      <div>
        <div
          className={`tree-node ${isSelected ? 'selected' : ''}`}
          style={{ paddingLeft: `${level * 20}px` }}
          onClick={() => {
            if (isFolder) {
              toggleExpand(node.id);
            }
            setSelected(node.id);
          }}
        >
          {isFolder && (
            <span className="icon">{isExpanded ? '📂' : '📁'}</span>
          )}
          {!isFolder && <span className="icon">📄</span>}
          <span className="name">{node.name}</span>
        </div>
        {isFolder && isExpanded && node.children && (
          <div className="children">
            {node.children.map(child => (
              <TreeNode key={child.id} node={child} level={level + 1} />
            ))}
          </div>
        )}
      </div>
    );
  };

  return (
    <div className="file-explorer">
      <TreeNode node={data} />
    </div>
  );
}

export default FileExplorer;
```

## Sample Data

```jsx
const fileSystem = {
  id: 'root',
  name: 'Root',
  type: 'folder',
  children: [
    {
      id: 'src',
      name: 'src',
      type: 'folder',
      children: [
        { id: 'app', name: 'App.js', type: 'file' },
        { id: 'index', name: 'index.js', type: 'file' }
      ]
    },
    {
      id: 'public',
      name: 'public',
      type: 'folder',
      children: [
        { id: 'html', name: 'index.html', type: 'file' }
      ]
    },
    { id: 'readme', name: 'README.md', type: 'file' }
  ]
};

<FileExplorer data={fileSystem} />
```

## CSS

```css
.file-explorer {
  font-family: monospace;
  padding: 1rem;
}

.tree-node {
  display: flex;
  align-items: center;
  padding: 0.25rem;
  cursor: pointer;
  user-select: none;
}

.tree-node:hover {
  background: #f0f0f0;
}

.tree-node.selected {
  background: #e3f2fd;
}

.icon {
  margin-right: 0.5rem;
}

.name {
  flex: 1;
}
```

## Edge Cases Handled
- Recursive rendering
- Expand/collapse state
- Selection highlighting
- Indentation by level


## MANG-Level Expectations
- **Recursion**: Clean recursive component pattern
- **State Management**: Efficient expanded state (Set vs object)
- **Performance**: Avoid re-rendering entire tree on expand/collapse
- **Features**: Search, drag-drop, context menu, breadcrumbs
- **Accessibility**: Keyboard navigation, ARIA tree roles

## Enhanced Solution with Comments

```jsx
/**
 * TRICKY PART #1: Using Set for expanded state
 * - MANG expectation: Explain why Set instead of object/array
 * - Set provides O(1) lookup and easy add/remove
 * - Alternative: Could use object { [id]: true }
 */
const [expanded, setExpanded] = useState(new Set());

/**
 * TRICKY PART #2: Recursive component pattern
 * - Component renders itself for children
 * - MANG will ask: "What about performance with deep nesting?"
 * - Answer: React handles this well, but could memoize if needed
 */
const TreeNode = ({ node, level = 0 }) => {
  // ... render logic
  {isFolder && isExpanded && node.children && (
    <div className="children">
      {node.children.map(child => (
        <TreeNode key={child.id} node={child} level={level + 1} />
      ))}
    </div>
  )}
};

/**
 * TRICKY PART #3: Immutable Set updates
 * - Must create new Set, can't mutate existing
 * - MANG expectation: Understand immutability with Sets
 */
const toggleExpand = (id) => {
  setExpanded(prev => {
    const newSet = new Set(prev); // Create new Set
    if (newSet.has(id)) {
      newSet.delete(id);
    } else {
      newSet.add(id);
    }
    return newSet;
  });
};
```

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import FileExplorer from './FileExplorer';

const mockData = {
  id: 'root',
  name: 'Root',
  type: 'folder',
  children: [
    {
      id: 'folder1',
      name: 'Folder 1',
      type: 'folder',
      children: [
        { id: 'file1', name: 'File 1.txt', type: 'file' }
      ]
    },
    { id: 'file2', name: 'File 2.txt', type: 'file' }
  ]
};

describe('FileExplorer', () => {
  test('renders root node', () => {
    render(<FileExplorer data={mockData} />);
    expect(screen.getByText('Root')).toBeInTheDocument();
  });

  test('expands folder on click', () => {
    render(<FileExplorer data={mockData} />);
    
    expect(screen.queryByText('Folder 1')).not.toBeInTheDocument();
    
    fireEvent.click(screen.getByText('Root'));
    
    expect(screen.getByText('Folder 1')).toBeInTheDocument();
    expect(screen.getByText('File 2.txt')).toBeInTheDocument();
  });

  test('collapses folder on second click', () => {
    render(<FileExplorer data={mockData} />);
    
    fireEvent.click(screen.getByText('Root'));
    expect(screen.getByText('Folder 1')).toBeInTheDocument();
    
    fireEvent.click(screen.getByText('Root'));
    expect(screen.queryByText('Folder 1')).not.toBeInTheDocument();
  });

  test('shows folder icon for folders', () => {
    render(<FileExplorer data={mockData} />);
    fireEvent.click(screen.getByText('Root'));
    
    const folder1 = screen.getByText('Folder 1').parentElement;
    expect(folder1.textContent).toContain('📁');
  });

  test('shows file icon for files', () => {
    render(<FileExplorer data={mockData} />);
    fireEvent.click(screen.getByText('Root'));
    
    const file2 = screen.getByText('File 2.txt').parentElement;
    expect(file2.textContent).toContain('📄');
  });

  test('changes folder icon when expanded', () => {
    render(<FileExplorer data={mockData} />);
    
    const rootNode = screen.getByText('Root').parentElement;
    expect(rootNode.textContent).toContain('📁');
    
    fireEvent.click(screen.getByText('Root'));
    expect(rootNode.textContent).toContain('📂');
  });

  test('handles nested folders', () => {
    render(<FileExplorer data={mockData} />);
    
    fireEvent.click(screen.getByText('Root'));
    fireEvent.click(screen.getByText('Folder 1'));
    
    expect(screen.getByText('File 1.txt')).toBeInTheDocument();
  });

  test('selects node on click', () => {
    render(<FileExplorer data={mockData} />);
    
    fireEvent.click(screen.getByText('Root'));
    fireEvent.click(screen.getByText('File 2.txt'));
    
    const file2Node = screen.getByText('File 2.txt').parentElement;
    expect(file2Node).toHaveClass('selected');
  });

  test('only one node selected at a time', () => {
    render(<FileExplorer data={mockData} />);
    
    fireEvent.click(screen.getByText('Root'));
    const rootNode = screen.getByText('Root').parentElement;
    expect(rootNode).toHaveClass('selected');
    
    fireEvent.click(screen.getByText('File 2.txt'));
    expect(rootNode).not.toHaveClass('selected');
  });

  test('applies correct indentation', () => {
    render(<FileExplorer data={mockData} />);
    
    fireEvent.click(screen.getByText('Root'));
    fireEvent.click(screen.getByText('Folder 1'));
    
    const file1Node = screen.getByText('File 1.txt').parentElement;
    expect(file1Node).toHaveStyle({ paddingLeft: '40px' }); // level 2 * 20px
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add search/filter functionality?**
```jsx
const [searchTerm, setSearchTerm] = useState('');

const filterTree = (node, term) => {
  if (!term) return node;
  
  if (node.name.toLowerCase().includes(term.toLowerCase())) {
    return node;
  }
  
  if (node.children) {
    const filteredChildren = node.children
      .map(child => filterTree(child, term))
      .filter(Boolean);
    
    if (filteredChildren.length > 0) {
      return { ...node, children: filteredChildren };
    }
  }
  
  return null;
};

const filteredData = filterTree(data, searchTerm);

// Auto-expand matching nodes
useEffect(() => {
  if (searchTerm) {
    const matchingIds = findMatchingNodeIds(data, searchTerm);
    setExpanded(new Set(matchingIds));
  }
}, [searchTerm, data]);
```

**Q: How would you add drag-and-drop to move files?**
```jsx
const [draggedNode, setDraggedNode] = useState(null);

const handleDragStart = (node) => (e) => {
  setDraggedNode(node);
  e.dataTransfer.effectAllowed = 'move';
};

const handleDrop = (targetNode) => (e) => {
  e.preventDefault();
  if (targetNode.type === 'folder' && draggedNode) {
    // Move draggedNode into targetNode
    onMove?.(draggedNode.id, targetNode.id);
  }
  setDraggedNode(null);
};

<div
  draggable
  onDragStart={handleDragStart(node)}
  onDrop={handleDrop(node)}
  onDragOver={(e) => e.preventDefault()}
>
  {node.name}
</div>
```

**Q: How would you add context menu (right-click)?**
```jsx
const [contextMenu, setContextMenu] = useState(null);

const handleContextMenu = (node) => (e) => {
  e.preventDefault();
  setContextMenu({
    node,
    x: e.clientX,
    y: e.clientY
  });
};

{contextMenu && (
  <div
    style={{ position: 'fixed', top: contextMenu.y, left: contextMenu.x }}
    className="context-menu"
  >
    <button onClick={() => handleRename(contextMenu.node)}>Rename</button>
    <button onClick={() => handleDelete(contextMenu.node)}>Delete</button>
    <button onClick={() => handleCopy(contextMenu.node)}>Copy</button>
  </div>
)}
```

**Q: How would you add breadcrumb navigation?**
```jsx
const [path, setPath] = useState([]);

const findPath = (node, targetId, currentPath = []) => {
  if (node.id === targetId) {
    return [...currentPath, node];
  }
  
  if (node.children) {
    for (const child of node.children) {
      const result = findPath(child, targetId, [...currentPath, node]);
      if (result) return result;
    }
  }
  
  return null;
};

useEffect(() => {
  if (selected) {
    const nodePath = findPath(data, selected);
    setPath(nodePath || []);
  }
}, [selected, data]);

<div className="breadcrumb">
  {path.map((node, i) => (
    <span key={node.id}>
      <button onClick={() => setSelected(node.id)}>{node.name}</button>
      {i < path.length - 1 && ' / '}
    </span>
  ))}
</div>
```

**Q: How would you make this accessible?**
```jsx
<div
  role="tree"
  aria-label="File explorer"
>
  <div
    role="treeitem"
    aria-expanded={isExpanded}
    aria-selected={isSelected}
    aria-level={level + 1}
    tabIndex={0}
    onKeyDown={(e) => {
      if (e.key === 'Enter' || e.key === ' ') {
        toggleExpand(node.id);
      }
      if (e.key === 'ArrowRight' && !isExpanded) {
        toggleExpand(node.id);
      }
      if (e.key === 'ArrowLeft' && isExpanded) {
        toggleExpand(node.id);
      }
    }}
  >
    {node.name}
  </div>
</div>
```

**Q: How would you optimize for large trees (10,000+ nodes)?**
```jsx
// 1. Virtualization (react-window)
// 2. Lazy loading (load children on expand)
// 3. Memoization

const TreeNode = React.memo(({ node, level }) => {
  // ... component logic
}, (prevProps, nextProps) => {
  return prevProps.node.id === nextProps.node.id &&
         prevProps.level === nextProps.level;
});

// 4. Flatten tree for virtualization
const flattenTree = (node, level = 0, expanded) => {
  const result = [{ ...node, level }];
  
  if (node.children && expanded.has(node.id)) {
    node.children.forEach(child => {
      result.push(...flattenTree(child, level + 1, expanded));
    });
  }
  
  return result;
};
```

## Performance Considerations

1. **Set vs Object for Expanded State**
   - Set: O(1) lookup, cleaner API
   - Object: Also O(1), but more verbose

2. **Memoization**
   - Use React.memo for TreeNode if tree is large
   - Prevents re-rendering unchanged branches

3. **Virtualization**
   - For 1000+ nodes, use react-window
   - Only render visible nodes

## Edge Cases Handled
- Recursive rendering for nested structures
- Expand/collapse state management
- Selection highlighting
- Indentation by level
- Folder vs file icons
- Empty folders
- Deep nesting
- Immutable state updates with Set
