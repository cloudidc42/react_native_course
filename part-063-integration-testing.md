# Part 063: Integration Testing ใน React Native

## บทนำ

Integration Testing คือการทดสอบการทำงานร่วมกันของหลาย components หรือ modules เพื่อให้แน่ใจว่าระบบทำงานได้อย่างถูกต้องเมื่อนำมารวมกัน ต่างจาก Unit Testing ที่ทดสอบแต่ละส่วนแยกกัน

---

## 1. React Native Testing Library

### ทำไมต้องใช้ React Native Testing Library?

```
- ทดสอบในมุมมองของผู้ใช้ ไม่ใช่ implementation
- API คล้ายกับ React Testing Library สำหรับ web
- รองรับ accessibility queries
- ลด brittle tests
```

### ติดตั้ง

```bash
npm install --save-dev @testing-library/react-native
npm install --save-dev @testing-library/jest-native
npm install --save-dev @testing-library/react-hooks
```

### setup ใน jest.setup.ts

```typescript
import '@testing-library/jest-native/extend-expect';
```

---

## 2. Queries และวิธีค้นหา Elements

### Query Types

```typescript
// getBy - ส่งกลับ element หรือ throw ถ้าไม่พบ/พบหลาย
const button = screen.getByRole('button', { name: 'Submit' });
const title = screen.getByText('Welcome');
const input = screen.getByTestId('email-input');
const label = screen.getByLabelText('Email');

// queryBy - ส่งกลับ null ถ้าไม่พบ (ไม่ throw)
const maybeError = screen.queryByText('Error message');

// findBy - async, คืน Promise
const asyncElement = await screen.findByText('Loaded data');

// getAllBy - ส่งกลับ array
const allButtons = screen.getAllByRole('button');
```

### Priority ของ Queries (ตามลำดับที่แนะนำ)

```
1. getByRole          - ดีที่สุด (accessibility)
2. getByLabelText     - สำหรับ form fields
3. getByPlaceholderText
4. getByText          - สำหรับ content
5. getByDisplayValue
6. getByAltText
7. getByTitle
8. getByTestId        - สุดท้าย (เมื่อไม่มีทางเลือกอื่น)
```

---

## 3. Testing Navigation

### Setup Navigation สำหรับ Tests

```typescript
// testUtils/navigationWrapper.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';
import { render } from '@testing-library/react-native';

const Stack = createStackNavigator();

// Wrapper component สำหรับ tests
export const createNavigationWrapper = (initialRoute: string = 'Home') => {
  const Wrapper: React.FC<{ children: React.ReactNode }> = ({ children }) => (
    <NavigationContainer>
      <Stack.Navigator initialRouteName={initialRoute}>
        <Stack.Screen name="Home" component={() => <>{children}</>} />
        <Stack.Screen name="Details" component={() => <></>} />
        <Stack.Screen name="Profile" component={() => <></>} />
        <Stack.Screen name="Settings" component={() => <></>} />
      </Stack.Navigator>
    </NavigationContainer>
  );
  return Wrapper;
};

// Custom render function
export const renderWithNavigation = (
  ui: React.ReactElement,
  options?: { initialRoute?: string }
) => {
  const Wrapper = createNavigationWrapper(options?.initialRoute);
  return render(ui, { wrapper: Wrapper });
};
```

### ทดสอบ Navigation

**screens/HomeScreen.tsx**:
```typescript
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useNavigation } from '@react-navigation/native';

const HomeScreen: React.FC = () => {
  const navigation = useNavigation<any>();

  return (
    <View style={styles.container}>
      <Text style={styles.title} testID="home-title">
        Welcome Home
      </Text>
      <TouchableOpacity
        testID="go-to-details"
        style={styles.button}
        onPress={() => navigation.navigate('Details', { id: '123' })}
      >
        <Text>View Details</Text>
      </TouchableOpacity>
      <TouchableOpacity
        testID="go-to-profile"
        style={styles.button}
        onPress={() => navigation.navigate('Profile')}
      >
        <Text>My Profile</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20 },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 20 },
  button: {
    padding: 16,
    backgroundColor: '#2196F3',
    borderRadius: 8,
    marginBottom: 10,
    alignItems: 'center',
  },
});

export default HomeScreen;
```

