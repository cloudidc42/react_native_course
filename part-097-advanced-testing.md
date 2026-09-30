# Part 097: Advanced Testing Strategies ใน React Native

## บทนำ

การทดสอบที่ดีช่วยให้มั่นใจว่า app ทำงานถูกต้อง เราจะเรียนรู้ TDD, BDD, Performance Testing, Visual Regression Testing, และ Test Automation

## หัวข้อที่จะเรียน

1. TDD กับ React Native
2. BDD
3. Performance Testing
4. Visual Regression
5. Test Automation
6. Workshop: Full Test Coverage

---

## 1. การตั้งค่า Testing Environment

```bash
# Core testing libraries
npm install --save-dev jest @testing-library/react-native
npm install --save-dev @testing-library/jest-native
npm install --save-dev jest-expo  # สำหรับ Expo

# Mocking
npm install --save-dev @jest/fake-timers
npm install --save-dev msw  # Mock Service Worker

# Visual testing
npm install --save-dev @storybook/addon-storyshots

# E2E
npm install --save-dev detox
```

### jest.config.js

```javascript
module.exports = {
  preset: '@testing-library/react-native',
  setupFilesAfterFramework: [
    '@testing-library/jest-native/extend-expect',
    './jest.setup.ts',
  ],
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?)/)',
  ],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '\\.svg$': '<rootDir>/__mocks__/svgMock.js',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.stories.{ts,tsx}',
    '!src/**/*.types.ts',
    '!src/index.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

### jest.setup.ts

```typescript
import '@testing-library/jest-native/extend-expect';
import { server } from './src/mocks/server';

// Setup MSW
beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Mock modules
jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

jest.mock('react-native-keychain', () => ({
  setGenericPassword: jest.fn().mockResolvedValue(true),
  getGenericPassword: jest.fn().mockResolvedValue({
    username: 'user',
    password: 'token',
  }),
  resetGenericPassword: jest.fn().mockResolvedValue(true),
}));

jest.mock('@react-navigation/native', () => ({
  ...jest.requireActual('@react-navigation/native'),
  useNavigation: () => ({
    navigate: jest.fn(),
    goBack: jest.fn(),
    push: jest.fn(),
    replace: jest.fn(),
  }),
  useRoute: () => ({
    params: {},
  }),
}));
```

---

## 2. TDD (Test-Driven Development)

### แนวทาง Red-Green-Refactor

```typescript
// 1. RED - เขียน test ที่ fail ก่อน
describe('CartService', () => {
  describe('addItem', () => {
    it('should add item to empty cart', () => {
      const cart = new CartService();
      const item = { id: '1', name: 'สินค้า A', price: 100, quantity: 1 };
      
      cart.addItem(item);
      
      expect(cart.items).toHaveLength(1);
      expect(cart.items[0]).toEqual(item);
    });

    it('should increase quantity if item already exists', () => {
      const cart = new CartService();
      const item = { id: '1', name: 'สินค้า A', price: 100, quantity: 1 };
      
      cart.addItem(item);
      cart.addItem(item);
      
      expect(cart.items).toHaveLength(1);
      expect(cart.items[0].quantity).toBe(2);
    });

    it('should calculate total correctly', () => {
      const cart = new CartService();
      cart.addItem({ id: '1', name: 'A', price: 100, quantity: 2 });
      cart.addItem({ id: '2', name: 'B', price: 50, quantity: 1 });
      
      expect(cart.total).toBe(250);
    });
  });

  describe('removeItem', () => {
    it('should remove item from cart', () => {
      const cart = new CartService();
      cart.addItem({ id: '1', name: 'A', price: 100, quantity: 1 });
      
      cart.removeItem('1');
      
      expect(cart.items).toHaveLength(0);
    });

    it('should throw error if item not found', () => {
      const cart = new CartService();
      
      expect(() => cart.removeItem('999')).toThrow('Item not found');
    });
  });
});

// 2. GREEN - implement ให้ test ผ่าน
class CartService {
  private _items: CartItem[] = [];

  get items(): CartItem[] {
    return [...this._items];
  }

  get total(): number {
    return this._items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  addItem(newItem: CartItem): void {
    const existing = this._items.find(i => i.id === newItem.id);
    if (existing) {
      existing.quantity += newItem.quantity;
    } else {
      this._items.push({ ...newItem });
    }
  }

  removeItem(id: string): void {
    const index = this._items.findIndex(i => i.id === id);
    if (index === -1) throw new Error('Item not found');
    this._items.splice(index, 1);
  }
}

// 3. REFACTOR - ปรับปรุงโค้ด (tests ยังผ่าน)
```

---

## 3. Component Testing

### __tests__/components/Button.test.tsx

```typescript
import React from 'react';
import {
  render,
  fireEvent,
  waitFor,
  screen,
} from '@testing-library/react-native';
import Button from '../../src/components/Button';

const TestWrapper: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <ThemeProvider>{children}</ThemeProvider>
);

