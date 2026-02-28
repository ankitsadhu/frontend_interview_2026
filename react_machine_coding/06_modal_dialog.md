# Modal / Dialog Component

## Problem
Create a modal with open/close functionality, overlay click to dismiss, and focus trap.

## Key Concepts
- React Portal for rendering outside DOM hierarchy
- `useEffect` for keydown listeners (Escape key)
- Focus trap for accessibility
- Body scroll lock when modal is open

## MANG-Level Expectations
- **Accessibility**: Focus trap, ARIA attributes, keyboard navigation
- **UX**: Smooth animations, backdrop click to close, prevent body scroll
- **Performance**: Portal rendering, cleanup on unmount
- **Extensibility**: Multiple modals, stacking, custom sizes

## Solution

```jsx
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

/**
 * TRICKY PART #1: React Portal
 * - Renders modal outside parent DOM hierarchy
 * - MANG expectation: Explain why Portal is needed
 * - Avoids z-index and overflow issues from parent containers
 */
function Modal({ isOpen, onClose, title, children, closeOnOverlay = true }) {
  const modalRef = useRef(null);
  const previousActiveElement = useRef(null);

  /**
   * TRICKY PART #2: Focus trap
   * - Keep focus within modal while open
   * - MANG will ask: "How do you handle accessibility?"
   * - Must trap Tab key and cycle through focusable elements
   */
  useEffect(() => {
    if (!isOpen) return;

    // Store currently focused element
    previousActiveElement.current = document.activeElement;

    // Focus first focusable element in modal
    const focusableElements = modalRef.current?.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    
    if (focusableElements?.length > 0) {
      focusableElements[0].focus();
    }

    // Restore focus on unmount
    return () => {
      previousActiveElement.current?.focus();
    };
  }, [isOpen]);

  /**
   * TRICKY PART #3: Keyboard event handling
   * - Escape to close
   * - Tab key for focus trap
   * - CRITICAL: Cleanup listener on unmount
   */
  useEffect(() => {
    if (!isOpen) return;

    const handleKeyDown = (e) => {
      if (e.key === 'Escape') {
        onClose();
      }

      // Focus trap logic
      if (e.key === 'Tab') {
        const focusableElements = modalRef.current?.querySelectorAll(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        );
        
        if (!focusableElements || focusableElements.length === 0) return;

        const firstElement = focusableElements[0];
        const lastElement = focusableElements[focusableElements.length - 1];

        if (e.shiftKey) {
          // Shift + Tab
          if (document.activeElement === firstElement) {
            e.preventDefault();
            lastElement.focus();
          }
        } else {
          // Tab
          if (document.activeElement === lastElement) {
            e.preventDefault();
            firstElement.focus();
          }
        }
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);

  /**
   * TRICKY PART #4: Prevent body scroll
   * - Lock body scroll when modal is open
   * - MANG expectation: Handle scroll position restoration
   */
  useEffect(() => {
    if (!isOpen) return;

    const originalOverflow = document.body.style.overflow;
    const originalPaddingRight = document.body.style.paddingRight;
    
    // Get scrollbar width to prevent layout shift
    const scrollbarWidth = window.innerWidth - document.documentElement.clientWidth;
    
    document.body.style.overflow = 'hidden';
    document.body.style.paddingRight = `${scrollbarWidth}px`;

    return () => {
      document.body.style.overflow = originalOverflow;
      document.body.style.paddingRight = originalPaddingRight;
    };
  }, [isOpen]);

  const handleOverlayClick = (e) => {
    // Only close if clicking the overlay itself, not its children
    if (closeOnOverlay && e.target === e.currentTarget) {
      onClose();
    }
  };

  if (!isOpen) return null;

  return createPortal(
    <div 
      className="modal-overlay" 
      onClick={handleOverlayClick}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <div className="modal-content" ref={modalRef}>
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button
            className="modal-close"
            onClick={onClose}
            aria-label="Close modal"
          >
            ×
          </button>
        </div>
        <div className="modal-body">
          {children}
        </div>
      </div>
    </div>,
    document.body
  );
}

export default Modal;
```

