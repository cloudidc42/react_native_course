# Part 029: Testing เบื้องต้น

## บทนำ

การทดสอบ (Testing) เป็นส่วนสำคัญของการพัฒนา software ที่มีคุณภาพ ช่วยให้เรามั่นใจว่า code ทำงานถูกต้องและป้องกัน bugs ที่อาจเกิดขึ้นจากการเปลี่ยนแปลง code ในอนาคต

## สารบัญ

1. ทำไมต้องทดสอบ
2. Jest Basics
3. React Native Testing Library
4. Unit Tests
5. Snapshot Tests
6. Workshop: Test a Component

---

## 1. ทำไมต้องทดสอบ

### ประโยชน์ของการทดสอบ

**1. ความมั่นใจในการ Refactor**
```javascript
// ก่อน: ไม่มี test
// เมื่อ refactor แล้วไม่รู้ว่าจะพัง feature ไหน

// หลัง: มี test
// ถ้า test ผ่านทั้งหมด = ไม่มีอะไรพัง ✓
```

**2. Documentation**
```javascript
// test เป็นเหมือน documentation ที่ทันสมัยเสมอ
describe('formatPrice', () => {
  it('แสดงราคาเป็นบาทไทย', () => {
    expect(formatPrice(1000)).toBe('฿1,000');
    expect(formatPrice(1000000)).toBe('฿1,000,000');
    expect(formatPrice(0)).toBe('฿0');
  });
});
```

**3. ค้นหา Bugs เร็วขึ้น**
```javascript
// Bug ถูกจับในขั้นตอน test ก่อนที่จะไปถึง production
it('ต้องจัดการ negative values', () => {
  expect(calculateDiscount(-100)).toThrow('ราคาต้องเป็นบวก');
});
```

### ประเภทของ Tests

```
Testing Pyramid
         /\
        /  \          E2E Tests
       /    \         (น้อย แต่ครอบคลุม)
      /------\
     /        \       Integration Tests
    /          \      (ทดสอบ components ร่วมกัน)
   /------------\
  /              \    Unit Tests
 /                \   (มาก เร็ว แยกทดสอบ)
/------------------\
```

---

## 2. Jest Basics

Jest เป็น test framework ที่ Expo/React Native ใช้โดย default

### ติดตั้ง

```bash
# Expo มา Jest มาแล้ว แต่ถ้าไม่มี:
npm install --save-dev jest @types/jest babel-jest

# Jest config ใน package.json
{
  "jest": {
    "preset": "jest-expo",
    "setupFilesAfterFramework": ["<rootDir>/jest.setup.js"],
    "testMatch": [
      "**/__tests__/**/*.[jt]s?(x)",
      "**/?(*.)+(spec|test).[jt]s?(x)"
    ]
  }
}
```

### Structure ของ Test

```javascript
// มาตรฐานการเขียน test: Arrange, Act, Assert (AAA)

describe('ชื่อ group ของ tests', () => {
  // Setup ที่ทำก่อนทุก test
  beforeAll(() => {
    // ทำครั้งเดียวก่อน tests ทั้งหมดใน describe
  });

  beforeEach(() => {
    // ทำก่อนแต่ละ test
  });

  afterEach(() => {
    // ทำหลังแต่ละ test
    jest.clearAllMocks();
  });

  afterAll(() => {
    // ทำครั้งเดียวหลัง tests ทั้งหมดใน describe
  });

  it('ชื่อ test (ควรอธิบายว่า expect อะไร)', () => {
    // Arrange - เตรียมข้อมูล
    const input = 100;
    
    // Act - ทำสิ่งที่ต้องการทดสอบ
    const result = calculateTax(input);
    
    // Assert - ตรวจสอบผลลัพธ์
    expect(result).toBe(7);
  });
});
```

### Matchers ที่ใช้บ่อย