describe('Button component', () => {
  it('renders correctly', () => {
    render(<Button>Click me</Button>, { wrapper: TestWrapper });
    expect(screen.getByText('Click me')).toBeTruthy();
  });

  it('calls onPress when pressed', () => {
    const onPress = jest.fn();
    render(<Button onPress={onPress}>Click me</Button>, { wrapper: TestWrapper });
    
    fireEvent.press(screen.getByText('Click me'));
    
    expect(onPress).toHaveBeenCalledTimes(1);
  });

  it('does not call onPress when disabled', () => {
    const onPress = jest.fn();
    render(
      <Button onPress={onPress} isDisabled>
        Click me
      </Button>,
      { wrapper: TestWrapper }
    );
    
    fireEvent.press(screen.getByText('Click me'));
    
    expect(onPress).not.toHaveBeenCalled();
  });

  it('shows loading indicator when isLoading', () => {
    render(<Button isLoading>Submit</Button>, { wrapper: TestWrapper });
    
    expect(screen.getByTestId('loading-indicator')).toBeTruthy();
    expect(screen.queryByText('Submit')).toBeNull();
  });

  it('matches snapshot', () => {
    const { toJSON } = render(
      <Button variant="filled" colorScheme="primary">Primary</Button>,
      { wrapper: TestWrapper }
    );
    
    expect(toJSON()).toMatchSnapshot();
  });

  describe('variants', () => {
    it.each(['filled', 'outlined', 'ghost'] as const)(
      'renders %s variant correctly',
      (variant) => {
        const { toJSON } = render(
          <Button variant={variant}>{variant}</Button>,
          { wrapper: TestWrapper }
        );
        expect(toJSON()).toMatchSnapshot();
      }
    );
  });
});
```

### __tests__/screens/LoginScreen.test.tsx

```typescript
import React from 'react';
import {
  render,
  fireEvent,
  waitFor,
  screen,
  act,
} from '@testing-library/react-native';
import LoginScreen from '../../src/screens/LoginScreen';
import { server } from '../mocks/server';
import { rest } from 'msw';

describe('LoginScreen', () => {
  const mockNavigation = {
    navigate: jest.fn(),
    replace: jest.fn(),
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders login form', () => {
    render(<LoginScreen navigation={mockNavigation as any} />);
    
    expect(screen.getByPlaceholderText('อีเมล')).toBeTruthy();
    expect(screen.getByPlaceholderText('รหัสผ่าน')).toBeTruthy();
    expect(screen.getByText('เข้าสู่ระบบ')).toBeTruthy();
  });

  it('validates required fields', async () => {
    render(<LoginScreen navigation={mockNavigation as any} />);
    
    fireEvent.press(screen.getByText('เข้าสู่ระบบ'));
    
    await waitFor(() => {
      expect(screen.getByText('กรุณากรอกอีเมล')).toBeTruthy();
      expect(screen.getByText('กรุณากรอกรหัสผ่าน')).toBeTruthy();
    });
  });

  it('validates email format', async () => {
    render(<LoginScreen navigation={mockNavigation as any} />);
    
    fireEvent.changeText(screen.getByPlaceholderText('อีเมล'), 'invalid-email');
    fireEvent.press(screen.getByText('เข้าสู่ระบบ'));
    
    await waitFor(() => {
      expect(screen.getByText('รูปแบบอีเมลไม่ถูกต้อง')).toBeTruthy();
    });
  });

  it('submits form with valid credentials', async () => {
    render(<LoginScreen navigation={mockNavigation as any} />);
    
    fireEvent.changeText(screen.getByPlaceholderText('อีเมล'), 'test@example.com');
    fireEvent.changeText(screen.getByPlaceholderText('รหัสผ่าน'), 'Password123!');
    
    await act(async () => {
      fireEvent.press(screen.getByText('เข้าสู่ระบบ'));
    });
    
    await waitFor(() => {
      expect(mockNavigation.replace).toHaveBeenCalledWith('Main');
    });
  });

  it('shows error on invalid credentials', async () => {
    server.use(
      rest.post('/api/auth/login', (req, res, ctx) => {
        return res(
          ctx.status(401),
          ctx.json({ error: 'Invalid credentials' })
        );
      })
    );
    
    render(<LoginScreen navigation={mockNavigation as any} />);
    
    fireEvent.changeText(screen.getByPlaceholderText('อีเมล'), 'wrong@example.com');
    fireEvent.changeText(screen.getByPlaceholderText('รหัสผ่าน'), 'wrongpassword');
    
    await act(async () => {
      fireEvent.press(screen.getByText('เข้าสู่ระบบ'));
    });
    
    await waitFor(() => {
      expect(screen.getByText('อีเมลหรือรหัสผ่านไม่ถูกต้อง')).toBeTruthy();
    });
  });
});
```

---

## 4. Custom Hooks Testing

### __tests__/hooks/useCart.test.ts

```typescript
import { renderHook, act } from '@testing-library/react-native';
import useCart from '../../src/hooks/useCart';

