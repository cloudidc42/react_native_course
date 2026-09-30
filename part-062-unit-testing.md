# Part 062: Unit Testing ด้วย Jest ใน React Native

## บทนำ

Unit Testing คือการทดสอบหน่วยเล็กๆ ของโค้ด (functions, components, hooks) แบบแยกส่วน เพื่อให้แน่ใจว่าแต่ละส่วนทำงานได้ถูกต้อง Jest เป็น testing framework ที่นิยมใช้กับ React Native มากที่สุด

---

## 1. ทำความเข้าใจ Unit Testing

### ทำไมต้องเขียน Tests?

```
- ค้นหา bugs ก่อนถึง production
- ให้ความมั่นใจในการ refactor โค้ด
- เป็น documentation ของโค้ด
- ลดเวลา debugging
- เพิ่มคุณภาพของโค้ด
```

### Testing Pyramid

```
           /\
          /  \
         / E2E\       <- น้อยที่สุด, แพงที่สุด
        /------\
       /        \
      /Integration\   <- ปานกลาง
     /------------\
    /              \
   /   Unit Tests   \  <- มากที่สุด, ถูกที่สุด
  /------------------\
```

---

## 2. การตั้งค่า Jest

### ติดตั้ง Dependencies

```bash
# Jest มาพร้อมกับ React Native แล้ว แต่ต้องติดตั้ง type definitions
npm install --save-dev jest @types/jest
npm install --save-dev @testing-library/react-native
npm install --save-dev @testing-library/jest-native
npm install --save-dev jest-expo  # ถ้าใช้ Expo

# สำหรับ mocking
npm install --save-dev jest-mock-extended
npm install --save-dev msw  # Mock Service Worker
```

### ตั้งค่า package.json

```json
{
  "jest": {
    "preset": "react-native",
    "setupFilesAfterFramework": [
      "@testing-library/jest-native/extend-expect",
      "./jest.setup.ts"
    ],
    "moduleFileExtensions": ["ts", "tsx", "js", "jsx", "json"],
    "transformIgnorePatterns": [
      "node_modules/(?!(react-native|@react-native|@react-navigation|react-native-vector-icons)/)"
    ],
    "moduleNameMapper": {
      "^@/(.*)$": "<rootDir>/src/$1"
    },
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.d.ts",
      "!src/**/index.ts"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 70,
        "functions": 70,
        "lines": 70,
        "statements": 70
      }
    }
  }
}
```

### jest.setup.ts

```typescript
import '@testing-library/jest-native/extend-expect';

// Mock AsyncStorage
jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

// Mock react-native-reanimated
jest.mock('react-native-reanimated', () => {
  const Reanimated = require('react-native-reanimated/mock');
  Reanimated.default.call = jest.fn();
  return Reanimated;
});

// Mock Navigation
jest.mock('@react-navigation/native', () => ({
  useNavigation: () => ({
    navigate: jest.fn(),
    goBack: jest.fn(),
    push: jest.fn(),
    pop: jest.fn(),
  }),
  useRoute: () => ({
    params: {},
  }),
  useFocusEffect: jest.fn(),
}));

// Global mocks
global.console = {
  ...console,
  error: jest.fn(),
  warn: jest.fn(),
};
```

---

## 3. Anatomy ของ Test Case

### โครงสร้างพื้นฐาน

```typescript
// รูปแบบ AAA (Arrange, Act, Assert)
describe('ComponentName', () => {
  // Group related tests
  
  beforeAll(() => {
    // รันครั้งเดียวก่อน tests ทั้งหมดใน describe block
  });
  
  beforeEach(() => {
    // รันก่อน test แต่ละตัว
  });
  
  afterEach(() => {
    // รันหลัง test แต่ละตัว
  });
  
  afterAll(() => {
    // รันครั้งเดียวหลัง tests ทั้งหมด
  });
  
  it('should do something specific', () => {
    // Arrange - เตรียม data
    const input = 'test';
    
    // Act - ทำการ
    const result = someFunction(input);
    
    // Assert - ตรวจสอบผล
    expect(result).toBe('expected');
  });
  
  test('another test', () => {
    // it และ test เหมือนกัน
  });
});
```

---

## 4. Testing Pure Functions

### ตัวอย่าง Utility Functions

