# Part 036: Authentication - JWT

## JWT คืออะไร?

JWT (JSON Web Token) เป็นมาตรฐานสำหรับส่งข้อมูลระหว่าง parties อย่างปลอดภัย

### โครงสร้าง JWT

JWT ประกอบด้วย 3 ส่วน แยกด้วย `.`:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9  ← Header (Base64)
.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4ifQ  ← Payload (Base64)
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

**Header**: algorithm และ token type
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload**: ข้อมูล (claims)
```json
{
  "sub": "user123",
  "name": "สมชาย",
  "email": "somchai@example.com",
  "role": "user",
  "iat": 1640000000,
  "exp": 1640003600
}
```

**Signature**: ป้องกันการแก้ไข

### Access Token vs Refresh Token

- **Access Token**: ใช้สำหรับ authenticate requests อายุสั้น (15 นาที - 1 ชั่วโมง)
- **Refresh Token**: ใช้ขอ access token ใหม่ อายุยาว (7-30 วัน)

---

## การติดตั้ง

```bash
# Secure Storage สำหรับเก็บ tokens
npm install expo-secure-store

# JWT decode
npm install jwt-decode

# หรือถ้าใช้ React Native bare
npm install react-native-keychain
```

---

## Token Storage (Secure Storage)

### ใช้ Expo SecureStore

```typescript
// src/utils/secureStorage.ts
import * as SecureStore from 'expo-secure-store';

const KEYS = {
  ACCESS_TOKEN: 'access_token',
  REFRESH_TOKEN: 'refresh_token',
  USER_DATA: 'user_data',
} as const;

export const secureStorage = {
  async saveTokens(accessToken: string, refreshToken: string): Promise<void> {
    await Promise.all([
      SecureStore.setItemAsync(KEYS.ACCESS_TOKEN, accessToken),
      SecureStore.setItemAsync(KEYS.REFRESH_TOKEN, refreshToken),
    ]);
  },

  async getAccessToken(): Promise<string | null> {
    return SecureStore.getItemAsync(KEYS.ACCESS_TOKEN);
  },

  async getRefreshToken(): Promise<string | null> {
    return SecureStore.getItemAsync(KEYS.REFRESH_TOKEN);
  },

  async saveUser(user: object): Promise<void> {
    await SecureStore.setItemAsync(KEYS.USER_DATA, JSON.stringify(user));
  },

  async getUser<T>(): Promise<T | null> {
    const data = await SecureStore.getItemAsync(KEYS.USER_DATA);
    return data ? JSON.parse(data) : null;
  },

  async clearAll(): Promise<void> {
    await Promise.all([
      SecureStore.deleteItemAsync(KEYS.ACCESS_TOKEN),
      SecureStore.deleteItemAsync(KEYS.REFRESH_TOKEN),
      SecureStore.deleteItemAsync(KEYS.USER_DATA),
    ]);
  },
};
```

---

## Login/Logout Flow

### Auth Context