describe('useCart hook', () => {
  it('initializes with empty cart', () => {
    const { result } = renderHook(() => useCart());
    
    expect(result.current.items).toEqual([]);
    expect(result.current.total).toBe(0);
    expect(result.current.count).toBe(0);
  });

  it('adds item to cart', () => {
    const { result } = renderHook(() => useCart());
    
    act(() => {
      result.current.addItem({
        id: '1',
        name: 'สินค้า A',
        price: 100,
        quantity: 1,
      });
    });
    
    expect(result.current.items).toHaveLength(1);
    expect(result.current.total).toBe(100);
    expect(result.current.count).toBe(1);
  });

  it('updates quantity for existing item', () => {
    const { result } = renderHook(() => useCart());
    
    act(() => {
      result.current.addItem({ id: '1', name: 'A', price: 100, quantity: 1 });
      result.current.addItem({ id: '1', name: 'A', price: 100, quantity: 2 });
    });
    
    expect(result.current.items[0].quantity).toBe(3);
  });

  it('removes item from cart', () => {
    const { result } = renderHook(() => useCart());
    
    act(() => {
      result.current.addItem({ id: '1', name: 'A', price: 100, quantity: 1 });
    });
    
    act(() => {
      result.current.removeItem('1');
    });
    
    expect(result.current.items).toHaveLength(0);
  });

  it('clears cart', () => {
    const { result } = renderHook(() => useCart());
    
    act(() => {
      result.current.addItem({ id: '1', name: 'A', price: 100, quantity: 1 });
      result.current.addItem({ id: '2', name: 'B', price: 200, quantity: 2 });
    });
    
    act(() => {
      result.current.clearCart();
    });
    
    expect(result.current.items).toHaveLength(0);
    expect(result.current.total).toBe(0);
  });

  it('applies discount correctly', () => {
    const { result } = renderHook(() => useCart());
    
    act(() => {
      result.current.addItem({ id: '1', name: 'A', price: 1000, quantity: 1 });
      result.current.applyDiscount({ code: 'SAVE10', type: 'percentage', value: 10 });
    });
    
    expect(result.current.discount).toBe(100);
    expect(result.current.totalAfterDiscount).toBe(900);
  });
});
```

---

## 5. MSW (Mock Service Worker)

### src/mocks/handlers.ts

```typescript
import { rest } from 'msw';

const mockUsers = [
  { id: '1', email: 'test@example.com', displayName: 'Test User', role: 'user' },
];

const mockProducts = [
  { id: '1', name: 'สินค้า A', price: 100, stock: 10 },
  { id: '2', name: 'สินค้า B', price: 200, stock: 5 },
];