```javascript
// Equality
expect(2 + 2).toBe(4);                         // strict equality (===)
expect({ a: 1 }).toEqual({ a: 1 });            // deep equality
expect({ a: 1, b: 2 }).toMatchObject({ a: 1 }); // partial match

// Truthiness
expect(true).toBeTruthy();
expect(false).toBeFalsy();
expect(null).toBeNull();
expect(undefined).toBeUndefined();
expect(0).toBeDefined();

// Numbers
expect(5).toBeGreaterThan(3);
expect(2).toBeLessThan(5);
expect(0.1 + 0.2).toBeCloseTo(0.3);

// Strings
expect('Hello World').toContain('World');
expect('foo123').toMatch(/\d+/);

// Arrays
expect([1, 2, 3]).toContain(2);
expect([1, 2, 3]).toHaveLength(3);
expect([1, 2, 3]).toEqual(expect.arrayContaining([1, 3]));

// Objects
expect({ name: 'John', age: 30 }).toHaveProperty('name', 'John');

// Functions / Errors
expect(() => {
  throw new Error('ข้อผิดพลาด');
}).toThrow('ข้อผิดพลาด');

expect(() => {
  throw new Error('ข้อผิดพลาด');
}).toThrow(Error);

// Async
await expect(fetchUser(1)).resolves.toEqual({ id: 1, name: 'John' });
await expect(fetchUser(-1)).rejects.toThrow('User not found');
```

### Mocking

```javascript
// Mock function
const mockFn = jest.fn();
mockFn('hello');
expect(mockFn).toHaveBeenCalledWith('hello');
expect(mockFn).toHaveBeenCalledTimes(1);

// Mock return values
const mockAdd = jest.fn().mockReturnValue(10);
expect(mockAdd(2, 3)).toBe(10);

// Mock implementation
const mockFetch = jest.fn().mockImplementation((url) => {
  if (url.includes('error')) {
    return Promise.reject(new Error('Not found'));
  }
  return Promise.resolve({ data: 'success' });
});

// Mock modules
jest.mock('./api/users', () => ({
  fetchUsers: jest.fn().mockResolvedValue([
    { id: 1, name: 'สมชาย' },
    { id: 2, name: 'สมหญิง' },
  ]),
}));

// Mock React Native modules
jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

jest.mock('react-native/Libraries/Animated/NativeAnimatedHelper');

// Spy on methods
const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});
// ทำ action
expect(consoleSpy).toHaveBeenCalledWith('expected error');
consoleSpy.mockRestore();
```

---

## 3. React Native Testing Library

React Native Testing Library (RNTL) ช่วยทดสอบ components ในลักษณะที่ใกล้เคียงกับที่ผู้ใช้จริงๆ ใช้งาน

### ติดตั้ง

```bash
npx expo install @testing-library/react-native @testing-library/jest-native
```

```javascript
// jest.setup.js
import '@testing-library/jest-native/extend-expect';
```

### Queries ที่ใช้บ่อย

```javascript
import { render, screen } from '@testing-library/react-native';

// getBy* - throw ถ้าไม่เจอหรือเจอมากกว่า 1
screen.getByText('Hello');
screen.getByRole('button', { name: 'ส่ง' });
screen.getByTestId('username-input');
screen.getByPlaceholderText('กรอกอีเมล');
screen.getByDisplayValue('current value');
screen.getByLabelText('Email');

// queryBy* - return null ถ้าไม่เจอ
screen.queryByText('Hidden Element');  // returns null, ไม่ throw

// findBy* - async, return promise
await screen.findByText('Loaded Data');  // รอจนกว่าจะแสดง

// getAllBy*, queryAllBy*, findAllBy* - return array
screen.getAllByRole('listitem');
```

### Events

```javascript
import { render, screen, fireEvent } from '@testing-library/react-native';
import userEvent from '@testing-library/user-event';

// fireEvent (ทันที)
fireEvent.press(button);
fireEvent.changeText(input, 'new text');
fireEvent.scroll(scrollView, { nativeEvent: { contentOffset: { y: 100 } } });

// userEvent (realistic, แนะนำ)
const user = userEvent.setup();
await user.press(button);
await user.type(input, 'hello world');
await user.clear(input);
```

---

## 4. Unit Tests

Unit tests ทดสอบ function เดี่ยวๆ หรือ logic ขนาดเล็ก

```javascript
// utils/formatters.js
export const formatPrice = (price, currency = 'THB') => {
  if (typeof price !== 'number') throw new TypeError('price ต้องเป็น number');
  if (price < 0) throw new RangeError('price ต้องไม่ติดลบ');

  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency,
    minimumFractionDigits: 0,
    maximumFractionDigits: 2,
  }).format(price);
};

export const calculateDiscount = (price, discountPercent) => {
  const discount = price * (discountPercent / 100);
  return {
    originalPrice: price,
    discountAmount: discount,
    finalPrice: price - discount,
    discountPercent,
  };
};

export const validateEmail = (email) => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
};

export const formatDate = (date, locale = 'th-TH') => {
  if (!(date instanceof Date)) throw new TypeError('ต้องเป็น Date object');
  return date.toLocaleDateString(locale, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  });
};
```