```typescript
// src/contexts/AuthContext.tsx
import React, {
  createContext,
  useContext,
  useState,
  useEffect,
  useCallback,
} from 'react';
import { jwtDecode } from 'jwt-decode';
import { secureStorage } from '../utils/secureStorage';
import { authApi } from '../services/api/authApi';
import { apiClient } from '../services/axiosInstance';

interface User {
  id: string;
  name: string;
  email: string;
  role: string;
  avatar?: string;
}

interface AuthContextType {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  register: (name: string, email: string, password: string) => Promise<void>;
  updateUser: (updates: Partial<User>) => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // ตรวจสอบ token เมื่อ app เริ่ม
  useEffect(() => {
    initializeAuth();
  }, []);

  const initializeAuth = async () => {
    try {
      const accessToken = await secureStorage.getAccessToken();

      if (accessToken && !isTokenExpired(accessToken)) {
        // ดึง user จาก token
        const decoded = jwtDecode<User & { exp: number }>(accessToken);
        setUser({
          id: decoded.id,
          name: decoded.name,
          email: decoded.email,
          role: decoded.role,
          avatar: decoded.avatar,
        });

        // Set token ใน API client
        apiClient.defaults.headers.common['Authorization'] =
          `Bearer ${accessToken}`;
      } else if (accessToken && isTokenExpired(accessToken)) {
        // Token หมดอายุ ลอง refresh
        await refreshTokens();
      }
    } catch (error) {
      console.error('Auth initialization error:', error);
      await secureStorage.clearAll();
    } finally {
      setIsLoading(false);
    }
  };

  const isTokenExpired = (token: string): boolean => {
    try {
      const decoded = jwtDecode<{ exp: number }>(token);
      const currentTime = Date.now() / 1000;
      return decoded.exp < currentTime;
    } catch {
      return true;
    }
  };

  const refreshTokens = async (): Promise<void> => {
    const refreshToken = await secureStorage.getRefreshToken();
    if (!refreshToken) throw new Error('No refresh token');

    const response = await authApi.refreshToken(refreshToken);
    await secureStorage.saveTokens(
      response.accessToken,
      response.refreshToken
    );
    apiClient.defaults.headers.common['Authorization'] =
      `Bearer ${response.accessToken}`;

    const decoded = jwtDecode<User>(response.accessToken);
    setUser(decoded);
  };

  const login = async (email: string, password: string) => {
    const response = await authApi.login({ email, password });

    await secureStorage.saveTokens(
      response.accessToken,
      response.refreshToken
    );

    apiClient.defaults.headers.common['Authorization'] =
      `Bearer ${response.accessToken}`;

    setUser(response.user);
  };

  const logout = useCallback(async () => {
    try {
      await authApi.logout();
    } catch {
      // Ignore logout API errors
    } finally {
      await secureStorage.clearAll();
      delete apiClient.defaults.headers.common['Authorization'];
      setUser(null);
    }
  }, []);

  const register = async (
    name: string,
    email: string,
    password: string
  ) => {
    const response = await authApi.register({ name, email, password });

    await secureStorage.saveTokens(
      response.accessToken,
      response.refreshToken
    );

    apiClient.defaults.headers.common['Authorization'] =
      `Bearer ${response.accessToken}`;

    setUser(response.user);
  };

  const updateUser = (updates: Partial<User>) => {
    setUser((prev) => (prev ? { ...prev, ...updates } : null));
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        isLoading,
        login,
        logout,
        register,
        updateUser,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

---

## Protected Routes

```typescript
// src/navigation/AppNavigator.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { ActivityIndicator, View } from 'react-native';
import { useAuth } from '../contexts/AuthContext';

const Stack = createNativeStackNavigator();

// Auth Navigator
function AuthNavigator() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="Login" component={LoginScreen} />
      <Stack.Screen name="Register" component={RegisterScreen} />
      <Stack.Screen name="ForgotPassword" component={ForgotPasswordScreen} />
    </Stack.Navigator>
  );
}

// App Navigator (Protected)
function MainNavigator() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Profile" component={ProfileScreen} />
      <Stack.Screen name="Settings" component={SettingsScreen} />
    </Stack.Navigator>
  );
}

// Root Navigator
export function AppNavigator() {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) {
    return (
      <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    );
  }

  return (
    <NavigationContainer>
      {isAuthenticated ? <MainNavigator /> : <AuthNavigator />}
    </NavigationContainer>
  );
}
```

---

## Token Refresh

### Interceptor สำหรับ Token Refresh อัตโนมัติ

```typescript
// src/services/setupInterceptors.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { apiClient } from './axiosInstance';
import { secureStorage } from '../utils/secureStorage';

interface RetryableRequest extends InternalAxiosRequestConfig {
  _retry?: boolean;
}

let isRefreshing = false;
let failedQueue: Array<{
  resolve: (value: string) => void;
  reject: (reason?: any) => void;
}> = [];

function processQueue(error: any, token: string | null = null) {
  failedQueue.forEach(({ resolve, reject }) => {
    if (error) {
      reject(error);
    } else {
      resolve(token!);
    }
  });
  failedQueue = [];
}

