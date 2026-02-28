# 27. Multi-Select Tag Input

## Problem Description
Build a Tag Input component where a user can type text, press "Enter" or "Comma", and it organically creates a visual tag. Tags must be removable by clicking an embedded "X" or pressing the backspace key strictly when the textual input is empty. Assume you can include suggestion dropdowns eventually.

**Key Concepts Tested:** Controlled inputs, complex array state transformations, targeted keyboard event handling (`onKeyDown`), robust composition, semantic accessibility.

## React Component Implementation Outline

```tsx
import React, { useState, KeyboardEvent, useRef } from 'react';
import './TagInput.css';

interface TagInputProps {
  initialTags?: string[];
  placeholder?: string;
}

export const TagInput: React.FC<TagInputProps> = ({ 
  initialTags = [], 
  placeholder = "Add a tag..." 
}) => {
  const [tags, setTags] = useState<string[]>(initialTags);
  const [inputValue, setInputValue] = useState('');
  const wrapperRef = useRef<HTMLDivElement>(null);

  const handleKeyDown = (e: KeyboardEvent<HTMLInputElement>) => {
    // Triggers adding a new tag
    if ((e.key === 'Enter' || e.key === ',') && inputValue.trim()) {
      e.preventDefault();
      const newTag = inputValue.trim();
      
      // Prevent duplicates
      if (!tags.some(tag => tag.toLowerCase() === newTag.toLowerCase())) {
        setTags([...tags, newTag]);
      }
      setInputValue('');
    } 
    // Triggers removing the last tag if pressing backspace on an empty field
    else if (e.key === 'Backspace' && !inputValue && tags.length > 0) {
      setTags(tags.slice(0, -1));
    }
  };

  const removeTag = (indexToRemove: number) => {
    setTags(tags.filter((_, idx) => idx !== indexToRemove));
  };

  return (
    <div 
      className="tag-input-container" 
      onClick={() => wrapperRef.current?.querySelector('input')?.focus()}
    >
      <div className="tags-wrapper" ref={wrapperRef}>
        {tags.map((tag, index) => (
          <span key={`${tag}-${index}`} className="tag">
            {tag}
            <button 
              type="button" 
              onClick={(e) => {
                e.stopPropagation(); // Stop bringing wrapper focus
                removeTag(index);
              }}
              aria-label={`Remove tag ${tag}`}
            >
              &times;
            </button>
          </span>
        ))}
        <input
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder={tags.length === 0 ? placeholder : ""}
          className="tag-input"
          aria-label="New tag input"
        />
      </div>
    </div>
  );
};
```

## Vitest Unit Test Cases

```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { TagInput } from './TagInput';

describe('TagInput Component', () => {
  it('renders correctly', () => {
    render(<TagInput />);
    expect(screen.getByPlaceholderText('Add a tag...')).toBeInTheDocument();
  });

  it('adds a tag on pressing Enter', () => {
    render(<TagInput />);
    const input = screen.getByPlaceholderText('Add a tag...');
    
    fireEvent.change(input, { target: { value: 'React' } });
    fireEvent.keyDown(input, { key: 'Enter', code: 'Enter' });
    
    expect(screen.getByText('React')).toBeInTheDocument();
    expect(input).toHaveValue(''); // Resets
  });

  it('adds a tag on pressing Comma', () => {
    render(<TagInput />);
    const input = screen.getByPlaceholderText('Add a tag...');
    
    fireEvent.change(input, { target: { value: 'JavaScript' } });
    fireEvent.keyDown(input, { key: ',', code: 'Comma' });
    
    expect(screen.getByText('JavaScript')).toBeInTheDocument();
  });

  it('removes specific tag on clicking its delete button', () => {
    render(<TagInput />);
    const input = screen.getByPlaceholderText('Add a tag...');
    
    fireEvent.change(input, { target: { value: 'React' } });
    fireEvent.keyDown(input, { key: 'Enter', code: 'Enter' });
    
    const removeBtn = screen.getByLabelText('Remove tag React');
    fireEvent.click(removeBtn);
    
    expect(screen.queryByText('React')).not.toBeInTheDocument();
  });

  it('removes the last tag on Backspace when input is totally empty', () => {
    render(<TagInput />);
    const input = screen.getByPlaceholderText('Add a tag...');
    
    fireEvent.change(input, { target: { value: 'React' } });
    fireEvent.keyDown(input, { key: 'Enter' });
    
    fireEvent.change(input, { target: { value: 'Vue' } });
    fireEvent.keyDown(input, { key: 'Enter' });
    
    // Simulate backspace with empty input
    fireEvent.keyDown(input, { key: 'Backspace', code: 'Backspace' });
    
    expect(screen.queryByText('Vue')).not.toBeInTheDocument(); // Last tag deleted
    expect(screen.getByText('React')).toBeInTheDocument(); // First tag preserved
  });
});
```

## MANG-Level Cross Questions

1. **Focus Management & Accessibility:** Right now, if users click the native delete icon via a mouse click pointer, the keyboard focus resets to `document.body`. How ensure keyboard context makes sense, returning visual focus to the input box reliably after every deletion? 
2. **Duplication & Hashing:** Currently, the component checks `!tags.some(tag => tag.toLowerCase() === newTag.toLowerCase())`. Why shouldn't you just use `tags.includes`? What if it's a huge list of tags, is there computationally a better data structure? *(Expectation: Discussing `Set` vs `Array` differences).*
3. **Data Scaling & Server-Side Suggestions:** The prompt hinted at an autocomplete dropdown. If the pool of suggestions includes 50,000 DB models fetched via API, how do you handle searching dynamically on keypress? *(Expectation: Debouncing user input, stale-closure request caching, abort controllers for obsolete fetches).*
4. **Layout Shifts (CLS):** When the `input` flex wrapping expands the parent container to multiple dynamic lines horizontally, it causes immense Cumulative Layout Shift (CLS) on the application layout blocks underneath. How construct the CSS (e.g., restricted-box scrolling vs absolute positioned overflow) to mitigate the DOM jitter impact?
5. **Pure Controlled Architecture:** The state (`tags`) is currently tightly-coupled and internal. If a parent higher up the DOM tree dictates server-side form validation (e.g., throwing "You MUST have exactly 3 tags!"), how would you redesign `TagInput` to act purely controlled via `value={propTags}` and `onChange={setPropTags}`?