**screens/__tests__/HomeScreen.test.tsx**:
```typescript
import React from 'react';
import { fireEvent, screen } from '@testing-library/react-native';
import { renderWithNavigation } from '../../testUtils/navigationWrapper';
import HomeScreen from '../HomeScreen';

// Mock navigation
const mockNavigate = jest.fn();
jest.mock('@react-navigation/native', () => ({
  ...jest.requireActual('@react-navigation/native'),
  useNavigation: () => ({
    navigate: mockNavigate,
    goBack: jest.fn(),
  }),
}));

describe('HomeScreen', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders welcome title', () => {
    renderWithNavigation(<HomeScreen />);
    expect(screen.getByTestId('home-title')).toBeTruthy();
    expect(screen.getByText('Welcome Home')).toBeTruthy();
  });

  it('navigates to Details screen when button pressed', () => {
    renderWithNavigation(<HomeScreen />);
    
    fireEvent.press(screen.getByTestId('go-to-details'));
    
    expect(mockNavigate).toHaveBeenCalledWith('Details', { id: '123' });
  });

  it('navigates to Profile screen when button pressed', () => {
    renderWithNavigation(<HomeScreen />);
    
    fireEvent.press(screen.getByTestId('go-to-profile'));
    
    expect(mockNavigate).toHaveBeenCalledWith('Profile');
  });
});
```

---

## 4. Testing Forms

### Complex Form ตัวอย่าง