```javascript
// __tests__/utils/formatters.test.js
import {
  formatPrice,
  calculateDiscount,
  validateEmail,
  formatDate,
} from '../../utils/formatters';

describe('formatPrice', () => {
  it('แสดงราคาพร้อมสัญลักษณ์เงินบาท', () => {
    expect(formatPrice(1000)).toBe('฿1,000');
  });

  it('แสดงราคาที่มีทศนิยม', () => {
    expect(formatPrice(99.5)).toBe('฿99.50');
  });

  it('แสดงราคา 0', () => {
    expect(formatPrice(0)).toBe('฿0');
  });

  it('แสดงราคาใหญ่', () => {
    expect(formatPrice(1000000)).toBe('฿1,000,000');
  });

  it('รองรับสกุลเงิน USD', () => {
    expect(formatPrice(50, 'USD')).toMatch(/\$50/);
  });

  it('throw error เมื่อ price ไม่ใช่ number', () => {
    expect(() => formatPrice('abc')).toThrow(TypeError);
    expect(() => formatPrice(null)).toThrow(TypeError);
  });

  it('throw error เมื่อ price ติดลบ', () => {
    expect(() => formatPrice(-100)).toThrow(RangeError);
    expect(() => formatPrice(-1)).toThrow('price ต้องไม่ติดลบ');
  });
});

describe('calculateDiscount', () => {
  it('คำนวณส่วนลด 10% ถูกต้อง', () => {
    const result = calculateDiscount(1000, 10);
    expect(result).toEqual({
      originalPrice: 1000,
      discountAmount: 100,
      finalPrice: 900,
      discountPercent: 10,
    });
  });

  it('คำนวณส่วนลด 50% ถูกต้อง', () => {
    const result = calculateDiscount(500, 50);
    expect(result.finalPrice).toBe(250);
    expect(result.discountAmount).toBe(250);
  });

  it('ส่วนลด 0% ไม่เปลี่ยนราคา', () => {
    const result = calculateDiscount(1000, 0);
    expect(result.finalPrice).toBe(1000);
    expect(result.discountAmount).toBe(0);
  });
});

describe('validateEmail', () => {
  const validEmails = [
    'user@example.com',
    'user.name@example.co.th',
    'user+tag@example.org',
  ];

  const invalidEmails = [
    'notanemail',
    '@example.com',
    'user@',
    'user @example.com',
    '',
  ];

  validEmails.forEach((email) => {
    it(`ยอมรับ "${email}"`, () => {
      expect(validateEmail(email)).toBe(true);
    });
  });

  invalidEmails.forEach((email) => {
    it(`ปฏิเสธ "${email}"`, () => {
      expect(validateEmail(email)).toBe(false);
    });
  });
});

describe('formatDate', () => {
  it('แสดงวันที่เป็นภาษาไทย', () => {
    const date = new Date('2025-01-15');
    const result = formatDate(date);
    expect(result).toContain('2025'); // หรือ 2568 (พ.ศ.)
  });

  it('throw error เมื่อไม่ใช่ Date object', () => {
    expect(() => formatDate('2025-01-15')).toThrow(TypeError);
    expect(() => formatDate(1705276800000)).toThrow(TypeError);
  });
});
```

### Testing Async Functions

```javascript
// services/userService.js
export const fetchUser = async (id) => {
  const response = await fetch(`https://api.example.com/users/${id}`);
  if (!response.ok) {
    throw new Error(`User ${id} not found`);
  }
  return response.json();
};

export const createUser = async (userData) => {
  if (!userData.email || !userData.name) {
    throw new Error('Email and name are required');
  }
  const response = await fetch('https://api.example.com/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(userData),
  });
  return response.json();
};
```

```javascript
// __tests__/services/userService.test.js
import { fetchUser, createUser } from '../../services/userService';

// Mock fetch globally
global.fetch = jest.fn();