export function setupTokenRefreshInterceptor(onLogout: () => void) {
  apiClient.interceptors.response.use(
    (response) => response,
    async (error: AxiosError) => {
      const originalRequest = error.config as RetryableRequest;

      if (error.response?.status !== 401 || originalRequest._retry) {
        return Promise.reject(error);
      }

      if (isRefreshing) {
        return new Promise<string>((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then((token) => {
            originalRequest.headers!['Authorization'] = `Bearer ${token}`;
            return apiClient(originalRequest);
          })
          .catch((err) => Promise.reject(err));
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const refreshToken = await secureStorage.getRefreshToken();

        if (!refreshToken) {
          throw new Error('No refresh token available');
        }

        const response = await axios.post(
          `${apiClient.defaults.baseURL}/auth/refresh`,
          { refreshToken }
        );

        const { accessToken, refreshToken: newRefreshToken } = response.data;

        await secureStorage.saveTokens(accessToken, newRefreshToken);
        apiClient.defaults.headers.common['Authorization'] =
          `Bearer ${accessToken}`;

        processQueue(null, accessToken);
        originalRequest.headers!['Authorization'] = `Bearer ${accessToken}`;

        return apiClient(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        await secureStorage.clearAll();
        onLogout();
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }
  );
}
```

---

## Workshop: Complete Auth Flow

### Login Screen

```typescript
// src/screens/auth/LoginScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
  ScrollView,
  Alert,
  ActivityIndicator,
} from 'react-native';
import { useAuth } from '../../contexts/AuthContext';

interface FormErrors {
  email?: string;
  password?: string;
}

export const LoginScreen: React.FC = ({ navigation }: any) => {
  const { login } = useAuth();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [showPassword, setShowPassword] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const [errors, setErrors] = useState<FormErrors>({});

  const validate = (): boolean => {
    const newErrors: FormErrors = {};

    if (!email) {
      newErrors.email = 'กรุณาใส่ email';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      newErrors.email = 'รูปแบบ email ไม่ถูกต้อง';
    }

    if (!password) {
      newErrors.password = 'กรุณาใส่รหัสผ่าน';
    } else if (password.length < 6) {
      newErrors.password = 'รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร';
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleLogin = async () => {
    if (!validate()) return;

    setIsLoading(true);
    try {
      await login(email, password);
      // Navigation จะเกิดขึ้นอัตโนมัติจาก AppNavigator
    } catch (error: any) {
      Alert.alert(
        'เข้าสู่ระบบล้มเหลว',
        error.message || 'อีเมลหรือรหัสผ่านไม่ถูกต้อง'
      );
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView
        contentContainerStyle={styles.scrollContent}
        keyboardShouldPersistTaps="handled"
      >
        {/* Logo */}
        <View style={styles.logoContainer}>
          <Text style={styles.logoText}>🔐</Text>
          <Text style={styles.appName}>MyApp</Text>
          <Text style={styles.tagline}>เข้าสู่ระบบเพื่อดำเนินการต่อ</Text>
        </View>

        {/* Form */}
        <View style={styles.form}>
          {/* Email */}
          <View style={styles.inputGroup}>
            <Text style={styles.label}>อีเมล</Text>
            <TextInput
              value={email}
              onChangeText={(text) => {
                setEmail(text);
                if (errors.email) setErrors((e) => ({ ...e, email: undefined }));
              }}
              placeholder="example@email.com"
              keyboardType="email-address"
              autoCapitalize="none"
              autoCorrect={false}
              style={[styles.input, errors.email && styles.inputError]}
            />
            {errors.email && (
              <Text style={styles.errorText}>{errors.email}</Text>
            )}
          </View>

          {/* Password */}
          <View style={styles.inputGroup}>
            <Text style={styles.label}>รหัสผ่าน</Text>
            <View style={[styles.passwordContainer, errors.password && styles.inputError]}>
              <TextInput
                value={password}
                onChangeText={(text) => {
                  setPassword(text);
                  if (errors.password)
                    setErrors((e) => ({ ...e, password: undefined }));
                }}
                placeholder="รหัสผ่าน"
                secureTextEntry={!showPassword}
                style={styles.passwordInput}
              />
              <TouchableOpacity
                onPress={() => setShowPassword(!showPassword)}
                style={styles.eyeButton}
              >
                <Text style={styles.eyeIcon}>
                  {showPassword ? '👁️' : '🙈'}
                </Text>
              </TouchableOpacity>
            </View>
            {errors.password && (
              <Text style={styles.errorText}>{errors.password}</Text>
            )}
          </View>

          {/* Forgot Password */}
          <TouchableOpacity
            onPress={() => navigation.navigate('ForgotPassword')}
            style={styles.forgotPassword}
          >
            <Text style={styles.forgotPasswordText}>ลืมรหัสผ่าน?</Text>
          </TouchableOpacity>

          {/* Login Button */}
          <TouchableOpacity
            onPress={handleLogin}
            style={[styles.loginButton, isLoading && styles.disabledButton]}
            disabled={isLoading}
          >
            {isLoading ? (
              <ActivityIndicator color="#fff" />
            ) : (
              <Text style={styles.loginButtonText}>เข้าสู่ระบบ</Text>
            )}
          </TouchableOpacity>

          {/* Divider */}
          <View style={styles.divider}>
            <View style={styles.dividerLine} />
            <Text style={styles.dividerText}>หรือ</Text>
            <View style={styles.dividerLine} />
          </View>

          {/* Register Link */}
          <View style={styles.registerRow}>
            <Text style={styles.registerText}>ยังไม่มีบัญชี? </Text>
            <TouchableOpacity onPress={() => navigation.navigate('Register')}>
              <Text style={styles.registerLink}>สมัครสมาชิก</Text>
            </TouchableOpacity>
          </View>
        </View>
      </ScrollView>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  scrollContent: { flexGrow: 1, paddingHorizontal: 24 },
  logoContainer: { alignItems: 'center', paddingTop: 60, paddingBottom: 40 },
  logoText: { fontSize: 72, marginBottom: 12 },
  appName: { fontSize: 28, fontWeight: 'bold', color: '#6200EE' },
  tagline: { fontSize: 14, color: '#666', marginTop: 8 },
  form: {},
  inputGroup: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', marginBottom: 6, color: '#333' },
  input: {
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    fontSize: 16,
    backgroundColor: '#F9F9F9',
  },
  inputError: { borderColor: '#F44336' },
  passwordContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    backgroundColor: '#F9F9F9',
    overflow: 'hidden',
  },
  passwordInput: { flex: 1, padding: 14, fontSize: 16 },
  eyeButton: { padding: 14 },
  eyeIcon: { fontSize: 20 },
  errorText: { color: '#F44336', fontSize: 12, marginTop: 4 },
  forgotPassword: { alignSelf: 'flex-end', marginBottom: 20 },
  forgotPasswordText: { color: '#6200EE', fontSize: 14 },
  loginButton: {
    backgroundColor: '#6200EE',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginBottom: 20,
  },
  disabledButton: { opacity: 0.7 },
  loginButtonText: { color: '#fff', fontSize: 16, fontWeight: 'bold' },
  divider: { flexDirection: 'row', alignItems: 'center', marginBottom: 20 },
  dividerLine: { flex: 1, height: 1, backgroundColor: '#E0E0E0' },
  dividerText: { marginHorizontal: 12, color: '#999' },
  registerRow: { flexDirection: 'row', justifyContent: 'center' },
  registerText: { fontSize: 14, color: '#666' },
  registerLink: { fontSize: 14, color: '#6200EE', fontWeight: 'bold' },
});
```

### Register Screen

```typescript
// src/screens/auth/RegisterScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
  ScrollView,
  Alert,
  ActivityIndicator,
} from 'react-native';
import { useAuth } from '../../contexts/AuthContext';

export const RegisterScreen: React.FC = ({ navigation }: any) => {
  const { register } = useAuth();
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [errors, setErrors] = useState<{ [key: string]: string }>({});

  const validate = (): boolean => {
    const newErrors: { [key: string]: string } = {};

    if (!name.trim()) newErrors.name = 'กรุณาใส่ชื่อ';

    if (!email) {
      newErrors.email = 'กรุณาใส่ email';
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      newErrors.email = 'รูปแบบ email ไม่ถูกต้อง';
    }

    if (!password) {
      newErrors.password = 'กรุณาใส่รหัสผ่าน';
    } else if (password.length < 8) {
      newErrors.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
    } else if (!/[A-Z]/.test(password)) {
      newErrors.password = 'รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว';
    } else if (!/[0-9]/.test(password)) {
      newErrors.password = 'รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว';
    }

    if (password !== confirmPassword) {
      newErrors.confirmPassword = 'รหัสผ่านไม่ตรงกัน';
    }

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleRegister = async () => {
    if (!validate()) return;

    setIsLoading(true);
    try {
      await register(name, email, password);
    } catch (error: any) {
      Alert.alert('สมัครสมาชิกล้มเหลว', error.message || 'กรุณาลองใหม่');
    } finally {
      setIsLoading(false);
    }
  };

  const renderInput = (
    label: string,
    value: string,
    onChangeText: (text: string) => void,
    placeholder: string,
    errorKey: string,
    options?: {
      secureTextEntry?: boolean;
      keyboardType?: any;
      autoCapitalize?: any;
    }
  ) => (
    <View style={styles.inputGroup}>
      <Text style={styles.label}>{label}</Text>
      <TextInput
        value={value}
        onChangeText={(text) => {
          onChangeText(text);
          if (errors[errorKey]) {
            setErrors((e) => ({ ...e, [errorKey]: '' }));
          }
        }}
        placeholder={placeholder}
        style={[styles.input, errors[errorKey] && styles.inputError]}
        autoCorrect={false}
        {...options}
      />
      {errors[errorKey] ? (
        <Text style={styles.errorText}>{errors[errorKey]}</Text>
      ) : null}
    </View>
  );

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView
        contentContainerStyle={styles.scrollContent}
        keyboardShouldPersistTaps="handled"
      >
        <View style={styles.header}>
          <Text style={styles.title}>สร้างบัญชีใหม่</Text>
          <Text style={styles.subtitle}>กรอกข้อมูลเพื่อเริ่มใช้งาน</Text>
        </View>

        {renderInput('ชื่อ-นามสกุล', name, setName, 'ชื่อ นามสกุล', 'name', {
          autoCapitalize: 'words',
        })}
        {renderInput('อีเมล', email, setEmail, 'example@email.com', 'email', {
          keyboardType: 'email-address',
          autoCapitalize: 'none',
        })}
        {renderInput(
          'รหัสผ่าน',
          password,
          setPassword,
          'รหัสผ่านอย่างน้อย 8 ตัวอักษร',
          'password',
          { secureTextEntry: true }
        )}
        {renderInput(
          'ยืนยันรหัสผ่าน',
          confirmPassword,
          setConfirmPassword,
          'ยืนยันรหัสผ่าน',
          'confirmPassword',
          { secureTextEntry: true }
        )}

        {/* Password strength indicator */}
        {password.length > 0 && (
          <PasswordStrengthIndicator password={password} />
        )}

        <TouchableOpacity
          onPress={handleRegister}
          style={[styles.registerButton, isLoading && styles.disabledButton]}
          disabled={isLoading}
        >
          {isLoading ? (
            <ActivityIndicator color="#fff" />
          ) : (
            <Text style={styles.registerButtonText}>สมัครสมาชิก</Text>
          )}
        </TouchableOpacity>

        <View style={styles.loginRow}>
          <Text style={styles.loginText}>มีบัญชีแล้ว? </Text>
          <TouchableOpacity onPress={() => navigation.navigate('Login')}>
            <Text style={styles.loginLink}>เข้าสู่ระบบ</Text>
          </TouchableOpacity>
        </View>
      </ScrollView>
    </KeyboardAvoidingView>
  );
};

// Password Strength Component
const PasswordStrengthIndicator: React.FC<{ password: string }> = ({
  password,
}) => {
  const checks = [
    { label: 'ความยาว 8+ ตัว', pass: password.length >= 8 },
    { label: 'มีตัวพิมพ์ใหญ่', pass: /[A-Z]/.test(password) },
    { label: 'มีตัวพิมพ์เล็ก', pass: /[a-z]/.test(password) },
    { label: 'มีตัวเลข', pass: /[0-9]/.test(password) },
    { label: 'มีอักขระพิเศษ', pass: /[^A-Za-z0-9]/.test(password) },
  ];

  const strength = checks.filter((c) => c.pass).length;
  const strengthColors = ['#F44336', '#FF9800', '#FFC107', '#8BC34A', '#4CAF50'];
  const strengthLabels = ['อ่อนมาก', 'อ่อน', 'ปานกลาง', 'ดี', 'แข็งแกร่ง'];

  return (
    <View style={pwStyles.container}>
      <View style={pwStyles.barContainer}>
        {[1, 2, 3, 4, 5].map((i) => (
          <View
            key={i}
            style={[
              pwStyles.bar,
              { backgroundColor: i <= strength ? strengthColors[strength - 1] : '#E0E0E0' },
            ]}
          />
        ))}
      </View>
      <Text style={[pwStyles.strengthLabel, { color: strengthColors[strength - 1] || '#999' }]}>
        {strength > 0 ? strengthLabels[strength - 1] : ''}
      </Text>
      <View style={pwStyles.checkList}>
        {checks.map((check, i) => (
          <View key={i} style={pwStyles.checkItem}>
            <Text style={check.pass ? pwStyles.checkPass : pwStyles.checkFail}>
              {check.pass ? '✓' : '✗'}
            </Text>
            <Text style={pwStyles.checkLabel}>{check.label}</Text>
          </View>
        ))}
      </View>
    </View>
  );
};

const pwStyles = StyleSheet.create({
  container: { marginBottom: 16 },
  barContainer: { flexDirection: 'row', gap: 4, marginBottom: 4 },
  bar: { flex: 1, height: 4, borderRadius: 2 },
  strengthLabel: { fontSize: 12, fontWeight: 'bold', marginBottom: 8 },
  checkList: { gap: 4 },
  checkItem: { flexDirection: 'row', alignItems: 'center', gap: 6 },
  checkPass: { color: '#4CAF50', fontWeight: 'bold', width: 14 },
  checkFail: { color: '#999', width: 14 },
  checkLabel: { fontSize: 12, color: '#666' },
});

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  scrollContent: { flexGrow: 1, paddingHorizontal: 24, paddingVertical: 32 },
  header: { marginBottom: 24 },
  title: { fontSize: 28, fontWeight: 'bold', color: '#333' },
  subtitle: { fontSize: 14, color: '#666', marginTop: 4 },
  inputGroup: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', marginBottom: 6, color: '#333' },
  input: {
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    fontSize: 16,
    backgroundColor: '#F9F9F9',
  },
  inputError: { borderColor: '#F44336' },
  errorText: { color: '#F44336', fontSize: 12, marginTop: 4 },
  registerButton: {
    backgroundColor: '#6200EE',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginVertical: 20,
  },
  disabledButton: { opacity: 0.7 },
  registerButtonText: { color: '#fff', fontSize: 16, fontWeight: 'bold' },
  loginRow: { flexDirection: 'row', justifyContent: 'center' },
  loginText: { fontSize: 14, color: '#666' },
  loginLink: { fontSize: 14, color: '#6200EE', fontWeight: 'bold' },
});
```

---

## Tips and Best Practices

### 1. อย่าเก็บ Token ใน AsyncStorage

```typescript
// ❌ ไม่ปลอดภัย - AsyncStorage ไม่ encrypt
AsyncStorage.setItem('token', token);

// ✅ ปลอดภัยกว่า - ใช้ SecureStore (encrypted)
SecureStore.setItemAsync('token', token);
```

### 2. ตรวจสอบ Token Expiry ก่อน Request

```typescript
const makeAuthenticatedRequest = async () => {
  const token = await secureStorage.getAccessToken();

  if (token && isTokenExpiredSoon(token)) {
    // Refresh ก่อนที่จะหมดอายุ (5 นาทีก่อน)
    await refreshTokens();
  }

  return apiClient.get('/protected-route');
};

const isTokenExpiredSoon = (token: string): boolean => {
  const decoded = jwtDecode<{ exp: number }>(token);
  const fiveMinutes = 5 * 60;
  return decoded.exp - Date.now() / 1000 < fiveMinutes;
};
```

### 3. Handle Auth State ด้วย Deep Link

```typescript
// เมื่อกด link จาก email verification
const handleDeepLink = (url: string) => {
  if (url.includes('verify-email')) {
    const token = url.split('token=')[1];
    authApi.verifyEmail(token).then(() => {
      Alert.alert('ยืนยันอีเมลสำเร็จ');
    });
  }
};
```

---

## สรุป

JWT Authentication ใน React Native:

1. **JWT Structure** - Header + Payload + Signature
2. **SecureStore** - เก็บ tokens อย่างปลอดภัย (encrypted)
3. **Auth Context** - จัดการ auth state ทั่ว app
4. **Token Refresh** - อัตโนมัติผ่าน interceptor
5. **Protected Routes** - แยก navigators ตาม auth state
6. **Form Validation** - ตรวจสอบข้อมูลก่อน submit

ใน Part 037 จะเรียนเรื่อง **OAuth และ Social Login** ด้วย Google, Facebook, Apple