**src/utils/validators.ts**:
```typescript
export const validateEmail = (email: string): boolean => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
};

export const validatePassword = (password: string): {
  isValid: boolean;
  errors: string[];
} => {
  const errors: string[] = [];
  
  if (password.length < 8) {
    errors.push('Password must be at least 8 characters');
  }
  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain at least one uppercase letter');
  }
  if (!/[0-9]/.test(password)) {
    errors.push('Password must contain at least one number');
  }
  if (!/[!@#$%^&*]/.test(password)) {
    errors.push('Password must contain at least one special character');
  }
  
  return {
    isValid: errors.length === 0,
    errors,
  };
};

export const formatPrice = (price: number, currency: string = 'USD'): string => {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
  }).format(price);
};

export const truncateText = (text: string, maxLength: number): string => {
  if (text.length <= maxLength) return text;
  return text.substring(0, maxLength - 3) + '...';
};
```

**src/utils/__tests__/validators.test.ts**:
```typescript
import {
  validateEmail,
  validatePassword,
  formatPrice,
  truncateText,
} from '../validators';

describe('validateEmail', () => {
  it('should return true for valid email', () => {
    expect(validateEmail('user@example.com')).toBe(true);
    expect(validateEmail('test.email+tag@domain.co.uk')).toBe(true);
  });

  it('should return false for invalid email', () => {
    expect(validateEmail('')).toBe(false);
    expect(validateEmail('notanemail')).toBe(false);
    expect(validateEmail('@domain.com')).toBe(false);
    expect(validateEmail('user@')).toBe(false);
    expect(validateEmail('user @domain.com')).toBe(false);
  });
});

describe('validatePassword', () => {
  it('should return valid for strong password', () => {
    const result = validatePassword('SecurePass1!');
    expect(result.isValid).toBe(true);
    expect(result.errors).toHaveLength(0);
  });

  it('should return errors for weak password', () => {
    const result = validatePassword('weak');
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('Password must be at least 8 characters');
    expect(result.errors).toContain('Password must contain at least one uppercase letter');
    expect(result.errors).toContain('Password must contain at least one number');
    expect(result.errors).toContain('Password must contain at least one special character');
  });

  it('should detect missing uppercase', () => {
    const result = validatePassword('lowercase1!');
    expect(result.errors).toContain('Password must contain at least one uppercase letter');
    expect(result.errors).not.toContain('Password must be at least 8 characters');
  });
});

describe('formatPrice', () => {
  it('should format USD currency correctly', () => {
    expect(formatPrice(10)).toBe('$10.00');
    expect(formatPrice(1234.56)).toBe('$1,234.56');
  });

  it('should format other currencies', () => {
    expect(formatPrice(100, 'EUR')).toMatch(/€/);
  });

  it('should handle zero', () => {
    expect(formatPrice(0)).toBe('$0.00');
  });
});

describe('truncateText', () => {
  it('should not truncate short text', () => {
    expect(truncateText('Hello', 10)).toBe('Hello');
    expect(truncateText('Hello', 5)).toBe('Hello');
  });

  it('should truncate long text with ellipsis', () => {
    expect(truncateText('Hello World', 8)).toBe('Hello...');
    expect(truncateText('This is a long text', 10)).toBe('This is...');
  });

  it('should handle edge cases', () => {
    expect(truncateText('', 5)).toBe('');
    expect(truncateText('Hi', 0)).toBe('...');
  });
});
```

---

## 5. Testing React Components

### Component ตัวอย่าง

**src/components/Button.tsx**:
```typescript
import React from 'react';
import {
  TouchableOpacity,
  Text,
  ActivityIndicator,
  StyleSheet,
  TouchableOpacityProps,
} from 'react-native';

interface ButtonProps extends TouchableOpacityProps {
  title: string;
  variant?: 'primary' | 'secondary' | 'danger';
  loading?: boolean;
  testID?: string;
}

const Button: React.FC<ButtonProps> = ({
  title,
  variant = 'primary',
  loading = false,
  disabled = false,
  onPress,
  testID = 'button',
  ...props
}) => {
  const buttonStyle = [
    styles.button,
    styles[variant],
    (disabled || loading) && styles.disabled,
  ];

  return (
    <TouchableOpacity
      style={buttonStyle}
      onPress={onPress}
      disabled={disabled || loading}
      testID={testID}
      accessibilityRole="button"
      accessibilityLabel={title}
      accessibilityState={{ disabled: disabled || loading }}
      {...props}
    >
      {loading ? (
        <ActivityIndicator
          color="#fff"
          testID={`${testID}-loading`}
        />
      ) : (
        <Text style={styles.text} testID={`${testID}-title`}>
          {title}
        </Text>
      )}
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  button: {
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
    minHeight: 48,
  },
  primary: {
    backgroundColor: '#2196F3',
  },
  secondary: {
    backgroundColor: '#9E9E9E',
  },
  danger: {
    backgroundColor: '#f44336',
  },
  disabled: {
    opacity: 0.5,
  },
  text: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default Button;
```