describe('fetchUser', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('ดึงข้อมูล user สำเร็จ', async () => {
    // Arrange
    const mockUser = { id: 1, name: 'สมชาย', email: 'somchai@example.com' };
    global.fetch.mockResolvedValueOnce({
      ok: true,
      json: async () => mockUser,
    });

    // Act
    const user = await fetchUser(1);

    // Assert
    expect(user).toEqual(mockUser);
    expect(global.fetch).toHaveBeenCalledWith(
      'https://api.example.com/users/1'
    );
  });

  it('throw error เมื่อ user ไม่มี', async () => {
    global.fetch.mockResolvedValueOnce({
      ok: false,
      status: 404,
    });

    await expect(fetchUser(999)).rejects.toThrow('User 999 not found');
  });

  it('throw error เมื่อ network ล้มเหลว', async () => {
    global.fetch.mockRejectedValueOnce(new Error('Network error'));

    await expect(fetchUser(1)).rejects.toThrow('Network error');
  });
});

describe('createUser', () => {
  it('สร้าง user สำเร็จ', async () => {
    const newUser = { name: 'สมชาย', email: 'somchai@example.com' };
    const createdUser = { id: 1, ...newUser };

    global.fetch.mockResolvedValueOnce({
      ok: true,
      json: async () => createdUser,
    });

    const result = await createUser(newUser);
    expect(result).toEqual(createdUser);
  });

  it('throw error เมื่อไม่มี email', async () => {
    await expect(createUser({ name: 'สมชาย' }))
      .rejects.toThrow('Email and name are required');
  });
});
```

---

## 5. Snapshot Tests

Snapshot tests บันทึก output ของ component และ compare กับ snapshot ที่บันทึกไว้ก่อนหน้า

```javascript
// components/UserCard.js
import React from 'react';
import { View, Text, Image, StyleSheet } from 'react-native';

const UserCard = ({ user, isSelected = false }) => {
  return (
    <View style={[styles.card, isSelected && styles.selectedCard]} testID="user-card">
      <View style={styles.avatar}>
        <Text style={styles.avatarText}>
          {user.name.charAt(0).toUpperCase()}
        </Text>
      </View>
      <View style={styles.info}>
        <Text style={styles.name}>{user.name}</Text>
        <Text style={styles.email}>{user.email}</Text>
        {user.role && (
          <View style={styles.badge}>
            <Text style={styles.badgeText}>{user.role}</Text>
          </View>
        )}
      </View>
      {isSelected && <Text testID="checkmark">✓</Text>}
    </View>
  );
};

const styles = StyleSheet.create({
  card: { flexDirection: 'row', padding: 16, backgroundColor: '#fff', borderRadius: 12 },
  selectedCard: { borderWidth: 2, borderColor: '#007AFF' },
  avatar: { width: 44, height: 44, borderRadius: 22, backgroundColor: '#007AFF', justifyContent: 'center', alignItems: 'center' },
  avatarText: { color: '#fff', fontSize: 18, fontWeight: 'bold' },
  info: { flex: 1, marginLeft: 12 },
  name: { fontSize: 16, fontWeight: '600' },
  email: { fontSize: 13, color: '#666', marginTop: 2 },
  badge: { backgroundColor: '#007AFF', paddingHorizontal: 8, paddingVertical: 2, borderRadius: 10, alignSelf: 'flex-start', marginTop: 4 },
  badgeText: { color: '#fff', fontSize: 11 },
});

export default UserCard;
```

```javascript
// __tests__/components/UserCard.test.js
import React from 'react';
import { render, screen } from '@testing-library/react-native';
import UserCard from '../../components/UserCard';

const mockUser = {
  id: 1,
  name: 'สมชาย ไทย',
  email: 'somchai@example.com',
  role: 'Admin',
};

describe('UserCard - Snapshot Tests', () => {
  it('renders correctly (default)', () => {
    const { toJSON } = render(<UserCard user={mockUser} />);
    expect(toJSON()).toMatchSnapshot();
  });

  it('renders correctly (selected)', () => {
    const { toJSON } = render(<UserCard user={mockUser} isSelected={true} />);
    expect(toJSON()).toMatchSnapshot();
  });

  it('renders correctly (no role)', () => {
    const userWithoutRole = { ...mockUser, role: undefined };
    const { toJSON } = render(<UserCard user={userWithoutRole} />);
    expect(toJSON()).toMatchSnapshot();
  });
});