**screens/RegisterScreen.tsx**:
```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ScrollView,
  StyleSheet,
  ActivityIndicator,
} from 'react-native';

interface RegisterData {
  firstName: string;
  lastName: string;
  email: string;
  password: string;
  confirmPassword: string;
  agreeToTerms: boolean;
}

interface RegisterFormErrors {
  firstName?: string;
  lastName?: string;
  email?: string;
  password?: string;
  confirmPassword?: string;
  agreeToTerms?: string;
  general?: string;
}

interface RegisterScreenProps {
  onRegister: (data: Omit<RegisterData, 'confirmPassword'>) => Promise<void>;
  onLoginPress: () => void;
}

const RegisterScreen: React.FC<RegisterScreenProps> = ({
  onRegister,
  onLoginPress,
}) => {
  const [formData, setFormData] = useState<RegisterData>({
    firstName: '',
    lastName: '',
    email: '',
    password: '',
    confirmPassword: '',
    agreeToTerms: false,
  });
  const [errors, setErrors] = useState<RegisterFormErrors>({});
  const [loading, setLoading] = useState(false);
  const [success, setSuccess] = useState(false);

  const updateField = (field: keyof RegisterData, value: string | boolean) => {
    setFormData(prev => ({ ...prev, [field]: value }));
    // Clear error when user starts typing
    if (errors[field as keyof RegisterFormErrors]) {
      setErrors(prev => ({ ...prev, [field]: undefined }));
    }
  };

  const validate = (): boolean => {
    const newErrors: RegisterFormErrors = {};
    
    if (!formData.firstName.trim()) {
      newErrors.firstName = 'First name is required';
    }
    
    if (!formData.lastName.trim()) {
      newErrors.lastName = 'Last name is required';
    }
    
    if (!formData.email) {
      newErrors.email = 'Email is required';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(formData.email)) {
      newErrors.email = 'Invalid email format';
    }
    
    if (!formData.password) {
      newErrors.password = 'Password is required';
    } else if (formData.password.length < 8) {
      newErrors.password = 'Password must be at least 8 characters';
    }
    
    if (formData.password !== formData.confirmPassword) {
      newErrors.confirmPassword = 'Passwords do not match';
    }
    
    if (!formData.agreeToTerms) {
      newErrors.agreeToTerms = 'You must agree to terms';
    }
    
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async () => {
    if (!validate()) return;
    
    try {
      setLoading(true);
      const { confirmPassword, ...registerData } = formData;
      await onRegister(registerData);
      setSuccess(true);
    } catch (error: any) {
      setErrors({ general: error.message || 'Registration failed' });
    } finally {
      setLoading(false);
    }
  };

  if (success) {
    return (
      <View style={styles.successContainer} testID="success-screen">
        <Text style={styles.successTitle}>Account Created!</Text>
        <Text style={styles.successText}>
          Please check your email to verify your account.
        </Text>
      </View>
    );
  }

  return (
    <ScrollView style={styles.container} testID="register-form">
      <Text style={styles.title}>Create Account</Text>
      
      {errors.general && (
        <View style={styles.errorBanner} testID="general-error">
          <Text style={styles.errorBannerText}>{errors.general}</Text>
        </View>
      )}

      <View style={styles.row}>
        <View style={styles.halfInput}>
          <TextInput
            testID="first-name-input"
            style={[styles.input, errors.firstName && styles.inputError]}
            placeholder="First Name"
            value={formData.firstName}
            onChangeText={v => updateField('firstName', v)}
          />
          {errors.firstName && (
            <Text testID="first-name-error" style={styles.errorText}>
              {errors.firstName}
            </Text>
          )}
        </View>
        <View style={styles.halfInput}>
          <TextInput
            testID="last-name-input"
            style={[styles.input, errors.lastName && styles.inputError]}
            placeholder="Last Name"
            value={formData.lastName}
            onChangeText={v => updateField('lastName', v)}
          />
          {errors.lastName && (
            <Text testID="last-name-error" style={styles.errorText}>
              {errors.lastName}
            </Text>
          )}
        </View>
      </View>

      <TextInput
        testID="email-input"
        style={[styles.input, errors.email && styles.inputError]}
        placeholder="Email"
        value={formData.email}
        onChangeText={v => updateField('email', v)}
        keyboardType="email-address"
        autoCapitalize="none"
      />
      {errors.email && (
        <Text testID="email-error" style={styles.errorText}>
          {errors.email}
        </Text>
      )}

      <TextInput
        testID="password-input"
        style={[styles.input, errors.password && styles.inputError]}
        placeholder="Password"
        value={formData.password}
        onChangeText={v => updateField('password', v)}
        secureTextEntry
      />
      {errors.password && (
        <Text testID="password-error" style={styles.errorText}>
          {errors.password}
        </Text>
      )}

      <TextInput
        testID="confirm-password-input"
        style={[styles.input, errors.confirmPassword && styles.inputError]}
        placeholder="Confirm Password"
        value={formData.confirmPassword}
        onChangeText={v => updateField('confirmPassword', v)}
        secureTextEntry
      />
      {errors.confirmPassword && (
        <Text testID="confirm-password-error" style={styles.errorText}>
          {errors.confirmPassword}
        </Text>
      )}

      <TouchableOpacity
        testID="terms-checkbox"
        style={styles.checkboxContainer}
        onPress={() => updateField('agreeToTerms', !formData.agreeToTerms)}
      >
        <View style={[styles.checkbox, formData.agreeToTerms && styles.checked]}>
          {formData.agreeToTerms && <Text style={styles.checkmark}>✓</Text>}
        </View>
        <Text style={styles.checkboxLabel}>I agree to Terms & Conditions</Text>
      </TouchableOpacity>
      {errors.agreeToTerms && (
        <Text testID="terms-error" style={styles.errorText}>
          {errors.agreeToTerms}
        </Text>
      )}

      <TouchableOpacity
        testID="submit-button"
        style={[styles.submitButton, loading && styles.disabledButton]}
        onPress={handleSubmit}
        disabled={loading}
      >
        {loading ? (
          <ActivityIndicator color="#fff" testID="loading-indicator" />
        ) : (
          <Text style={styles.submitText}>Create Account</Text>
        )}
      </TouchableOpacity>

      <TouchableOpacity testID="login-link" onPress={onLoginPress}>
        <Text style={styles.loginLink}>Already have an account? Sign in</Text>
      </TouchableOpacity>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20 },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 24 },
  row: { flexDirection: 'row', gap: 12 },
  halfInput: { flex: 1 },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    marginBottom: 4,
    fontSize: 16,
  },
  inputError: { borderColor: '#f44336' },
  errorText: { color: '#f44336', fontSize: 12, marginBottom: 8 },
  errorBanner: {
    backgroundColor: '#ffebee',
    borderRadius: 8,
    padding: 12,
    marginBottom: 16,
  },
  errorBannerText: { color: '#c62828' },
  checkboxContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 8,
  },
  checkbox: {
    width: 20,
    height: 20,
    borderWidth: 2,
    borderColor: '#ddd',
    borderRadius: 4,
    marginRight: 8,
    alignItems: 'center',
    justifyContent: 'center',
  },
  checked: { backgroundColor: '#2196F3', borderColor: '#2196F3' },
  checkmark: { color: '#fff', fontSize: 12 },
  checkboxLabel: { fontSize: 14 },
  submitButton: {
    backgroundColor: '#2196F3',
    borderRadius: 8,
    padding: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  disabledButton: { opacity: 0.7 },
  submitText: { color: '#fff', fontSize: 16, fontWeight: 'bold' },
  loginLink: { color: '#2196F3', textAlign: 'center', marginTop: 16 },
  successContainer: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 20 },
  successTitle: { fontSize: 24, fontWeight: 'bold', color: '#4CAF50', marginBottom: 12 },
  successText: { fontSize: 16, textAlign: 'center', color: '#666' },
});

export default RegisterScreen;
```