**src/components/__tests__/Button.test.tsx**:
```typescript
import React from 'react';
import { render, fireEvent, screen } from '@testing-library/react-native';
import Button from '../Button';

describe('Button Component', () => {
  const mockOnPress = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('Rendering', () => {
    it('renders correctly with required props', () => {
      render(<Button title="Click Me" onPress={mockOnPress} />);
      expect(screen.getByText('Click Me')).toBeTruthy();
    });

    it('renders with default primary variant', () => {
      const { getByTestId } = render(
        <Button title="Primary" onPress={mockOnPress} />
      );
      const button = getByTestId('button');
      expect(button.props.style).toEqual(
        expect.arrayContaining([
          expect.objectContaining({ backgroundColor: '#2196F3' }),
        ])
      );
    });

    it('renders secondary variant correctly', () => {
      const { getByTestId } = render(
        <Button title="Secondary" variant="secondary" onPress={mockOnPress} />
      );
      const button = getByTestId('button');
      expect(button.props.style).toEqual(
        expect.arrayContaining([
          expect.objectContaining({ backgroundColor: '#9E9E9E' }),
        ])
      );
    });

    it('renders danger variant correctly', () => {
      const { getByTestId } = render(
        <Button title="Delete" variant="danger" onPress={mockOnPress} />
      );
      const button = getByTestId('button');
      expect(button.props.style).toEqual(
        expect.arrayContaining([
          expect.objectContaining({ backgroundColor: '#f44336' }),
        ])
      );
    });
  });

  describe('Loading State', () => {
    it('shows loading indicator when loading is true', () => {
      const { getByTestId, queryByTestId } = render(
        <Button title="Submit" loading={true} onPress={mockOnPress} />
      );
      expect(getByTestId('button-loading')).toBeTruthy();
      expect(queryByTestId('button-title')).toBeNull();
    });

    it('hides loading indicator when loading is false', () => {
      const { getByTestId, queryByTestId } = render(
        <Button title="Submit" loading={false} onPress={mockOnPress} />
      );
      expect(getByTestId('button-title')).toBeTruthy();
      expect(queryByTestId('button-loading')).toBeNull();
    });

    it('disables press when loading', () => {
      const { getByTestId } = render(
        <Button title="Submit" loading={true} onPress={mockOnPress} />
      );
      fireEvent.press(getByTestId('button'));
      expect(mockOnPress).not.toHaveBeenCalled();
    });
  });

  describe('Disabled State', () => {
    it('does not call onPress when disabled', () => {
      const { getByTestId } = render(
        <Button title="Disabled" disabled={true} onPress={mockOnPress} />
      );
      fireEvent.press(getByTestId('button'));
      expect(mockOnPress).not.toHaveBeenCalled();
    });

    it('applies disabled style when disabled', () => {
      const { getByTestId } = render(
        <Button title="Disabled" disabled={true} onPress={mockOnPress} />
      );
      const button = getByTestId('button');
      expect(button.props.style).toEqual(
        expect.arrayContaining([
          expect.objectContaining({ opacity: 0.5 }),
        ])
      );
    });
  });

  describe('Interaction', () => {
    it('calls onPress when pressed', () => {
      const { getByTestId } = render(
        <Button title="Click Me" onPress={mockOnPress} />
      );
      fireEvent.press(getByTestId('button'));
      expect(mockOnPress).toHaveBeenCalledTimes(1);
    });

    it('calls onPress multiple times when pressed multiple times', () => {
      const { getByTestId } = render(
        <Button title="Multi Press" onPress={mockOnPress} />
      );
      const button = getByTestId('button');
      fireEvent.press(button);
      fireEvent.press(button);
      fireEvent.press(button);
      expect(mockOnPress).toHaveBeenCalledTimes(3);
    });
  });

  describe('Accessibility', () => {
    it('has correct accessibility role', () => {
      const { getByRole } = render(
        <Button title="Accessible Button" onPress={mockOnPress} />
      );
      expect(getByRole('button')).toBeTruthy();
    });

    it('has correct accessibility label', () => {
      const { getByLabelText } = render(
        <Button title="Submit Form" onPress={mockOnPress} />
      );
      expect(getByLabelText('Submit Form')).toBeTruthy();
    });

    it('marks as disabled in accessibility state', () => {
      const { getByTestId } = render(
        <Button title="Disabled" disabled={true} onPress={mockOnPress} />
      );
      const button = getByTestId('button');
      expect(button.props.accessibilityState.disabled).toBe(true);
    });
  });
});
```