describe('UserCard - Interaction Tests', () => {
  it('แสดงชื่อและอีเมลของ user', () => {
    render(<UserCard user={mockUser} />);

    expect(screen.getByText('สมชาย ไทย')).toBeTruthy();
    expect(screen.getByText('somchai@example.com')).toBeTruthy();
  });

  it('แสดง role badge เมื่อมี role', () => {
    render(<UserCard user={mockUser} />);
    expect(screen.getByText('Admin')).toBeTruthy();
  });

  it('ไม่แสดง role badge เมื่อไม่มี role', () => {
    const userWithoutRole = { ...mockUser, role: undefined };
    render(<UserCard user={userWithoutRole} />);
    expect(screen.queryByText('Admin')).toBeNull();
  });

  it('แสดง checkmark เมื่อถูก select', () => {
    render(<UserCard user={mockUser} isSelected={true} />);
    expect(screen.getByTestId('checkmark')).toBeTruthy();
  });

  it('ไม่แสดง checkmark เมื่อไม่ถูก select', () => {
    render(<UserCard user={mockUser} isSelected={false} />);
    expect(screen.queryByTestId('checkmark')).toBeNull();
  });

  it('แสดงตัวแรกของชื่อใน avatar', () => {
    render(<UserCard user={mockUser} />);
    expect(screen.getByText('ส')).toBeTruthy();
  });
});

describe('UserCard - Accessibility', () => {
  it('มี testID', () => {
    render(<UserCard user={mockUser} />);
    expect(screen.getByTestId('user-card')).toBeTruthy();
  });
});
```

---

## 6. Workshop: Test a Component

สร้าง Login Form พร้อม Tests ครบครัน

```javascript
// components/LoginForm/index.js
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ActivityIndicator,
  StyleSheet,
  Alert,
} from 'react-native';

export const validateLoginForm = (email, password) => {
  const errors = {};

  if (!email.trim()) {
    errors.email = 'กรุณากรอกอีเมล';
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    errors.email = 'รูปแบบอีเมลไม่ถูกต้อง';
  }

  if (!password) {
    errors.password = 'กรุณากรอกรหัสผ่าน';
  } else if (password.length < 6) {
    errors.password = 'รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร';
  }

  return {
    isValid: Object.keys(errors).length === 0,
    errors,
  };
};

