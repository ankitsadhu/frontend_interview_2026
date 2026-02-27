# Multi-Step Form / Wizard

## Problem
Create a multi-step form with step navigation, validation per step, and summary review.

## Key Concepts
- `useState` for step and form state
- Conditional rendering per step
- Validation logic

## Solution

```jsx
import { useState } from 'react';

function MultiStepForm() {
  const [currentStep, setCurrentStep] = useState(1);
  const [formData, setFormData] = useState({
    // Step 1
    firstName: '',
    lastName: '',
    email: '',
    // Step 2
    address: '',
    city: '',
    zipCode: '',
    // Step 3
    cardNumber: '',
    expiryDate: '',
    cvv: ''
  });
  const [errors, setErrors] = useState({});

  const totalSteps = 3;

  const updateField = (field, value) => {
    setFormData(prev => ({ ...prev, [field]: value }));
    // Clear error for this field
    if (errors[field]) {
      setErrors(prev => ({ ...prev, [field]: '' }));
    }
  };

  const validateStep = (step) => {
    const newErrors = {};

    if (step === 1) {
      if (!formData.firstName.trim()) newErrors.firstName = 'First name required';
      if (!formData.lastName.trim()) newErrors.lastName = 'Last name required';
      if (!formData.email.includes('@')) newErrors.email = 'Valid email required';
    }

    if (step === 2) {
      if (!formData.address.trim()) newErrors.address = 'Address required';
      if (!formData.city.trim()) newErrors.city = 'City required';
      if (!/^\d{5}$/.test(formData.zipCode)) newErrors.zipCode = 'Valid 5-digit zip required';
    }

    if (step === 3) {
      if (!/^\d{16}$/.test(formData.cardNumber.replace(/\s/g, ''))) {
        newErrors.cardNumber = 'Valid 16-digit card number required';
      }
      if (!/^\d{2}\/\d{2}$/.test(formData.expiryDate)) {
        newErrors.expiryDate = 'Format: MM/YY';
      }
      if (!/^\d{3}$/.test(formData.cvv)) newErrors.cvv = '3-digit CVV required';
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleNext = () => {
    if (validateStep(currentStep)) {
      setCurrentStep(prev => prev + 1);
    }
  };

  const handleBack = () => {
    setCurrentStep(prev => prev - 1);
  };

  const handleSubmit = () => {
    if (validateStep(currentStep)) {
      console.log('Form submitted:', formData);
      alert('Form submitted successfully!');
    }
  };

  return (
    <div className="multi-step-form">
      {/* Progress indicator */}
      <div className="progress">
        {[...Array(totalSteps)].map((_, i) => (
          <div
            key={i}
            className={`step ${i + 1 <= currentStep ? 'active' : ''}`}
          >
            {i + 1}
          </div>
        ))}
      </div>

      {/* Step 1: Personal Info */}
      {currentStep === 1 && (
        <div className="form-step">
          <h2>Personal Information</h2>
          <input
            type="text"
            placeholder="First Name"
            value={formData.firstName}
            onChange={(e) => updateField('firstName', e.target.value)}
          />
          {errors.firstName && <span className="error">{errors.firstName}</span>}

          <input
            type="text"
            placeholder="Last Name"
            value={formData.lastName}
            onChange={(e) => updateField('lastName', e.target.value)}
          />
          {errors.lastName && <span className="error">{errors.lastName}</span>}

          <input
            type="email"
            placeholder="Email"
            value={formData.email}
            onChange={(e) => updateField('email', e.target.value)}
          />
          {errors.email && <span className="error">{errors.email}</span>}
        </div>
      )}

      {/* Step 2: Address */}
      {currentStep === 2 && (
        <div className="form-step">
          <h2>Address</h2>
          <input
            type="text"
            placeholder="Street Address"
            value={formData.address}
            onChange={(e) => updateField('address', e.target.value)}
          />
          {errors.address && <span className="error">{errors.address}</span>}

          <input
            type="text"
            placeholder="City"
            value={formData.city}
            onChange={(e) => updateField('city', e.target.value)}
          />
          {errors.city && <span className="error">{errors.city}</span>}

          <input
            type="text"
            placeholder="Zip Code"
            value={formData.zipCode}
            onChange={(e) => updateField('zipCode', e.target.value)}
          />
          {errors.zipCode && <span className="error">{errors.zipCode}</span>}
        </div>
      )}

      {/* Step 3: Payment */}
      {currentStep === 3 && (
        <div className="form-step">
          <h2>Payment Information</h2>
          <input
            type="text"
            placeholder="Card Number"
            value={formData.cardNumber}
            onChange={(e) => updateField('cardNumber', e.target.value)}
          />
          {errors.cardNumber && <span className="error">{errors.cardNumber}</span>}

          <input
            type="text"
            placeholder="Expiry Date (MM/YY)"
            value={formData.expiryDate}
            onChange={(e) => updateField('expiryDate', e.target.value)}
          />
          {errors.expiryDate && <span className="error">{errors.expiryDate}</span>}

          <input
            type="text"
            placeholder="CVV"
            value={formData.cvv}
            onChange={(e) => updateField('cvv', e.target.value)}
          />
          {errors.cvv && <span className="error">{errors.cvv}</span>}
        </div>
      )}

      {/* Navigation buttons */}
      <div className="navigation">
        {currentStep > 1 && (
          <button onClick={handleBack}>Back</button>
        )}
        {currentStep < totalSteps ? (
          <button onClick={handleNext}>Next</button>
        ) : (
          <button onClick={handleSubmit}>Submit</button>
        )}
      </div>
    </div>
  );
}

export default MultiStepForm;
```