---

## 6. Testing Custom Hooks

**src/hooks/useCounter.ts**:
```typescript
import { useState, useCallback } from 'react';

interface UseCounterOptions {
  initialValue?: number;
  min?: number;
  max?: number;
  step?: number;
}

interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setValue: (value: number) => void;
}

export const useCounter = (options: UseCounterOptions = {}): UseCounterReturn => {
  const { initialValue = 0, min, max, step = 1 } = options;
  const [count, setCount] = useState(initialValue);

  const increment = useCallback(() => {
    setCount(prev => {
      const newValue = prev + step;
      if (max !== undefined && newValue > max) return prev;
      return newValue;
    });
  }, [step, max]);

  const decrement = useCallback(() => {
    setCount(prev => {
      const newValue = prev - step;
      if (min !== undefined && newValue < min) return prev;
      return newValue;
    });
  }, [step, min]);

  const reset = useCallback(() => {
    setCount(initialValue);
  }, [initialValue]);

  const setValue = useCallback((value: number) => {
    if (min !== undefined && value < min) return;
    if (max !== undefined && value > max) return;
    setCount(value);
  }, [min, max]);

  return { count, increment, decrement, reset, setValue };
};
```

**src/hooks/__tests__/useCounter.test.ts**:
```typescript
import { renderHook, act } from '@testing-library/react-hooks';
import { useCounter } from '../useCounter';

describe('useCounter Hook', () => {
  it('initializes with default value of 0', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('initializes with custom value', () => {
    const { result } = renderHook(() => useCounter({ initialValue: 10 }));
    expect(result.current.count).toBe(10);
  });

  describe('increment', () => {
    it('increments by 1 by default', () => {
      const { result } = renderHook(() => useCounter());
      
      act(() => {
        result.current.increment();
      });
      
      expect(result.current.count).toBe(1);
    });

    it('increments by custom step', () => {
      const { result } = renderHook(() => useCounter({ step: 5 }));
      
      act(() => {
        result.current.increment();
      });
      
      expect(result.current.count).toBe(5);
    });

    it('does not exceed max value', () => {
      const { result } = renderHook(() => useCounter({ initialValue: 9, max: 10 }));
      
      act(() => {
        result.current.increment();
        result.current.increment(); // Should be blocked
      });
      
      expect(result.current.count).toBe(10);
    });
  });

  describe('decrement', () => {
    it('decrements by 1 by default', () => {
      const { result } = renderHook(() => useCounter({ initialValue: 5 }));
      
      act(() => {
        result.current.decrement();
      });
      
      expect(result.current.count).toBe(4);
    });

    it('does not go below min value', () => {
      const { result } = renderHook(() => useCounter({ initialValue: 1, min: 0 }));
      
      act(() => {
        result.current.decrement();
        result.current.decrement(); // Should be blocked
      });
      
      expect(result.current.count).toBe(0);
    });
  });

  describe('reset', () => {
    it('resets to initial value', () => {
      const { result } = renderHook(() => useCounter({ initialValue: 5 }));
      
      act(() => {
        result.current.increment();
        result.current.increment();
        result.current.reset();
      });
      
      expect(result.current.count).toBe(5);
    });
  });

  describe('setValue', () => {
    it('sets specific value', () => {
      const { result } = renderHook(() => useCounter());
      
      act(() => {
        result.current.setValue(42);
      });
      
      expect(result.current.count).toBe(42);
    });

    it('respects min/max constraints', () => {
      const { result } = renderHook(() => useCounter({ min: 0, max: 100 }));
      
      act(() => {
        result.current.setValue(-5);
      });
      expect(result.current.count).toBe(0); // ไม่เปลี่ยน
      
      act(() => {
        result.current.setValue(150);
      });
      expect(result.current.count).toBe(0); // ไม่เปลี่ยน
      
      act(() => {
        result.current.setValue(50);
      });
      expect(result.current.count).toBe(50);
    });
  });
});
```