const LoginForm = ({ onLogin, onForgotPassword, testID }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState({});
  const [loading, setLoading] = useState(false);
  const [showPassword, setShowPassword] = useState(false);

  const handleSubmit = useCallback(async () => {
    const { isValid, errors: validationErrors } = validateLoginForm(email, password);

    if (!isValid) {
      setErrors(validationErrors);
      return;
    }

    setErrors({});
    setLoading(true);

    try {
      await onLogin(email, password);
    } catch (error) {
      setErrors({ general: error.message || 'เข้าสู่ระบบไม่สำเร็จ' });
    } finally {
      setLoading(false);
    }
  }, [email, password, onLogin]);

  return (
    <View style={styles.container} testID={testID || 'login-form'}>
      <Text style={styles.title}>เข้าสู่ระบบ</Text>

      {errors.general && (
        <View style={styles.errorBanner} testID="error-banner">
          <Text style={styles.errorBannerText}>{errors.general}</Text>
        </View>
      )}

      <View style={styles.field}>
        <Text style={styles.label}>อีเมล</Text>
        <TextInput
          style={[styles.input, errors.email && styles.inputError]}
          placeholder="email@example.com"
          value={email}
          onChangeText={(text) => {
            setEmail(text);
            if (errors.email) setErrors((prev) => ({ ...prev, email: undefined }));
          }}
          keyboardType="email-address"
          autoCapitalize="none"
          testID="email-input"
          accessibilityLabel="อีเมล"
        />
        {errors.email && (
          <Text style={styles.fieldError} testID="email-error">
            {errors.email}
          </Text>
        )}
      </View>

      <View style={styles.field}>
        <Text style={styles.label}>รหัสผ่าน</Text>
        <View style={styles.passwordContainer}>
          <TextInput
            style={[styles.input, styles.passwordInput, errors.password && styles.inputError]}
            placeholder="รหัสผ่าน"
            value={password}
            onChangeText={(text) => {
              setPassword(text);
              if (errors.password) setErrors((prev) => ({ ...prev, password: undefined }));
            }}
            secureTextEntry={!showPassword}
            testID="password-input"
            accessibilityLabel="รหัสผ่าน"
          />
          <TouchableOpacity
            onPress={() => setShowPassword(!showPassword)}
            testID="toggle-password"
            style={styles.eyeButton}
          >
            <Text>{showPassword ? '🙈' : '👁️'}</Text>
          </TouchableOpacity>
        </View>
        {errors.password && (
          <Text style={styles.fieldError} testID="password-error">
            {errors.password}
          </Text>
        )}
      </View>

      <TouchableOpacity
        onPress={onForgotPassword}
        testID="forgot-password"
        style={styles.forgotBtn}
      >
        <Text style={styles.forgotText}>ลืมรหัสผ่าน?</Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={[styles.submitBtn, loading && styles.disabledBtn]}
        onPress={handleSubmit}
        disabled={loading}
        testID="submit-button"
        accessibilityLabel="เข้าสู่ระบบ"
        accessibilityRole="button"
      >
        {loading ? (
          <ActivityIndicator color="#fff" testID="loading-indicator" />
        ) : (
          <Text style={styles.submitText}>เข้าสู่ระบบ</Text>
        )}
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 24 },
  title: { fontSize: 28, fontWeight: 'bold', marginBottom: 24 },
  errorBanner: { backgroundColor: '#FFF0F0', borderRadius: 8, padding: 12, marginBottom: 16, borderLeftWidth: 4, borderLeftColor: '#FF3B30' },
  errorBannerText: { color: '#FF3B30' },
  field: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', color: '#333', marginBottom: 8 },
  input: { borderWidth: 1, borderColor: '#e0e0e0', borderRadius: 10, padding: 14, fontSize: 16 },
  inputError: { borderColor: '#FF3B30' },
  fieldError: { color: '#FF3B30', fontSize: 12, marginTop: 4 },
  passwordContainer: { flexDirection: 'row', alignItems: 'center' },
  passwordInput: { flex: 1 },
  eyeButton: { position: 'absolute', right: 12, padding: 4 },
  forgotBtn: { alignSelf: 'flex-end', marginBottom: 24 },
  forgotText: { color: '#007AFF', fontSize: 14 },
  submitBtn: { backgroundColor: '#007AFF', padding: 16, borderRadius: 12, alignItems: 'center' },
  disabledBtn: { backgroundColor: '#ccc' },
  submitText: { color: '#fff', fontSize: 18, fontWeight: '600' },
});

export default LoginForm;
```

```javascript
// __tests__/components/LoginForm.test.js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import userEvent from '@testing-library/user-event';
import LoginForm, { validateLoginForm } from '../../components/LoginForm';

// ============================================
// Unit Tests สำหรับ validateLoginForm
// ============================================

describe('validateLoginForm', () => {
  describe('Email validation', () => {
    it('ผ่านเมื่ออีเมลถูกต้อง', () => {
      const { isValid, errors } = validateLoginForm('test@example.com', 'password123');
      expect(isValid).toBe(true);
      expect(errors.email).toBeUndefined();
    });

    it('error เมื่ออีเมลว่าง', () => {
      const { isValid, errors } = validateLoginForm('', 'password123');
      expect(isValid).toBe(false);
      expect(errors.email).toBe('กรุณากรอกอีเมล');
    });

    it('error เมื่ออีเมลรูปแบบผิด', () => {
      const { isValid, errors } = validateLoginForm('notanemail', 'password123');
      expect(isValid).toBe(false);
      expect(errors.email).toBe('รูปแบบอีเมลไม่ถูกต้อง');
    });
  });

  describe('Password validation', () => {
    it('ผ่านเมื่อรหัสผ่านถูกต้อง', () => {
      const { isValid } = validateLoginForm('test@example.com', 'password123');
      expect(isValid).toBe(true);
    });

    it('error เมื่อรหัสผ่านว่าง', () => {
      const { errors } = validateLoginForm('test@example.com', '');
      expect(errors.password).toBe('กรุณากรอกรหัสผ่าน');
    });

    it('error เมื่อรหัสผ่านสั้นกว่า 6 ตัว', () => {
      const { errors } = validateLoginForm('test@example.com', '123');
      expect(errors.password).toBe('รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร');
    });
  });
});

// ============================================
// Component Tests สำหรับ LoginForm
// ============================================