**screens/__tests__/RegisterScreen.test.tsx**:
```typescript
import React from 'react';
import { render, fireEvent, waitFor, screen } from '@testing-library/react-native';
import RegisterScreen from '../RegisterScreen';

describe('RegisterScreen', () => {
  const mockOnRegister = jest.fn();
  const mockOnLoginPress = jest.fn();

  const defaultProps = {
    onRegister: mockOnRegister,
    onLoginPress: mockOnLoginPress,
  };

  const fillValidForm = () => {
    fireEvent.changeText(screen.getByTestId('first-name-input'), 'John');
    fireEvent.changeText(screen.getByTestId('last-name-input'), 'Doe');
    fireEvent.changeText(screen.getByTestId('email-input'), 'john@example.com');
    fireEvent.changeText(screen.getByTestId('password-input'), 'Password123!');
    fireEvent.changeText(screen.getByTestId('confirm-password-input'), 'Password123!');
    fireEvent.press(screen.getByTestId('terms-checkbox'));
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('Rendering', () => {
    it('renders all form fields', () => {
      render(<RegisterScreen {...defaultProps} />);
      
      expect(screen.getByTestId('first-name-input')).toBeTruthy();
      expect(screen.getByTestId('last-name-input')).toBeTruthy();
      expect(screen.getByTestId('email-input')).toBeTruthy();
      expect(screen.getByTestId('password-input')).toBeTruthy();
      expect(screen.getByTestId('confirm-password-input')).toBeTruthy();
      expect(screen.getByTestId('terms-checkbox')).toBeTruthy();
      expect(screen.getByTestId('submit-button')).toBeTruthy();
    });
  });

  describe('Form Validation', () => {
    it('shows all required field errors when form is empty', async () => {
      render(<RegisterScreen {...defaultProps} />);
      
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('first-name-error')).toBeTruthy();
        expect(screen.getByTestId('last-name-error')).toBeTruthy();
        expect(screen.getByTestId('email-error')).toBeTruthy();
        expect(screen.getByTestId('password-error')).toBeTruthy();
        expect(screen.getByTestId('terms-error')).toBeTruthy();
      });
    });

    it('shows password mismatch error', async () => {
      render(<RegisterScreen {...defaultProps} />);
      
      fireEvent.changeText(screen.getByTestId('password-input'), 'Password123!');
      fireEvent.changeText(screen.getByTestId('confirm-password-input'), 'DifferentPassword!');
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('confirm-password-error')).toBeTruthy();
        expect(screen.getByText('Passwords do not match')).toBeTruthy();
      });
    });

    it('clears error when user starts typing', async () => {
      render(<RegisterScreen {...defaultProps} />);
      
      // Trigger validation errors
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('email-error')).toBeTruthy();
      });
      
      // Start typing - error should clear
      fireEvent.changeText(screen.getByTestId('email-input'), 'j');
      
      await waitFor(() => {
        expect(screen.queryByTestId('email-error')).toBeNull();
      });
    });
  });

  describe('Successful Registration', () => {
    it('calls onRegister with correct data', async () => {
      mockOnRegister.mockResolvedValueOnce(undefined);
      render(<RegisterScreen {...defaultProps} />);
      
      fillValidForm();
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(mockOnRegister).toHaveBeenCalledWith({
          firstName: 'John',
          lastName: 'Doe',
          email: 'john@example.com',
          password: 'Password123!',
          agreeToTerms: true,
        });
      });
    });

    it('shows success screen after registration', async () => {
      mockOnRegister.mockResolvedValueOnce(undefined);
      render(<RegisterScreen {...defaultProps} />);
      
      fillValidForm();
      fireEvent.press(screen.getByTestId('submit-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('success-screen')).toBeTruthy();
        expect(screen.getByText('Account Created!')).toBeTruthy();
      });
    });
  });

  describe('Login Navigation', () => {
    it('calls onLoginPress when login link is pressed', () => {
      render(<RegisterScreen {...defaultProps} />);
      
      fireEvent.press(screen.getByTestId('login-link'));
      
      expect(mockOnLoginPress).toHaveBeenCalledTimes(1);
    });
  });
});
```