export const handlers = [
  // Auth
  rest.post('/api/auth/login', async (req, res, ctx) => {
    const { email, password } = await req.json();
    
    if (email === 'test@example.com' && password === 'Password123!') {
      return res(
        ctx.status(200),
        ctx.json({
          data: {
            user: mockUsers[0],
            accessToken: 'mock-access-token',
            refreshToken: 'mock-refresh-token',
          },
        })
      );
    }
    
    return res(
      ctx.status(401),
      ctx.json({ error: 'Invalid credentials' })
    );
  }),

  // Products
  rest.get('/api/products', (req, res, ctx) => {
    const search = req.url.searchParams.get('search');
    const filtered = search
      ? mockProducts.filter(p => p.name.includes(search))
      : mockProducts;
    
    return res(
      ctx.status(200),
      ctx.json({
        data: filtered,
        pagination: { total: filtered.length, page: 1, perPage: 20, totalPages: 1 },
      })
    );
  }),

  rest.get('/api/products/:id', (req, res, ctx) => {
    const { id } = req.params;
    const product = mockProducts.find(p => p.id === id);
    
    if (!product) {
      return res(ctx.status(404), ctx.json({ error: 'Product not found' }));
    }
    
    return res(ctx.status(200), ctx.json({ data: product }));
  }),

  // Orders
  rest.post('/api/orders', async (req, res, ctx) => {
    const order = await req.json();
    return res(
      ctx.status(201),
      ctx.json({
        data: {
          orderId: 'order-123',
          status: 'pending',
          ...order,
        },
      })
    );
  }),
];
```

---

## 6. E2E Testing กับ Detox

### .detoxrc.js

```javascript
module.exports = {
  testRunner: {
    args: {
      $0: 'jest',
      config: 'e2e/jest.config.js',
    },
    jest: { setupTimeout: 120000 },
  },
  apps: {
    'ios.debug': {
      type: 'ios.app',
      binaryPath: 'ios/build/Build/Products/Debug-iphonesimulator/YourApp.app',
      build: 'xcodebuild -workspace ios/YourApp.xcworkspace -scheme YourApp -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build',
    },
    'android.debug': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/debug/app-debug.apk',
      build: 'cd android && ./gradlew assembleDebug assembleAndroidTest -DtestBuildType=debug',
    },
  },
  devices: {
    simulator: {
      type: 'ios.simulator',
      device: { type: 'iPhone 14' },
    },
    emulator: {
      type: 'android.emulator',
      device: { avdName: 'Pixel_6_API_33' },
    },
  },
  configurations: {
    'ios.sim.debug': {
      device: 'simulator',
      app: 'ios.debug',
    },
    'android.emu.debug': {
      device: 'emulator',
      app: 'android.debug',
    },
  },
};
```

### e2e/tests/login.e2e.ts

```typescript
import { device, element, by, expect, waitFor } from 'detox';

describe('Login Flow', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  beforeEach(async () => {
    await device.reloadReactNative();
  });

  it('should show login screen', async () => {
    await expect(element(by.id('email-input'))).toBeVisible();
    await expect(element(by.id('password-input'))).toBeVisible();
    await expect(element(by.id('login-button'))).toBeVisible();
  });

  it('should login successfully', async () => {
    await element(by.id('email-input')).typeText('test@example.com');
    await element(by.id('password-input')).typeText('Password123!');
    await element(by.id('login-button')).tap();
    
    await waitFor(element(by.id('home-screen')))
      .toBeVisible()
      .withTimeout(10000);
  });

  it('should show error for invalid credentials', async () => {
    await element(by.id('email-input')).typeText('wrong@example.com');
    await element(by.id('password-input')).typeText('wrongpass');
    await element(by.id('login-button')).tap();
    
    await waitFor(element(by.text('อีเมลหรือรหัสผ่านไม่ถูกต้อง')))
      .toBeVisible()
      .withTimeout(5000);
  });

  it('should navigate to register screen', async () => {
    await element(by.id('register-link')).tap();
    
    await waitFor(element(by.id('register-screen')))
      .toBeVisible()
      .withTimeout(3000);
  });
});
```

---

## 7. Performance Testing

```typescript
// __tests__/performance/renderPerformance.test.tsx
import React from 'react';
import { render } from '@testing-library/react-native';

describe('Performance Tests', () => {
  it('renders large list within time limit', async () => {
    const items = Array.from({ length: 1000 }, (_, i) => ({
      id: String(i),
      title: `Item ${i}`,
    }));
    
    const start = performance.now();
    
    render(
      <FlatList
        data={items}
        keyExtractor={item => item.id}
        renderItem={({ item }) => <Text>{item.title}</Text>}
        initialNumToRender={20}
      />
    );
    
    const end = performance.now();
    const renderTime = end - start;
    
    expect(renderTime).toBeLessThan(500); // ต้องใช้เวลาน้อยกว่า 500ms
  });

  it('component re-renders minimal times', () => {
    const renderCount = { current: 0 };
    
    const TestComponent = React.memo(() => {
      renderCount.current++;
      return <Text>Test</Text>;
    });
    
    const { rerender } = render(<TestComponent />);
    rerender(<TestComponent />);
    rerender(<TestComponent />);
    
    expect(renderCount.current).toBe(1); // React.memo ป้องกัน re-render
  });
});
```

---

## Workshop Exercises

1. **100% Coverage** - เขียน tests ให้ได้ coverage 100% สำหรับ utility functions
2. **Integration Tests** - test flow ที่ครอบคลุม multiple components
3. **Snapshot Testing** - setup snapshot tests สำหรับทุก screen
4. **CI Integration** - เพิ่ม tests ใน GitHub Actions

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **TDD** - Red-Green-Refactor cycle
2. **Component Testing** - test React Native components
3. **Hook Testing** - test custom hooks
4. **MSW** - mock API responses
5. **E2E Testing** - Detox สำหรับ end-to-end tests
6. **Performance** - ตรวจสอบ render performance
