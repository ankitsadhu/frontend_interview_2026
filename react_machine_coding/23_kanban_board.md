# 23. Kanban Board

## Problem Description
Create a Kanban Board with multiple columns (e.g., "Todo", "In Progress", "Done"). 
Users should be able to drag and drop cards between columns, add new cards, edit existing cards, and delete cards.

**Key Concepts Tested:** Complex multidimensional state, HTML5 Drag & Drop API or 3rd-party D&D libraries, multi-list state management, component composition.

## React Component Implementation Outline

```tsx
import React, { useState } from 'react';
import './KanbanBoard.css';

type TaskStatus = 'todo' | 'in-progress' | 'done';

interface Task {
  id: string;
  title: string;
  status: TaskStatus;
}

export const KanbanBoard: React.FC = () => {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [draggedTaskId, setDraggedTaskId] = useState<string | null>(null);

  const handleDragStart = (e: React.DragEvent, id: string) => {
    setDraggedTaskId(id);
    e.dataTransfer.effectAllowed = 'move';
  };

  const handleDragOver = (e: React.DragEvent) => {
    // Necessary to allow dropping via HTML5 DnD 
    e.preventDefault(); 
  };

  const handleDrop = (e: React.DragEvent, newStatus: TaskStatus) => {
    e.preventDefault();
    if (!draggedTaskId) return;

    setTasks((prev) => 
      prev.map(t => t.id === draggedTaskId ? { ...t, status: newStatus } : t)
    );
    setDraggedTaskId(null);
  };

  const addTask = (status: TaskStatus) => {
    const title = prompt('Enter task title:');
    if (!title) return;
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      status
    };
    setTasks([...tasks, newTask]);
  };

  const columns: { id: TaskStatus; label: string }[] = [
    { id: 'todo', label: 'TODO' },
    { id: 'in-progress', label: 'IN PROGRESS' },
    { id: 'done', label: 'DONE' },
  ];

  return (
    <div className="board-layout">
      {columns.map(col => (
        <div 
          key={col.id}
          className="kanban-column"
          onDragOver={handleDragOver}
          onDrop={(e) => handleDrop(e, col.id)}
        >
          <h3>{col.label}</h3>
          <button onClick={() => addTask(col.id)}>+ Add Task</button>
          <div className="task-list">
            {tasks.filter(t => t.status === col.id).map(task => (
              <div
                key={task.id}
                draggable
                onDragStart={(e) => handleDragStart(e, task.id)}
                className="kanban-card"
              >
                {task.title}
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
};
```

## Vitest Unit Test Cases

```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { KanbanBoard } from './KanbanBoard';

describe('KanbanBoard Component', () => {
  it('renders columns correctly', () => {
    render(<KanbanBoard />);
    expect(screen.getByText('TODO')).toBeInTheDocument();
    expect(screen.getByText('IN PROGRESS')).toBeInTheDocument();
    expect(screen.getByText('DONE')).toBeInTheDocument();
  });

  it('can add a new task', () => {
    vi.spyOn(window, 'prompt').mockReturnValue('First API Integration');
    render(<KanbanBoard />);
    
    // Add task to TODO column
    const addTaskBtns = screen.getAllByText('+ Add Task');
    fireEvent.click(addTaskBtns[0]);
    
    expect(screen.getByText('First API Integration')).toBeInTheDocument();
    vi.restoreAllMocks();
  });

  // Note: Full HTML5 Drag & Drop is notoriously hard to test in JSDOM environments.
  // Interviews often expect you to discuss e2e testing (like Playwright/Cypress) 
  // or testing the standalone utility functions mapping states independently!
});
```

## MANG-Level Cross Questions

1. **Drag and Drop APIs:** You used the native HTML5 Drag and Drop API. What are its major limitations regarding mobile/touch devices? How would you build a solution that gracefully works on both desktop and mobile? *(Expectation: Pointer events mapping or robust libraries like `@dnd-kit/core`)*.
2. **State Modeling Trade-offs:** In your state, tasks are kept in a flat array, and columns derive from `Array.filter()`. What are the space/time complexity trade-offs of this vs. storing a map/dictionary of `status -> Task[]`?
3. **Optimistic Updates & Retry Mechanisms:** When a user drags a card, how would you optimistically update the UI to show the card in the new column instantly while an API call processes in the background? How do you smoothly handle reverting the UI if the API call eventually fails?
4. **Intra-Column Reordering:** Your current implementation doesn't support custom reordering *within* the same column. How would you modify your data schema and operations to track and persist arbitrary index ordering inside a column? *(Expectation: Linked lists, floating point/fractional index systems).*
5. **List Virtualization:** If a project gets massive and the "Done" column scales to 15,000 items, rendering them natively will crash the browser. How would you virtualize the list while keeping drag-and-drop functional across columns?