## CSS

```css
.multi-step-form {
  max-width: 500px;
  margin: 2rem auto;
  padding: 2rem;
}

.progress {
  display: flex;
  justify-content: space-between;
  margin-bottom: 2rem;
}

.step {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: #ddd;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
}

.step.active {
  background: #007bff;
  color: white;
}

.form-step input {
  width: 100%;
  padding: 0.5rem;
  margin-bottom: 0.5rem;
  font-size: 1rem;
}

.error {
  color: #d32f2f;
  font-size: 0.875rem;
  display: block;
  margin-bottom: 1rem;
}

.navigation {
  display: flex;
  justify-content: space-between;
  margin-top: 2rem;
}

.navigation button {
  padding: 0.5rem 2rem;
  cursor: pointer;
}
```

## Edge Cases Handled
- Per-step validation
- Error messages
- Progress indicator
- Back navigation
- Form state persistence across steps


## MANG-Level Expectations
- **State Management**: Single source of truth, validation per step
- **UX**: Progress indicator, back navigation, summary review
- **Validation**: Real-time vs on-submit, error messages
- **Persistence**: Save draft, resume later
- **Accessibility**: Focus management, ARIA attributes

## Test Cases

```jsx
import { render, screen, fireEvent } from '@testing-library/react';
import MultiStepForm from './MultiStepForm';

describe('MultiStepForm', () => {
  test('renders first step initially', () => {
    render(<MultiStepForm />);
    expect(screen.getByText('Personal Information')).toBeInTheDocument();
    expect(screen.getByPlaceholderText('First Name')).toBeInTheDocument();
  });

  test('validates required fields', () => {
    render(<MultiStepForm />);
    
    fireEvent.click(screen.getByText('Next'));
    
    expect(screen.getByText('First name required')).toBeInTheDocument();
    expect(screen.getByText('Last name required')).toBeInTheDocument();
  });

  test('validates email format', () => {
    render(<MultiStepForm />);
    
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'invalid-email' }
    });
    fireEvent.click(screen.getByText('Next'));
    
    expect(screen.getByText('Valid email required')).toBeInTheDocument();
  });

  test('moves to next step on valid input', () => {
    render(<MultiStepForm />);
    
    fireEvent.change(screen.getByPlaceholderText('First Name'), {
      target: { value: 'John' }
    });
    fireEvent.change(screen.getByPlaceholderText('Last Name'), {
      target: { value: 'Doe' }
    });
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'john@example.com' }
    });
    
    fireEvent.click(screen.getByText('Next'));
    
    expect(screen.getByText('Address')).toBeInTheDocument();
  });

  test('goes back to previous step', () => {
    render(<MultiStepForm />);
    
    // Fill step 1 and go to step 2
    fireEvent.change(screen.getByPlaceholderText('First Name'), {
      target: { value: 'John' }
    });
    fireEvent.change(screen.getByPlaceholderText('Last Name'), {
      target: { value: 'Doe' }
    });
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'john@example.com' }
    });
    fireEvent.click(screen.getByText('Next'));
    
    // Go back
    fireEvent.click(screen.getByText('Back'));
    
    expect(screen.getByText('Personal Information')).toBeInTheDocument();
    expect(screen.getByPlaceholderText('First Name')).toHaveValue('John');
  });

  test('validates zip code format', () => {
    render(<MultiStepForm />);
    
    // Navigate to step 2
    fireEvent.change(screen.getByPlaceholderText('First Name'), {
      target: { value: 'John' }
    });
    fireEvent.change(screen.getByPlaceholderText('Last Name'), {
      target: { value: 'Doe' }
    });
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'john@example.com' }
    });
    fireEvent.click(screen.getByText('Next'));
    
    // Invalid zip
    fireEvent.change(screen.getByPlaceholderText('Zip Code'), {
      target: { value: '123' }
    });
    fireEvent.click(screen.getByText('Next'));
    
    expect(screen.getByText('Valid 5-digit zip required')).toBeInTheDocument();
  });

  test('shows submit button on last step', () => {
    render(<MultiStepForm />);
    
    // Navigate to last step
    // ... fill all steps ...
    
    expect(screen.getByText('Submit')).toBeInTheDocument();
    expect(screen.queryByText('Next')).not.toBeInTheDocument();
  });

  test('clears error when field is corrected', () => {
    render(<MultiStepForm />);
    
    fireEvent.click(screen.getByText('Next'));
    expect(screen.getByText('First name required')).toBeInTheDocument();
    
    fireEvent.change(screen.getByPlaceholderText('First Name'), {
      target: { value: 'John' }
    });
    
    expect(screen.queryByText('First name required')).not.toBeInTheDocument();
  });

  test('progress indicator shows current step', () => {
    render(<MultiStepForm />);
    
    const steps = screen.getAllByText(/\d/);
    expect(steps[0]).toHaveClass('active');
    expect(steps[1]).not.toHaveClass('active');
  });
});
```