---

## 7. Mocking

### Mock Modules

```typescript
// __mocks__/react-native-device-info.ts
export default {
  getVersion: jest.fn(() => '1.0.0'),
  getBuildNumber: jest.fn(() => '1'),
  getDeviceId: jest.fn(() => 'test-device-id'),
  isEmulator: jest.fn(() => Promise.resolve(true)),
};

// Mock navigation ใน test
jest.mock('@react-navigation/native', () => ({
  ...jest.requireActual('@react-navigation/native'),
  useNavigation: () => ({
    navigate: jest.fn(),
    goBack: jest.fn(),
  }),
  useRoute: () => ({
    params: { id: '123' },
  }),
}));
```

### Mock API Calls

**src/api/userApi.ts**:
```typescript
import axios from 'axios';

export interface User {
  id: string;
  name: string;
  email: string;
  avatar?: string;
}

export const fetchUser = async (id: string): Promise<User> => {
  const response = await axios.get(`/api/users/${id}`);
  return response.data;
};

export const updateUser = async (id: string, data: Partial<User>): Promise<User> => {
  const response = await axios.put(`/api/users/${id}`, data);
  return response.data;
};
```

**src/api/__tests__/userApi.test.ts**:
```typescript
import axios from 'axios';
import { fetchUser, updateUser } from '../userApi';

// Mock axios
jest.mock('axios');
const mockedAxios = axios as jest.Mocked<typeof axios>;

describe('userApi', () => {
  const mockUser = {
    id: '123',
    name: 'John Doe',
    email: 'john@example.com',
  };

  describe('fetchUser', () => {
    it('fetches user successfully', async () => {
      mockedAxios.get.mockResolvedValueOnce({ data: mockUser });
      
      const user = await fetchUser('123');
      
      expect(mockedAxios.get).toHaveBeenCalledWith('/api/users/123');
      expect(user).toEqual(mockUser);
    });

    it('throws error on failed request', async () => {
      mockedAxios.get.mockRejectedValueOnce(new Error('Network Error'));
      
      await expect(fetchUser('123')).rejects.toThrow('Network Error');
    });
  });

  describe('updateUser', () => {
    it('updates user successfully', async () => {
      const updatedUser = { ...mockUser, name: 'Jane Doe' };
      mockedAxios.put.mockResolvedValueOnce({ data: updatedUser });
      
      const user = await updateUser('123', { name: 'Jane Doe' });
      
      expect(mockedAxios.put).toHaveBeenCalledWith('/api/users/123', { name: 'Jane Doe' });
      expect(user.name).toBe('Jane Doe');
    });
  });
});
```

---

## 8. Testing Async Code

```typescript
// Testing promises
it('handles async operations', async () => {
  const data = await fetchData();
  expect(data).toBeDefined();
});

// Testing with fake timers
it('calls function after delay', () => {
  jest.useFakeTimers();
  const callback = jest.fn();
  
  setTimeout(callback, 1000);
  
  expect(callback).not.toHaveBeenCalled();
  
  jest.advanceTimersByTime(1000);
  
  expect(callback).toHaveBeenCalledTimes(1);
  
  jest.useRealTimers();
});

// Testing rejected promises
it('handles errors gracefully', async () => {
  jest.spyOn(global, 'fetch').mockRejectedValueOnce(new Error('API Error'));
  
  await expect(fetchData()).rejects.toThrow('API Error');
});
```

---

## 9. Snapshot Testing

```typescript
import React from 'react';
import renderer from 'react-test-renderer';
import UserCard from '../UserCard';

describe('UserCard Snapshot', () => {
  const mockUser = {
    id: '1',
    name: 'John Doe',
    email: 'john@example.com',
    avatar: 'https://example.com/avatar.jpg',
  };

  it('renders correctly', () => {
    const tree = renderer.create(
      <UserCard user={mockUser} onPress={() => {}} />
    ).toJSON();
    
    expect(tree).toMatchSnapshot();
  });

  it('renders without avatar', () => {
    const userWithoutAvatar = { ...mockUser, avatar: undefined };
    const tree = renderer.create(
      <UserCard user={userWithoutAvatar} onPress={() => {}} />
    ).toJSON();
    
    expect(tree).toMatchSnapshot();
  });
});
```

