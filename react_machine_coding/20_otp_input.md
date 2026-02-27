# OTP Input Component

## Problem
Create a 4-6 digit OTP input with auto-focusing boxes, paste support, and backspace navigation.

## Key Concepts
- `useRef` array for input elements
- `onKeyDown` for navigation
- `onPaste` for paste handling
- Focus management

## Solution

```jsx
import { useState, useRef, useEffect } from 'react';

function OTPInput({ length = 6, onComplete }) {
  const [otp, setOtp] = useState(Array(length).fill(''));
  const inputRefs = useRef([]);

  useEffect(() => {
    // Focus first input on mount
    inputRefs.current[0]?.focus();
  }, []);

  useEffect(() => {
    // Check if OTP is complete
    if (otp.every(digit => digit !== '')) {
      onComplete?.(otp.join(''));
    }
  }, [otp, onComplete]);

  const handleChange = (index, value) => {
    // Only allow digits
    if (!/^\d*$/.test(value)) return;

    const newOtp = [...otp];
    newOtp[index] = value.slice(-1); // Take only last character
    setOtp(newOtp);

    // Auto-focus next input
    if (value && index < length - 1) {
      inputRefs.current[index + 1]?.focus();
    }
  };

  const handleKeyDown = (index, e) => {
    if (e.key === 'Backspace') {
      if (!otp[index] && index > 0) {
        // Move to previous input if current is empty
        inputRefs.current[index - 1]?.focus();
      } else {
        // Clear current input
        const newOtp = [...otp];
        newOtp[index] = '';
        setOtp(newOtp);
      }
    } else if (e.key === 'ArrowLeft' && index > 0) {
      inputRefs.current[index - 1]?.focus();
    } else if (e.key === 'ArrowRight' && index < length - 1) {
      inputRefs.current[index + 1]?.focus();
    }
  };

  const handlePaste = (e) => {
    e.preventDefault();
    const pastedData = e.clipboardData.getData('text').slice(0, length);
    
    if (!/^\d+$/.test(pastedData)) return;

    const newOtp = [...otp];
    pastedData.split('').forEach((char, i) => {
      if (i < length) newOtp[i] = char;
    });
    setOtp(newOtp);

    // Focus last filled input or next empty
    const nextIndex = Math.min(pastedData.length, length - 1);
    inputRefs.current[nextIndex]?.focus();
  };

  return (
    <div className="otp-input">
      {otp.map((digit, index) => (
        <input
          key={index}
          ref={(el) => (inputRefs.current[index] = el)}
          type="text"
          inputMode="numeric"
          maxLength={1}
          value={digit}
          onChange={(e) => handleChange(index, e.target.value)}
          onKeyDown={(e) => handleKeyDown(index, e)}
          onPaste={handlePaste}
          className="otp-box"
        />
      ))}
    </div>
  );
}

export default OTPInput;
```

## CSS

```css
.otp-input {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
}

.otp-box {
  width: 3rem;
  height: 3rem;
  text-align: center;
  font-size: 1.5rem;
  border: 2px solid #ccc;
  border-radius: 4px;
  transition: border-color 0.2s;
}

.otp-box:focus {
  outline: none;
  border-color: #007bff;
}
```

## Usage

```jsx
<OTPInput 
  length={6}
  onComplete={(otp) => console.log('OTP entered:', otp)}
/>
```

## Edge Cases Handled
- Auto-focus on mount
- Paste support for full OTP
- Backspace navigation
- Arrow key navigation
- Only numeric input
- Auto-focus next on input
- onComplete callback when filled


## MANG-Level Expectations
- **Accessibility**: ARIA labels, screen reader support, focus management
- **UX**: Auto-focus, paste support, backspace navigation, arrow keys
- **Security**: Mask input option, auto-clear on timeout
- **Mobile**: Numeric keyboard, proper input types
- **Validation**: Real-time validation, error states

## Test Cases

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import OTPInput from './OTPInput';