## CSS

```css
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeIn 0.2s ease-out;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.modal-content {
  background: white;
  border-radius: 8px;
  max-width: 500px;
  width: 90%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  animation: slideUp 0.3s ease-out;
}

@keyframes slideUp {
  from {
    transform: translateY(20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border-bottom: 1px solid #eee;
}

.modal-header h2 {
  margin: 0;
  font-size: 1.25rem;
}

.modal-close {
  background: none;
  border: none;
  font-size: 2rem;
  cursor: pointer;
  color: #666;
  line-height: 1;
  padding: 0;
  width: 2rem;
  height: 2rem;
}

.modal-close:hover {
  color: #000;
}

.modal-body {
  padding: 1rem;
}
```

## Usage

```jsx
function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      
      <Modal
        isOpen={isOpen}
        onClose={() => setIsOpen(false)}
        title="Confirm Action"
      >
        <p>Are you sure you want to proceed?</p>
        <div style={{ display: 'flex', gap: '1rem', marginTop: '1rem' }}>
          <button onClick={() => setIsOpen(false)}>Cancel</button>
          <button onClick={() => {
            console.log('Confirmed');
            setIsOpen(false);
          }}>
            Confirm
          </button>
        </div>
      </Modal>
    </>
  );
}
```

## Test Cases

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Modal from './Modal';