---

## 5. Testing API Calls ใน Components

### Setup MSW (Mock Service Worker) สำหรับ React Native

```typescript
// testUtils/mswSetup.ts
import { setupServer } from 'msw/node';
import { rest } from 'msw';

export const server = setupServer();

// jest.setup.ts
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### ทดสอบ Component ที่ดึงข้อมูลจาก API

**screens/ProductListScreen.tsx**:
```typescript
import React, { useEffect, useState } from 'react';
import {
  View,
  Text,
  FlatList,
  ActivityIndicator,
  TouchableOpacity,
  StyleSheet,
  RefreshControl,
} from 'react-native';
import axios from 'axios';

interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
}

const ProductListScreen: React.FC = () => {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [refreshing, setRefreshing] = useState(false);

  const fetchProducts = async () => {
    try {
      setError(null);
      const response = await axios.get('/api/products');
      setProducts(response.data);
    } catch (err: any) {
      setError(err.message || 'Failed to load products');
    } finally {
      setLoading(false);
      setRefreshing(false);
    }
  };

  useEffect(() => {
    fetchProducts();
  }, []);

  const handleRefresh = () => {
    setRefreshing(true);
    fetchProducts();
  };

  if (loading) {
    return (
      <View style={styles.centered} testID="loading-screen">
        <ActivityIndicator size="large" />
        <Text>Loading products...</Text>
      </View>
    );
  }

  if (error) {
    return (
      <View style={styles.centered} testID="error-screen">
        <Text style={styles.errorText}>{error}</Text>
        <TouchableOpacity testID="retry-button" onPress={fetchProducts}>
          <Text style={styles.retryText}>Retry</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <FlatList
      testID="product-list"
      data={products}
      keyExtractor={item => item.id.toString()}
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
      }
      renderItem={({ item }) => (
        <View testID={`product-${item.id}`} style={styles.productCard}>
          <Text style={styles.productName}>{item.name}</Text>
          <Text style={styles.productPrice}>${item.price}</Text>
          <Text style={styles.productDesc}>{item.description}</Text>
        </View>
      )}
      ListEmptyComponent={
        <View testID="empty-list" style={styles.centered}>
          <Text>No products available</Text>
        </View>
      }
    />
  );
};

const styles = StyleSheet.create({
  centered: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 20 },
  errorText: { color: '#f44336', fontSize: 16, marginBottom: 16 },
  retryText: { color: '#2196F3', fontSize: 16 },
  productCard: { padding: 16, backgroundColor: '#fff', marginBottom: 8 },
  productName: { fontSize: 18, fontWeight: 'bold' },
  productPrice: { fontSize: 16, color: '#4CAF50', marginVertical: 4 },
  productDesc: { fontSize: 14, color: '#666' },
});

export default ProductListScreen;
```

**screens/__tests__/ProductListScreen.test.tsx**:
```typescript
import React from 'react';
import { render, screen, waitFor, fireEvent } from '@testing-library/react-native';
import axios from 'axios';
import ProductListScreen from '../ProductListScreen';

jest.mock('axios');
const mockedAxios = axios as jest.Mocked<typeof axios>;

const mockProducts = [
  { id: 1, name: 'Product 1', price: 29.99, description: 'Description 1' },
  { id: 2, name: 'Product 2', price: 49.99, description: 'Description 2' },
  { id: 3, name: 'Product 3', price: 9.99, description: 'Description 3' },
];