describe('OTPInput', () => {
  const mockOnComplete = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders correct number of input boxes', () => {
    render(<OTPInput length={6} />);
    const inputs = screen.getAllByRole('textbox');
    expect(inputs).toHaveLength(6);
  });

  test('focuses first input on mount', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');
    expect(inputs[0]).toHaveFocus();
  });

  test('moves to next input on digit entry', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');

    fireEvent.change(inputs[0], { target: { value: '1' } });
    expect(inputs[1]).toHaveFocus();

    fireEvent.change(inputs[1], { target: { value: '2' } });
    expect(inputs[2]).toHaveFocus();
  });

  test('only accepts numeric input', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');

    fireEvent.change(inputs[0], { target: { value: 'a' } });
    expect(inputs[0].value).toBe('');

    fireEvent.change(inputs[0], { target: { value: '5' } });
    expect(inputs[0].value).toBe('5');
  });

  test('handles backspace navigation', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');

    fireEvent.change(inputs[0], { target: { value: '1' } });
    fireEvent.change(inputs[1], { target: { value: '2' } });

    fireEvent.keyDown(inputs[1], { key: 'Backspace' });
    expect(inputs[1].value).toBe('');

    fireEvent.keyDown(inputs[1], { key: 'Backspace' });
    expect(inputs[0]).toHaveFocus();
  });

  test('handles paste of full OTP', () => {
    render(<OTPInput length={6} />);
    const inputs = screen.getAllByRole('textbox');

    const pasteEvent = {
      clipboardData: {
        getData: () => '123456'
      }
    };

    fireEvent.paste(inputs[0], pasteEvent);

    expect(inputs[0].value).toBe('1');
    expect(inputs[1].value).toBe('2');
    expect(inputs[2].value).toBe('3');
    expect(inputs[3].value).toBe('4');
    expect(inputs[4].value).toBe('5');
    expect(inputs[5].value).toBe('6');
  });

  test('handles paste of partial OTP', () => {
    render(<OTPInput length={6} />);
    const inputs = screen.getAllByRole('textbox');

    const pasteEvent = {
      clipboardData: {
        getData: () => '123'
      }
    };

    fireEvent.paste(inputs[0], pasteEvent);

    expect(inputs[0].value).toBe('1');
    expect(inputs[1].value).toBe('2');
    expect(inputs[2].value).toBe('3');
    expect(inputs[3].value).toBe('');
  });

  test('ignores non-numeric paste', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');

    const pasteEvent = {
      clipboardData: {
        getData: () => 'abcd'
      }
    };

    fireEvent.paste(inputs[0], pasteEvent);

    inputs.forEach(input => {
      expect(input.value).toBe('');
    });
  });

  test('calls onComplete when all digits entered', async () => {
    render(<OTPInput length={4} onComplete={mockOnComplete} />);
    const inputs = screen.getAllByRole('textbox');

    fireEvent.change(inputs[0], { target: { value: '1' } });
    fireEvent.change(inputs[1], { target: { value: '2' } });
    fireEvent.change(inputs[2], { target: { value: '3' } });
    fireEvent.change(inputs[3], { target: { value: '4' } });

    await waitFor(() => {
      expect(mockOnComplete).toHaveBeenCalledWith('1234');
    });
  });

  test('handles arrow key navigation', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');

    inputs[1].focus();

    fireEvent.keyDown(inputs[1], { key: 'ArrowLeft' });
    expect(inputs[0]).toHaveFocus();

    fireEvent.keyDown(inputs[0], { key: 'ArrowRight' });
    expect(inputs[1]).toHaveFocus();
  });

  test('takes only last character if multiple entered', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');

    fireEvent.change(inputs[0], { target: { value: '123' } });
    expect(inputs[0].value).toBe('3');
  });

  test('does not move focus on invalid input', () => {
    render(<OTPInput length={4} />);
    const inputs = screen.getAllByRole('textbox');

    inputs[0].focus();
    fireEvent.change(inputs[0], { target: { value: 'x' } });
    
    expect(inputs[0]).toHaveFocus();
    expect(inputs[0].value).toBe('');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add a masked/secure input mode?**
```jsx
// Add type="password" or custom masking
<input
  type={secure ? 'password' : 'text'}
  // Or use custom rendering with bullets
  value={secure ? '•'.repeat(digit.length) : digit}
/>
```

**Q: How would you handle auto-submit after completion?**
```jsx
useEffect(() => {
  if (otp.every(digit => digit !== '')) {
    const otpValue = otp.join('');
    onComplete?.(otpValue);
    
    // Auto-submit after delay
    if (autoSubmit) {
      setTimeout(() => {
        onSubmit?.(otpValue);
      }, 500);
    }
  }
}, [otp, onComplete, autoSubmit, onSubmit]);
```

**Q: How would you add a resend OTP feature?**
```jsx
const [canResend, setCanResend] = useState(false);
const [countdown, setCountdown] = useState(30);

useEffect(() => {
  if (countdown > 0) {
    const timer = setTimeout(() => setCountdown(c => c - 1), 1000);
    return () => clearTimeout(timer);
  } else {
    setCanResend(true);
  }
}, [countdown]);

const handleResend = () => {
  onResend?.();
  setCountdown(30);
  setCanResend(false);
  setOtp(Array(length).fill(''));
};
```

**Q: How would you make this accessible?**
```jsx
// Add ARIA attributes
<div role="group" aria-label="One-time password input">
  {otp.map((digit, index) => (
    <input
      aria-label={`Digit ${index + 1} of ${length}`}
      aria-required="true"
      aria-invalid={error ? 'true' : 'false'}
      // ... rest of props
    />
  ))}
</div>

// Announce completion to screen readers
{isComplete && (
  <div role="status" aria-live="polite" className="sr-only">
    OTP entered successfully
  </div>
)}
```

**Q: How would you handle mobile keyboards?**
```jsx
// Use inputMode for numeric keyboard
<input
  inputMode="numeric"  // Shows numeric keyboard on mobile
  pattern="[0-9]*"     // iOS numeric keyboard
  type="text"          // Not "number" to avoid spinners
/>
```

## Tricky Parts Explained

**1. Focus Management**
```jsx
// TRICKY: Need array of refs, not single ref
const inputRefs = useRef([]);

// Store refs correctly
ref={(el) => (inputRefs.current[index] = el)}

// Focus programmatically
inputRefs.current[nextIndex]?.focus();
```

**2. Paste Handling**
```jsx
// TRICKY: Must preventDefault to avoid default paste behavior
const handlePaste = (e) => {
  e.preventDefault(); // CRITICAL
  const pastedData = e.clipboardData.getData('text');
  // Process and distribute to inputs
};
```

**3. Backspace Logic**
```jsx
// TRICKY: Two behaviors
// 1. If current input has value, clear it
// 2. If current input is empty, move to previous
if (!otp[index] && index > 0) {
  inputRefs.current[index - 1]?.focus();
} else {
  // Clear current
}
```

## Edge Cases Handled
- Auto-focus on mount
- Paste support for full/partial OTP
- Backspace navigation (clear current or move back)
- Arrow key navigation
- Only numeric input validation
- Auto-focus next on input
- onComplete callback when filled
- Takes only last character if multiple entered
- Prevents non-numeric paste
- Mobile-friendly input types