## MANG Interview Follow-ups

**Q: How would you add form persistence (save draft)?**
```jsx
// Save to localStorage on every change
useEffect(() => {
  localStorage.setItem('formDraft', JSON.stringify(formData));
}, [formData]);

// Load on mount
const [formData, setFormData] = useState(() => {
  const saved = localStorage.getItem('formDraft');
  return saved ? JSON.parse(saved) : initialFormData;
});

// Clear on submit
const handleSubmit = () => {
  // ... submit logic
  localStorage.removeItem('formDraft');
};
```

**Q: How would you add conditional fields?**
```jsx
// Show field based on previous answer
{formData.country === 'USA' && (
  <input
    placeholder="State"
    value={formData.state}
    onChange={(e) => updateField('state', e.target.value)}
  />
)}

// Update validation logic
if (formData.country === 'USA' && !formData.state) {
  newErrors.state = 'State required for USA';
}
```

**Q: How would you add async validation (check email availability)?**
```jsx
const [isValidating, setIsValidating] = useState(false);

const validateEmail = async (email) => {
  setIsValidating(true);
  try {
    const response = await fetch(`/api/check-email?email=${email}`);
    const { available } = await response.json();
    if (!available) {
      setErrors(prev => ({ ...prev, email: 'Email already taken' }));
    }
  } finally {
    setIsValidating(false);
  }
};

// Debounce the validation
useEffect(() => {
  const timer = setTimeout(() => {
    if (formData.email.includes('@')) {
      validateEmail(formData.email);
    }
  }, 500);
  return () => clearTimeout(timer);
}, [formData.email]);
```

**Q: How would you add a summary/review step?**
```jsx
{currentStep === totalSteps + 1 && (
  <div className="summary">
    <h2>Review Your Information</h2>
    <div>
      <strong>Name:</strong> {formData.firstName} {formData.lastName}
    </div>
    <div>
      <strong>Email:</strong> {formData.email}
    </div>
    <div>
      <strong>Address:</strong> {formData.address}, {formData.city} {formData.zipCode}
    </div>
    <button onClick={() => setCurrentStep(1)}>Edit</button>
    <button onClick={handleSubmit}>Confirm & Submit</button>
  </div>
)}
```

**Q: How would you make this accessible?**
```jsx
<form
  role="form"
  aria-label="Multi-step registration form"
  aria-describedby="form-instructions"
>
  <div id="form-instructions" className="sr-only">
    Step {currentStep} of {totalSteps}
  </div>
  
  <input
    aria-label="First name"
    aria-required="true"
    aria-invalid={errors.firstName ? 'true' : 'false'}
    aria-describedby={errors.firstName ? 'firstName-error' : undefined}
  />
  
  {errors.firstName && (
    <span id="firstName-error" role="alert">
      {errors.firstName}
    </span>
  )}
</form>
```

## Tricky Parts Explained

**1. Validation Timing**
```jsx
// TRICKY: When to validate?
// Option 1: On blur (better UX)
// Option 2: On submit (simpler)
// Option 3: Real-time (can be annoying)

// Clear error on change (good UX)
if (errors[field]) {
  setErrors(prev => ({ ...prev, [field]: '' }));
}
```

**2. State Persistence Across Steps**
```jsx
// TRICKY: Keep all form data in single state object
// Don't create separate state for each step
// This allows easy back navigation and summary view
```

**3. Progress Indicator**
```jsx
// TRICKY: Visual feedback for completed steps
className={`step ${i + 1 <= currentStep ? 'active' : ''}`}
// Could also add 'completed' class for steps before current
```