describe('ProductListScreen', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('Loading State', () => {
    it('shows loading indicator initially', () => {
      mockedAxios.get.mockImplementation(
        () => new Promise(() => {}) // Never resolves
      );
      
      render(<ProductListScreen />);
      
      expect(screen.getByTestId('loading-screen')).toBeTruthy();
      expect(screen.getByText('Loading products...')).toBeTruthy();
    });

    it('hides loading indicator after data loads', async () => {
      mockedAxios.get.mockResolvedValueOnce({ data: mockProducts });
      
      render(<ProductListScreen />);
      
      await waitFor(() => {
        expect(screen.queryByTestId('loading-screen')).toBeNull();
      });
    });
  });

  describe('Success State', () => {
    it('renders product list after successful fetch', async () => {
      mockedAxios.get.mockResolvedValueOnce({ data: mockProducts });
      
      render(<ProductListScreen />);
      
      await waitFor(() => {
        expect(screen.getByTestId('product-list')).toBeTruthy();
        expect(screen.getByText('Product 1')).toBeTruthy();
        expect(screen.getByText('Product 2')).toBeTruthy();
        expect(screen.getByText('Product 3')).toBeTruthy();
      });
    });

    it('shows product prices correctly', async () => {
      mockedAxios.get.mockResolvedValueOnce({ data: mockProducts });
      
      render(<ProductListScreen />);
      
      await waitFor(() => {
        expect(screen.getByText('$29.99')).toBeTruthy();
        expect(screen.getByText('$49.99')).toBeTruthy();
      });
    });

    it('shows empty state when no products', async () => {
      mockedAxios.get.mockResolvedValueOnce({ data: [] });
      
      render(<ProductListScreen />);
      
      await waitFor(() => {
        expect(screen.getByTestId('empty-list')).toBeTruthy();
        expect(screen.getByText('No products available')).toBeTruthy();
      });
    });
  });

  describe('Error State', () => {
    it('shows error message when fetch fails', async () => {
      mockedAxios.get.mockRejectedValueOnce(new Error('Network Error'));
      
      render(<ProductListScreen />);
      
      await waitFor(() => {
        expect(screen.getByTestId('error-screen')).toBeTruthy();
        expect(screen.getByText('Network Error')).toBeTruthy();
      });
    });

    it('retries fetch when retry button pressed', async () => {
      mockedAxios.get
        .mockRejectedValueOnce(new Error('Network Error'))
        .mockResolvedValueOnce({ data: mockProducts });
      
      render(<ProductListScreen />);
      
      await waitFor(() => {
        expect(screen.getByTestId('retry-button')).toBeTruthy();
      });
      
      fireEvent.press(screen.getByTestId('retry-button'));
      
      await waitFor(() => {
        expect(screen.getByTestId('product-list')).toBeTruthy();
        expect(mockedAxios.get).toHaveBeenCalledTimes(2);
      });
    });
  });
});
```

---

## 6. Testing Redux/Context Integration

### Context Testing

```typescript
// contexts/CartContext.tsx
import React, { createContext, useContext, useReducer } from 'react';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  total: number;
}

type CartAction =
  | { type: 'ADD_ITEM'; payload: CartItem }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'CLEAR_CART' };

const cartReducer = (state: CartState, action: CartAction): CartState => {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existingItem = state.items.find(i => i.id === action.payload.id);
      let newItems: CartItem[];
      
      if (existingItem) {
        newItems = state.items.map(i =>
          i.id === action.payload.id
            ? { ...i, quantity: i.quantity + 1 }
            : i
        );
      } else {
        newItems = [...state.items, action.payload];
      }
      
      const total = newItems.reduce((sum, i) => sum + i.price * i.quantity, 0);
      return { items: newItems, total };
    }
    case 'REMOVE_ITEM': {
      const newItems = state.items.filter(i => i.id !== action.payload);
      const total = newItems.reduce((sum, i) => sum + i.price * i.quantity, 0);
      return { items: newItems, total };
    }
    case 'CLEAR_CART':
      return { items: [], total: 0 };
    default:
      return state;
  }
};

const CartContext = createContext<{
  state: CartState;
  dispatch: React.Dispatch<CartAction>;
} | null>(null);

export const CartProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  return (
    <CartContext.Provider value={{ state, dispatch }}>
      {children}
    </CartContext.Provider>
  );
};

export const useCart = () => {
  const context = useContext(CartContext);
  if (!context) throw new Error('useCart must be used within CartProvider');
  return context;
};
```

```typescript
// contexts/__tests__/CartContext.test.tsx
import React from 'react';
import { render, fireEvent, screen } from '@testing-library/react-native';
import { View, Text, TouchableOpacity } from 'react-native';
import { CartProvider, useCart } from '../CartContext';