describe('Modal', () => {
  test('does not render when closed', () => {
    render(
      <Modal isOpen={false} onClose={() => {}} title="Test">
        Content
      </Modal>
    );
    
    expect(screen.queryByText('Test')).not.toBeInTheDocument();
  });

  test('renders when open', () => {
    render(
      <Modal isOpen={true} onClose={() => {}} title="Test Modal">
        Modal Content
      </Modal>
    );
    
    expect(screen.getByText('Test Modal')).toBeInTheDocument();
    expect(screen.getByText('Modal Content')).toBeInTheDocument();
  });

  test('closes on Escape key', () => {
    const handleClose = jest.fn();
    render(
      <Modal isOpen={true} onClose={handleClose} title="Test">
        Content
      </Modal>
    );
    
    fireEvent.keyDown(document, { key: 'Escape' });
    
    expect(handleClose).toHaveBeenCalled();
  });

  test('closes on close button click', () => {
    const handleClose = jest.fn();
    render(
      <Modal isOpen={true} onClose={handleClose} title="Test">
        Content
      </Modal>
    );
    
    fireEvent.click(screen.getByLabelText('Close modal'));
    
    expect(handleClose).toHaveBeenCalled();
  });

  test('closes on overlay click', () => {
    const handleClose = jest.fn();
    render(
      <Modal isOpen={true} onClose={handleClose} title="Test">
        Content
      </Modal>
    );
    
    fireEvent.click(screen.getByRole('dialog'));
    
    expect(handleClose).toHaveBeenCalled();
  });

  test('does not close on content click', () => {
    const handleClose = jest.fn();
    render(
      <Modal isOpen={true} onClose={handleClose} title="Test">
        Content
      </Modal>
    );
    
    fireEvent.click(screen.getByText('Content'));
    
    expect(handleClose).not.toHaveBeenCalled();
  });

  test('does not close on overlay click when closeOnOverlay is false', () => {
    const handleClose = jest.fn();
    render(
      <Modal isOpen={true} onClose={handleClose} title="Test" closeOnOverlay={false}>
        Content
      </Modal>
    );
    
    fireEvent.click(screen.getByRole('dialog'));
    
    expect(handleClose).not.toHaveBeenCalled();
  });

  test('traps focus within modal', async () => {
    render(
      <Modal isOpen={true} onClose={() => {}} title="Test">
        <button>First</button>
        <button>Second</button>
        <button>Last</button>
      </Modal>
    );
    
    const buttons = screen.getAllByRole('button');
    const lastButton = buttons[buttons.length - 1];
    
    lastButton.focus();
    expect(lastButton).toHaveFocus();
    
    // Tab from last element should cycle to first
    userEvent.tab();
    
    await waitFor(() => {
      expect(buttons[0]).toHaveFocus();
    });
  });

  test('restores focus on close', () => {
    const trigger = document.createElement('button');
    trigger.textContent = 'Open';
    document.body.appendChild(trigger);
    trigger.focus();
    
    const { rerender } = render(
      <Modal isOpen={true} onClose={() => {}} title="Test">
        Content
      </Modal>
    );
    
    rerender(
      <Modal isOpen={false} onClose={() => {}} title="Test">
        Content
      </Modal>
    );
    
    expect(trigger).toHaveFocus();
    
    document.body.removeChild(trigger);
  });

  test('has correct ARIA attributes', () => {
    render(
      <Modal isOpen={true} onClose={() => {}} title="Test Modal">
        Content
      </Modal>
    );
    
    const dialog = screen.getByRole('dialog');
    expect(dialog).toHaveAttribute('aria-modal', 'true');
    expect(dialog).toHaveAttribute('aria-labelledby', 'modal-title');
  });

  test('prevents body scroll when open', () => {
    const { rerender } = render(
      <Modal isOpen={true} onClose={() => {}} title="Test">
        Content
      </Modal>
    );
    
    expect(document.body.style.overflow).toBe('hidden');
    
    rerender(
      <Modal isOpen={false} onClose={() => {}} title="Test">
        Content
      </Modal>
    );
    
    expect(document.body.style.overflow).toBe('');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you handle stacked modals?**
```jsx
// Use context to manage modal stack
const ModalContext = createContext();

export function ModalProvider({ children }) {
  const [modals, setModals] = useState([]);

  const openModal = (modal) => {
    setModals(prev => [...prev, { ...modal, id: Date.now() }]);
  };

  const closeModal = (id) => {
    setModals(prev => prev.filter(m => m.id !== id));
  };

  return (
    <ModalContext.Provider value={{ openModal, closeModal }}>
      {children}
      {modals.map((modal, index) => (
        <Modal
          key={modal.id}
          isOpen={true}
          onClose={() => closeModal(modal.id)}
          style={{ zIndex: 1000 + index }}
          {...modal}
        />
      ))}
    </ModalContext.Provider>
  );
}
```

**Q: How would you add animations for enter/exit?**
```jsx
// Use react-transition-group or framer-motion
import { AnimatePresence, motion } from 'framer-motion';

{isOpen && (
  <AnimatePresence>
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
      className="modal-overlay"
    >
      <motion.div
        initial={{ scale: 0.9, y: 20 }}
        animate={{ scale: 1, y: 0 }}
        exit={{ scale: 0.9, y: 20 }}
        className="modal-content"
      >
        {children}
      </motion.div>
    </motion.div>
  </AnimatePresence>
)}
```

**Q: How would you handle form submission in modal?**
```jsx
function FormModal({ isOpen, onClose, onSubmit }) {
  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    
    try {
      await onSubmit(Object.fromEntries(formData));
      onClose();
    } catch (error) {
      // Handle error
    }
  };

  return (
    <Modal isOpen={isOpen} onClose={onClose} title="Submit Form">
      <form onSubmit={handleSubmit}>
        <input name="name" required />
        <button type="submit">Submit</button>
      </form>
    </Modal>
  );
}
```

**Q: How would you make modal size responsive?**
```jsx
function Modal({ size = 'medium', ...props }) {
  const sizeClasses = {
    small: 'max-w-sm',
    medium: 'max-w-md',
    large: 'max-w-2xl',
    full: 'max-w-full'
  };

  return (
    <div className={`modal-content ${sizeClasses[size]}`}>
      {/* content */}
    </div>
  );
}
```

## Edge Cases Handled
- Portal rendering outside parent DOM
- Focus trap with Tab/Shift+Tab
- Escape key to close
- Overlay click to dismiss (configurable)
- Body scroll lock with scrollbar compensation
- Focus restoration on close
- ARIA attributes for accessibility
- Cleanup on unmount
- Multiple focusable elements handling
- Empty modal (no focusable elements)