---

## 10. Code Coverage

### การรันและดู Coverage

```bash
# รัน tests พร้อม coverage
npm test -- --coverage

# รัน tests แบบ watch mode
npm test -- --watch

# รันเฉพาะไฟล์ที่เปลี่ยน
npm test -- --watchAll=false

# รันเฉพาะ test file
npm test -- Button.test.tsx
```

### ตัวอย่าง Coverage Report

```
--------------------------|---------|----------|---------|---------|
File                      | % Stmts | % Branch | % Funcs | % Lines |
--------------------------|---------|----------|---------|---------|
 src/components/          |   95.2  |   89.3   |   100   |   95.1  |
   Button.tsx             |   100   |   100    |   100   |   100   |
   UserCard.tsx           |   90.5  |   78.6   |   100   |   90.2  |
 src/hooks/               |   98.1  |   92.3   |   100   |   98.0  |
   useCounter.ts          |   100   |   100    |   100   |   100   |
 src/utils/               |   96.4  |   93.8   |   95.2  |   96.3  |
   validators.ts          |   100   |   100    |   100   |   100   |
   formatters.ts          |   92.9  |   87.5   |   90.5  |   92.7  |
--------------------------|---------|----------|---------|---------|
```

---

## Workshop: Test Suite for Components

### Form Component ที่จะทดสอบ

**src/components/LoginForm.tsx**:
```typescript
import React, { useState } from 'react';
import {
  View,
  TextInput,
  Text,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
} from 'react-native';
import { validateEmail, validatePassword } from '../utils/validators';

interface LoginFormProps {
  onSubmit: (email: string, password: string) => Promise<void>;
}

const LoginForm: React.FC<LoginFormProps> = ({ onSubmit }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<{ email?: string; password?: string }>({});
  const [loading, setLoading] = useState(false);
  const [submitted, setSubmitted] = useState(false);

  const validate = (): boolean => {
    const newErrors: { email?: string; password?: string } = {};
    
    if (!email) {
      newErrors.email = 'Email is required';
    } else if (!validateEmail(email)) {
      newErrors.email = 'Please enter a valid email';
    }
    
    if (!password) {
      newErrors.password = 'Password is required';
    }
    
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async () => {
    if (!validate()) return;
    
    try {
      setLoading(true);
      await onSubmit(email, password);
      setSubmitted(true);
    } catch (error) {
      setErrors({ email: 'Login failed. Please try again.' });
    } finally {
      setLoading(false);
    }
  };

  if (submitted) {
    return (
      <View testID="success-message" style={styles.successContainer}>
        <Text style={styles.successText}>Login successful!</Text>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <View style={styles.inputContainer}>
        <TextInput
          testID="email-input"
          style={[styles.input, errors.email && styles.inputError]}
          placeholder="Email"
          value={email}
          onChangeText={setEmail}
          autoCapitalize="none"
          keyboardType="email-address"
          accessibilityLabel="Email input"
        />
        {errors.email && (
          <Text testID="email-error" style={styles.errorText}>
            {errors.email}
          </Text>
        )}
      </View>

      <View style={styles.inputContainer}>
        <TextInput
          testID="password-input"
          style={[styles.input, errors.password && styles.inputError]}
          placeholder="Password"
          value={password}
          onChangeText={setPassword}
          secureTextEntry
          accessibilityLabel="Password input"
        />
        {errors.password && (
          <Text testID="password-error" style={styles.errorText}>
            {errors.password}
          </Text>
        )}
      </View>

      <TouchableOpacity
        testID="submit-button"
        style={[styles.button, loading && styles.buttonDisabled]}
        onPress={handleSubmit}
        disabled={loading}
      >
        {loading ? (
          <ActivityIndicator color="#fff" testID="loading-indicator" />
        ) : (
          <Text style={styles.buttonText}>Sign In</Text>
        )}
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 20 },
  inputContainer: { marginBottom: 16 },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
  },
  inputError: { borderColor: '#f44336' },
  errorText: { color: '#f44336', fontSize: 12, marginTop: 4 },
  button: {
    backgroundColor: '#2196F3',
    borderRadius: 8,
    padding: 16,
    alignItems: 'center',
  },
  buttonDisabled: { opacity: 0.7 },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  successContainer: { padding: 20, alignItems: 'center' },
  successText: { fontSize: 18, color: '#4CAF50', fontWeight: 'bold' },
});

export default LoginForm;
```