const TestCartComponent: React.FC = () => {
  const { state, dispatch } = useCart();
  
  return (
    <View>
      <Text testID="cart-total">Total: ${state.total.toFixed(2)}</Text>
      <Text testID="cart-count">{state.items.length} items</Text>
      
      <TouchableOpacity
        testID="add-item"
        onPress={() => dispatch({
          type: 'ADD_ITEM',
          payload: { id: '1', name: 'Item 1', price: 10, quantity: 1 },
        })}
      >
        <Text>Add Item 1</Text>
      </TouchableOpacity>
      
      <TouchableOpacity
        testID="remove-item"
        onPress={() => dispatch({ type: 'REMOVE_ITEM', payload: '1' })}
      >
        <Text>Remove Item 1</Text>
      </TouchableOpacity>
      
      <TouchableOpacity
        testID="clear-cart"
        onPress={() => dispatch({ type: 'CLEAR_CART' })}
      >
        <Text>Clear Cart</Text>
      </TouchableOpacity>
    </View>
  );
};

const renderWithCart = () => {
  return render(
    <CartProvider>
      <TestCartComponent />
    </CartProvider>
  );
};

describe('CartContext', () => {
  it('starts with empty cart', () => {
    renderWithCart();
    expect(screen.getByTestId('cart-total')).toHaveTextContent('Total: $0.00');
    expect(screen.getByTestId('cart-count')).toHaveTextContent('0 items');
  });

  it('adds item to cart', () => {
    renderWithCart();
    fireEvent.press(screen.getByTestId('add-item'));
    
    expect(screen.getByTestId('cart-count')).toHaveTextContent('1 items');
    expect(screen.getByTestId('cart-total')).toHaveTextContent('Total: $10.00');
  });

  it('removes item from cart', () => {
    renderWithCart();
    fireEvent.press(screen.getByTestId('add-item'));
    fireEvent.press(screen.getByTestId('remove-item'));
    
    expect(screen.getByTestId('cart-count')).toHaveTextContent('0 items');
    expect(screen.getByTestId('cart-total')).toHaveTextContent('Total: $0.00');
  });

  it('clears all items', () => {
    renderWithCart();
    fireEvent.press(screen.getByTestId('add-item'));
    fireEvent.press(screen.getByTestId('add-item'));
    fireEvent.press(screen.getByTestId('clear-cart'));
    
    expect(screen.getByTestId('cart-count')).toHaveTextContent('0 items');
  });
});
```

---

## Workshop: Integration Test Suite

### Test Helper Utilities

```typescript
// testUtils/renderWithProviders.tsx
import React from 'react';
import { render, RenderOptions } from '@testing-library/react-native';
import { NavigationContainer } from '@react-navigation/native';
import { Provider } from 'react-redux';
import { store } from '../src/store';
import { CartProvider } from '../src/contexts/CartContext';

interface CustomRenderOptions extends RenderOptions {
  navigationOptions?: object;
}

const AllProviders: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  return (
    <Provider store={store}>
      <CartProvider>
        <NavigationContainer>
          {children}
        </NavigationContainer>
      </CartProvider>
    </Provider>
  );
};

export const renderWithProviders = (
  ui: React.ReactElement,
  options?: CustomRenderOptions
) => {
  return render(ui, { wrapper: AllProviders, ...options });
};

export * from '@testing-library/react-native';
export { renderWithProviders as render };
```

---

## Tips และ Best Practices

### 1. Test ใน User Flow

```typescript
// ทดสอบเส้นทางที่ผู้ใช้ทำงานจริง
it('user can complete checkout flow', async () => {
  render(<CheckoutFlow />);
  
  // Step 1: Add item
  fireEvent.press(screen.getByTestId('add-to-cart-1'));
  
  // Step 2: Go to cart
  fireEvent.press(screen.getByTestId('cart-button'));
  
  // Step 3: Checkout
  fireEvent.press(screen.getByTestId('checkout-button'));
  
  // Step 4: Fill payment
  fireEvent.changeText(screen.getByTestId('card-number'), '4242424242424242');
  
  await waitFor(() => {
    expect(screen.getByText('Order Confirmed!')).toBeTruthy();
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Testing Library Queries**: วิธีค้นหา elements ต่างๆ
2. **Testing Navigation**: ทดสอบการ navigate ระหว่าง screens
3. **Testing Forms**: ทดสอบ validation และ submission
4. **Testing API Calls**: Mock HTTP requests ใน tests
5. **Testing Context/Redux**: ทดสอบ state management
6. **Integration Test Suite**: การรวมทุกอย่างเข้าด้วยกัน