describe('LoginForm Component', () => {
  const mockOnLogin = jest.fn();
  const mockOnForgotPassword = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  const renderLoginForm = (props = {}) => {
    return render(
      <LoginForm
        onLogin={mockOnLogin}
        onForgotPassword={mockOnForgotPassword}
        {...props}
      />
    );
  };

  describe('Rendering', () => {
    it('แสดง form elements ครบ', () => {
      renderLoginForm();

      expect(screen.getByTestId('login-form')).toBeTruthy();
      expect(screen.getByTestId('email-input')).toBeTruthy();
      expect(screen.getByTestId('password-input')).toBeTruthy();
      expect(screen.getByTestId('submit-button')).toBeTruthy();
      expect(screen.getByTestId('forgot-password')).toBeTruthy();
      expect(screen.getByText('เข้าสู่ระบบ')).toBeTruthy();
    });

    it('Snapshot ตรงกัน', () => {
      const { toJSON } = renderLoginForm();
      expect(toJSON()).toMatchSnapshot();
    });
  });

  describe('User Interactions', () => {
    it('กรอก email ได้', () => {
      renderLoginForm();
      const emailInput = screen.getByTestId('email-input');

      fireEvent.changeText(emailInput, 'test@example.com');

      expect(emailInput.props.value).toBe('test@example.com');
    });

    it('กรอก password ได้', () => {
      renderLoginForm();
      const passwordInput = screen.getByTestId('password-input');

      fireEvent.changeText(passwordInput, 'password123');

      expect(passwordInput.props.value).toBe('password123');
    });

    it('toggle show/hide password', () => {
      renderLoginForm();
      const passwordInput = screen.getByTestId('password-input');
      const toggleBtn = screen.getByTestId('toggle-password');

      // เริ่มต้น: ซ่อน password
      expect(passwordInput.props.secureTextEntry).toBe(true);

      // กด toggle: แสดง password
      fireEvent.press(toggleBtn);
      expect(passwordInput.props.secureTextEntry).toBe(false);

      // กด toggle อีกครั้ง: ซ่อน password
      fireEvent.press(toggleBtn);
      expect(passwordInput.props.secureTextEntry).toBe(true);
    });

    it('กด forgot password เรียก onForgotPassword', () => {
      renderLoginForm();
      fireEvent.press(screen.getByTestId('forgot-password'));
      expect(mockOnForgotPassword).toHaveBeenCalledTimes(1);
    });
  });

  describe('Validation', () => {
    it('แสดง error เมื่อกด submit โดยไม่กรอก email', async () => {
      renderLoginForm();

      fireEvent.press(screen.getByTestId('submit-button'));

      await waitFor(() => {
        expect(screen.getByTestId('email-error')).toBeTruthy();
        expect(screen.getByText('กรุณากรอกอีเมล')).toBeTruthy();
      });
    });

    it('แสดง error เมื่อกด submit โดยไม่กรอก password', async () => {
      renderLoginForm();

      fireEvent.changeText(screen.getByTestId('email-input'), 'test@example.com');
      fireEvent.press(screen.getByTestId('submit-button'));

      await waitFor(() => {
        expect(screen.getByTestId('password-error')).toBeTruthy();
      });
    });

    it('clear error เมื่อกรอกข้อมูลที่ถูกต้อง', async () => {
      renderLoginForm();

      // กด submit เพื่อให้เกิด error
      fireEvent.press(screen.getByTestId('submit-button'));

      await waitFor(() => {
        expect(screen.getByTestId('email-error')).toBeTruthy();
      });

      // กรอกอีเมลที่ถูกต้อง
      fireEvent.changeText(screen.getByTestId('email-input'), 'test@example.com');

      await waitFor(() => {
        expect(screen.queryByTestId('email-error')).toBeNull();
      });
    });
  });

  describe('Form Submission', () => {
    it('เรียก onLogin พร้อม email และ password ที่ถูกต้อง', async () => {
      mockOnLogin.mockResolvedValueOnce({ success: true });
      renderLoginForm();

      fireEvent.changeText(screen.getByTestId('email-input'), 'test@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'password123');
      fireEvent.press(screen.getByTestId('submit-button'));

      await waitFor(() => {
        expect(mockOnLogin).toHaveBeenCalledWith('test@example.com', 'password123');
      });
    });

    it('แสดง loading indicator ระหว่าง submit', async () => {
      let resolveLogin;
      mockOnLogin.mockReturnValue(new Promise((resolve) => {
        resolveLogin = resolve;
      }));

      renderLoginForm();
      fireEvent.changeText(screen.getByTestId('email-input'), 'test@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'password123');
      fireEvent.press(screen.getByTestId('submit-button'));

      // ระหว่าง loading ควรเห็น indicator
      await waitFor(() => {
        expect(screen.getByTestId('loading-indicator')).toBeTruthy();
      });

      // resolve login
      resolveLogin({ success: true });

      // หลัง loading indicator หาย
      await waitFor(() => {
        expect(screen.queryByTestId('loading-indicator')).toBeNull();
      });
    });

    it('แสดง error banner เมื่อ login ล้มเหลว', async () => {
      mockOnLogin.mockRejectedValueOnce(new Error('อีเมลหรือรหัสผ่านไม่ถูกต้อง'));
      renderLoginForm();

      fireEvent.changeText(screen.getByTestId('email-input'), 'test@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'password123');
      fireEvent.press(screen.getByTestId('submit-button'));

      await waitFor(() => {
        expect(screen.getByTestId('error-banner')).toBeTruthy();
        expect(screen.getByText('อีเมลหรือรหัสผ่านไม่ถูกต้อง')).toBeTruthy();
      });
    });

    it('ปิดใช้งาน submit button ระหว่าง loading', async () => {
      let resolveLogin;
      mockOnLogin.mockReturnValue(new Promise((resolve) => {
        resolveLogin = resolve;
      }));

      renderLoginForm();
      fireEvent.changeText(screen.getByTestId('email-input'), 'test@example.com');
      fireEvent.changeText(screen.getByTestId('password-input'), 'password123');
      fireEvent.press(screen.getByTestId('submit-button'));

      await waitFor(() => {
        const button = screen.getByTestId('submit-button');
        expect(button.props.accessibilityState.disabled).toBe(true);
      });

      resolveLogin({ success: true });
    });
  });

  describe('Accessibility', () => {
    it('submit button มี accessibility label', () => {
      renderLoginForm();
      const button = screen.getByLabelText('เข้าสู่ระบบ');
      expect(button).toBeTruthy();
    });

    it('email input มี accessibility label', () => {
      renderLoginForm();
      const input = screen.getByLabelText('อีเมล');
      expect(input).toBeTruthy();
    });
  });
});
```

### รัน Tests

```bash
# รัน tests ทั้งหมด
npm test

# รัน tests แบบ watch mode
npm test -- --watch

# รัน test file เดียว
npm test LoginForm

# ดู coverage
npm test -- --coverage

# รัน test ที่ match pattern
npm test -- --testNamePattern="Validation"

# Verbose output
npm test -- --verbose
```

---

## Tips และ Best Practices

### 1. Test Naming Convention
```javascript
// ✅ ดี: อธิบายว่า expect อะไร
it('แสดง error เมื่อ email ว่าง', () => {});

// ❌ ไม่ดี: ไม่รู้ว่า test อะไร
it('test 1', () => {});
it('validation test', () => {});
```

### 2. ทดสอบ behavior ไม่ใช่ implementation
```javascript
// ✅ ทดสอบว่า user เห็นอะไร
expect(screen.getByText('เกิดข้อผิดพลาด')).toBeTruthy();

// ❌ ทดสอบ internal state (brittle)
expect(component.state.hasError).toBe(true);
```

### 3. หลีกเลี่ยง Tests ที่ซ้ำซ้อน
```javascript
// ✅ Test อิสระ
it('แสดง error เมื่อ email ว่าง', () => {...});
it('แสดง error เมื่อ password ว่าง', () => {...});

// ❌ Test ที่ต้องพึ่งกัน
it('กรอกข้อมูล', () => { fillForm(); });
it('submit form', () => { clickSubmit(); }); // พึ่ง test ก่อน!
```

### 4. Coverage Goals
```
Unit Tests: 80%+ coverage
Integration Tests: สำหรับ flows สำคัญ
E2E Tests: สำหรับ critical user journeys
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- ทำไมการ test ถึงสำคัญ
- Jest basics: matchers, mocking, async tests
- React Native Testing Library สำหรับ component tests
- Unit tests สำหรับ utility functions
- Snapshot tests สำหรับ visual regression
- Workshop: Login Form พร้อม tests ครบครัน

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Build และ Deploy เบื้องต้น