**src/components/__tests__/LoginForm.test.tsx**:
```typescript
import React from 'react';
import { render, fireEvent, waitFor, screen } from '@testing-library/react-native';
import LoginForm from '../LoginForm';

describe('LoginForm', () => {
  const mockOnSubmit = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('Rendering', () => {
    it('renders all form elements', () => {
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      expect(screen.getByTestId('email-input')).toBeTruthy();
      expect(screen.getByTestId('password-input')).toBeTruthy();
      expect(screen.getByTestId('submit-button')).toBeTruthy();
    });
  });

  describe('Validation', () => {
    it('shows error for empty email', async () => {
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('email-error')).toBeTruthy();
        expect(screen.getByText('Email is required')).toBeTruthy();
      });
    });

    it('shows error for invalid email format', async () => {
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.changeText(screen.getByTestId('email-input'), 'invalid-email');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByText('Please enter a valid email')).toBeTruthy();
      });
    });

    it('shows error for empty password', async () => {
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.changeText(screen.getByTestId('email-input'), 'valid@email.com');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('password-error')).toBeTruthy();
      });
    });

    it('does not show errors for valid input', async () => {
      mockOnSubmit.mockResolvedValueOnce(undefined);
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.changeText(screen.getByTestId('email-input'), 'valid@email.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'password123');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.queryByTestId('email-error')).toBeNull();
        expect(screen.queryByTestId('password-error')).toBeNull();
      });
    });
  });

  describe('Submission', () => {
    it('calls onSubmit with correct credentials', async () => {
      mockOnSubmit.mockResolvedValueOnce(undefined);
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.changeText(screen.getByTestId('email-input'), 'user@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'mypassword');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(mockOnSubmit).toHaveBeenCalledWith('user@example.com', 'mypassword');
      });
    });

    it('shows loading state during submission', async () => {
      mockOnSubmit.mockImplementation(
        () => new Promise(resolve => setTimeout(resolve, 100))
      );
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.changeText(screen.getByTestId('email-input'), 'user@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'password');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      expect(screen.getByTestId('loading-indicator')).toBeTruthy();
      
      await waitFor(() => {
        expect(screen.queryByTestId('loading-indicator')).toBeNull();
      });
    });

    it('shows success message after successful login', async () => {
      mockOnSubmit.mockResolvedValueOnce(undefined);
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.changeText(screen.getByTestId('email-input'), 'user@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'password');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('success-message')).toBeTruthy();
        expect(screen.getByText('Login successful!')).toBeTruthy();
      });
    });

    it('shows error message on failed login', async () => {
      mockOnSubmit.mockRejectedValueOnce(new Error('Invalid credentials'));
      render(<LoginForm onSubmit={mockOnSubmit} />);
      
      fireEvent.changeText(screen.getByTestId('email-input'), 'user@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'wrong-password');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByText('Login failed. Please try again.')).toBeTruthy();
      });
    });
  });
});
```

---

## Tips และ Best Practices

### 1. ตั้งชื่อ Tests ที่ดี

```typescript
// ไม่ดี
it('test button', () => {});

// ดี
it('should disable button when loading prop is true', () => {});
it('should call onPress handler when button is pressed', () => {});
```

### 2. หลีกเลี่ยง Implementation Details

```typescript
// ไม่ดี - ทดสอบ state internal
expect(component.state.count).toBe(1);

// ดี - ทดสอบ behavior ที่ user เห็น
expect(screen.getByText('Count: 1')).toBeTruthy();
```

### 3. ใช้ Test Data Factories

```typescript
// factories/userFactory.ts
export const createUser = (overrides = {}) => ({
  id: '1',
  name: 'Test User',
  email: 'test@example.com',
  ...overrides,
});

// ใน tests
const user = createUser({ name: 'Custom Name' });
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Jest Configuration**: การตั้งค่า testing environment
2. **Test Structure**: รูปแบบ AAA pattern
3. **Component Testing**: ทดสอบ React components ด้วย Testing Library
4. **Hook Testing**: ทดสอบ custom hooks
5. **Mocking**: Mock modules และ API calls
6. **Code Coverage**: วัดความครอบคลุมของ tests
