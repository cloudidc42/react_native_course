# Part 059: Error Handling และ Error Boundaries ใน React Native

## ความเข้าใจ Error Handling

Error handling ที่ดีทำให้แอปไม่ crash โดยกะทันหัน และแสดง UI ที่เป็นมิตรต่อผู้ใช้แทนที่จะแสดงหน้า white screen

### ประเภทของ Errors

```
1. Synchronous Errors - เกิดระหว่าง render
2. Asynchronous Errors - เกิดใน async functions
3. Network Errors - การเชื่อมต่อล้มเหลว
4. Native Errors - เกิดใน native modules
5. Unhandled Promise Rejections - Promise ที่ไม่มี catch
```

---

## Error Boundaries

Error Boundary คือ React component พิเศษที่ดักจับ JavaScript errors ในส่วนของ component tree ข้างล่าง

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';
import { View, Text, TouchableOpacity, StyleSheet, ScrollView } from 'react-native';

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, info: ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
  errorInfo: ErrorInfo | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null,
    };
  }

  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    // Update state เพื่อ render fallback UI
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    // บันทึก error สำหรับ debugging
    console.error('Error caught by boundary:', error, errorInfo);
    
    this.setState({ errorInfo });
    
    // เรียก callback ถ้ามี
    this.props.onError?.(error, errorInfo);
    
    // ส่ง error ไปยัง crash reporting service
    // crashlytics().recordError(error);
  }

  handleReset = (): void => {
    this.setState({ hasError: false, error: null, errorInfo: null });
  };

  render(): ReactNode {
    if (this.state.hasError) {
      // Render fallback UI
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <View style={ebStyles.errorContainer}>
          <Text style={ebStyles.errorIcon}>💥</Text>
          <Text style={ebStyles.errorTitle}>เกิดข้อผิดพลาด</Text>
          <Text style={ebStyles.errorMessage}>
            {this.state.error?.message || 'เกิดข้อผิดพลาดที่ไม่คาดคิด'}
          </Text>
          
          {__DEV__ && this.state.errorInfo && (
            <ScrollView style={ebStyles.stackContainer}>
              <Text style={ebStyles.stackTitle}>Stack Trace (Dev Only):</Text>
              <Text style={ebStyles.stackText}>
                {this.state.errorInfo.componentStack}
              </Text>
            </ScrollView>
          )}
          
          <TouchableOpacity style={ebStyles.resetButton} onPress={this.handleReset}>
            <Text style={ebStyles.resetButtonText}>ลองใหม่</Text>
          </TouchableOpacity>
        </View>
      );
    }

    return this.props.children;
  }
}

const ebStyles = StyleSheet.create({
  errorContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
    backgroundColor: '#FFF5F5',
  },
  errorIcon: { fontSize: 60, marginBottom: 16 },
  errorTitle: { fontSize: 22, fontWeight: 'bold', color: '#C62828', marginBottom: 8 },
  errorMessage: { fontSize: 15, color: '#666', textAlign: 'center', marginBottom: 16 },
  stackContainer: {
    backgroundColor: '#1a1a1a',
    borderRadius: 8,
    padding: 12,
    maxHeight: 200,
    width: '100%',
    marginBottom: 16,
  },
  stackTitle: { color: '#F44336', fontSize: 12, marginBottom: 5, fontWeight: 'bold' },
  stackText: { color: '#ddd', fontSize: 10, fontFamily: 'monospace', lineHeight: 16 },
  resetButton: {
    backgroundColor: '#F44336',
    paddingHorizontal: 30,
    paddingVertical: 12,
    borderRadius: 25,
  },
  resetButtonText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
});

// การใช้งาน
const AppWithErrorBoundary: React.FC = () => {
  return (
    <ErrorBoundary
      onError={(error, info) => {
        // Log to monitoring service
        console.log('Reporting error:', error.message);
      }}
    >
      <MainApp />
    </ErrorBoundary>
  );
};

// Error Boundary เฉพาะส่วน
const ProductSection: React.FC = () => {
  return (
    <ErrorBoundary
      fallback={
        <View style={{ padding: 20, backgroundColor: '#FFF3E0' }}>
          <Text style={{ color: '#E65100' }}>ไม่สามารถโหลดสินค้าได้</Text>
        </View>
      }
    >
      <ProductList />
    </ErrorBoundary>
  );
};
```

---

## Try/Catch ใน Async Functions

```typescript
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  ActivityIndicator,
  Alert,
  StyleSheet,
} from 'react-native';

// Custom Error classes
class NetworkError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
    this.name = 'NetworkError';
  }
}

class ValidationError extends Error {
  constructor(public field: string, message: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

class AuthError extends Error {
  constructor(message = 'กรุณาเข้าสู่ระบบใหม่') {
    super(message);
    this.name = 'AuthError';
  }
}

// Error Handler ที่รองรับหลาย error types
const handleError = (error: unknown): string => {
  if (error instanceof NetworkError) {
    switch (error.statusCode) {
      case 400: return 'ข้อมูลไม่ถูกต้อง';
      case 401: return 'กรุณาเข้าสู่ระบบใหม่';
      case 403: return 'ไม่มีสิทธิ์เข้าถึง';
      case 404: return 'ไม่พบข้อมูลที่ต้องการ';
      case 500: return 'เซิร์ฟเวอร์มีปัญหา กรุณาลองใหม่ภายหลัง';
      default: return `เกิดข้อผิดพลาด (${error.statusCode})`;
    }
  }
  
  if (error instanceof ValidationError) {
    return `${error.field}: ${error.message}`;
  }
  
  if (error instanceof AuthError) {
    return error.message;
  }
  
  if (error instanceof TypeError && error.message.includes('Network request failed')) {
    return 'ไม่มีการเชื่อมต่ออินเทอร์เน็ต';
  }
  
  if (error instanceof Error) {
    return error.message;
  }
  
  return 'เกิดข้อผิดพลาดที่ไม่คาดคิด';
};

// API Service พร้อม error handling
const apiCall = async <T>(
  url: string,
  options?: RequestInit
): Promise<T> => {
  try {
    const response = await fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers,
      },
    });

    if (!response.ok) {
      const errorBody = await response.json().catch(() => ({}));
      throw new NetworkError(
        response.status,
        errorBody.message || `HTTP Error ${response.status}`
      );
    }

    return response.json();
  } catch (error) {
    if (error instanceof NetworkError) throw error;
    
    // Network failure (no internet)
    if (error instanceof TypeError) {
      throw new NetworkError(0, 'ไม่มีการเชื่อมต่ออินเทอร์เน็ต');
    }
    
    throw error;
  }
};

// Component ที่ใช้ error handling
const DataFetchComponent: React.FC = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [data, setData] = useState<any>(null);

  const fetchData = useCallback(async () => {
    setLoading(true);
    setError(null);
    
    try {
      const result = await apiCall<any>('https://api.example.com/data');
      setData(result);
    } catch (err) {
      const message = handleError(err);
      setError(message);
      
      // แสดง alert สำหรับ auth error
      if (err instanceof AuthError) {
        Alert.alert(
          'Session หมดอายุ',
          'กรุณาเข้าสู่ระบบใหม่',
          [{ text: 'OK', onPress: () => {} }]
        );
      }
    } finally {
      setLoading(false);
    }
  }, []);

  return (
    <View style={asyncStyles.container}>
      <TouchableOpacity
        style={asyncStyles.button}
        onPress={fetchData}
        disabled={loading}
      >
        {loading ? (
          <ActivityIndicator color="white" />
        ) : (
          <Text style={asyncStyles.buttonText}>โหลดข้อมูล</Text>
        )}
      </TouchableOpacity>
      
      {error && (
        <View style={asyncStyles.errorBox}>
          <Text style={asyncStyles.errorText}>❌ {error}</Text>
          <TouchableOpacity onPress={fetchData}>
            <Text style={asyncStyles.retryText}>ลองใหม่</Text>
          </TouchableOpacity>
        </View>
      )}
      
      {data && (
        <View style={asyncStyles.dataBox}>
          <Text style={asyncStyles.dataText}>✅ โหลดข้อมูลสำเร็จ</Text>
        </View>
      )}
    </View>
  );
};

const asyncStyles = StyleSheet.create({
  container: { padding: 20 },
  button: {
    backgroundColor: '#2196F3',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 15,
  },
  buttonText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
  errorBox: {
    backgroundColor: '#FFEBEE',
    padding: 14,
    borderRadius: 10,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  errorText: { color: '#C62828', flex: 1, fontSize: 14 },
  retryText: { color: '#1976D2', fontWeight: 'bold', fontSize: 14 },
  dataBox: {
    backgroundColor: '#E8F5E9',
    padding: 14,
    borderRadius: 10,
  },
  dataText: { color: '#2E7D32', fontSize: 14, fontWeight: '500' },
});
```

---

## Global Error Handler

```typescript
import { ErrorUtils } from 'react-native';

const setupGlobalErrorHandler = () => {
  // จัดการ JS errors ที่ไม่ถูก catch
  const originalHandler = ErrorUtils.getGlobalHandler();
  
  ErrorUtils.setGlobalHandler((error: Error, isFatal: boolean) => {
    console.error('[GlobalError]', error.message, { isFatal });
    
    // ส่ง error ไปยัง crash reporting
    if (isFatal) {
      // crashlytics().recordError(error);
      // Sentry.captureException(error);
      
      // แสดง user-friendly message
      Alert.alert(
        'แอปเกิดข้อผิดพลาด',
        'เกิดข้อผิดพลาดร้ายแรง กรุณาเปิดแอปใหม่',
        [{ text: 'OK' }]
      );
    }
    
    // เรียก original handler
    if (!isFatal) {
      originalHandler(error, isFatal);
    }
  });

  // จัดการ unhandled Promise rejections
  if (typeof global !== 'undefined') {
    const tracking = global as any;
    const originalUnhandledRejection = tracking.onunhandledrejection;
    
    tracking.onunhandledrejection = (event: any) => {
      console.error('[UnhandledRejection]', event.reason);
      // crashlytics().recordError(event.reason);
      originalUnhandledRejection?.(event);
    };
  }
};

// เรียกใช้ใน index.js
// setupGlobalErrorHandler();
```

---

## Crash Reporting

```typescript
// Installation
// npm install @react-native-firebase/crashlytics
// หรือ
// npm install @sentry/react-native

// ตัวอย่างการใช้ Sentry
import * as Sentry from '@sentry/react-native';

const initCrashReporting = () => {
  Sentry.init({
    dsn: 'YOUR_SENTRY_DSN',
    environment: __DEV__ ? 'development' : 'production',
    enableNative: true,
    enableAutoSessionTracking: true,
    
    // Sampling
    tracesSampleRate: __DEV__ ? 1.0 : 0.2,
    
    // Before sending
    beforeSend(event) {
      // กรอง sensitive data
      if (event.request?.data) {
        delete event.request.data.password;
        delete event.request.data.token;
      }
      return event;
    },
  });
};

// บันทึก error ด้วยตนเอง
const reportError = (error: Error, context?: Record<string, any>) => {
  Sentry.withScope(scope => {
    if (context) {
      scope.setContext('additional', context);
    }
    Sentry.captureException(error);
  });
};

// บันทึก user info
const setUserContext = (userId: string, email: string) => {
  Sentry.setUser({ id: userId, email });
};

// บันทึก custom event
const trackEvent = (eventName: string, data?: Record<string, any>) => {
  Sentry.addBreadcrumb({
    category: 'user',
    message: eventName,
    data,
    level: 'info',
  });
};
```

---

## User-Friendly Error Messages

```typescript
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Animated,
  Modal,
  ActivityIndicator,
} from 'react-native';

type ErrorSeverity = 'info' | 'warning' | 'error' | 'success';

interface AppError {
  message: string;
  severity: ErrorSeverity;
  action?: {
    label: string;
    handler: () => void;
  };
  autoHide?: boolean;
  duration?: number;
}

// Toast Notification Component
const Toast: React.FC<{
  error: AppError;
  onDismiss: () => void;
}> = ({ error, onDismiss }) => {
  const opacity = React.useRef(new Animated.Value(0)).current;

  React.useEffect(() => {
    Animated.timing(opacity, {
      toValue: 1,
      duration: 300,
      useNativeDriver: true,
    }).start();

    if (error.autoHide) {
      const timer = setTimeout(() => {
        Animated.timing(opacity, {
          toValue: 0,
          duration: 300,
          useNativeDriver: true,
        }).start(() => onDismiss());
      }, error.duration || 3000);

      return () => clearTimeout(timer);
    }
  }, []);

  const bgColors: Record<ErrorSeverity, string> = {
    info: '#2196F3',
    warning: '#FF9800',
    error: '#F44336',
    success: '#4CAF50',
  };

  const icons: Record<ErrorSeverity, string> = {
    info: 'ℹ️',
    warning: '⚠️',
    error: '❌',
    success: '✅',
  };

  return (
    <Animated.View style={[
      toastStyles.container,
      { backgroundColor: bgColors[error.severity], opacity }
    ]}>
      <Text style={toastStyles.icon}>{icons[error.severity]}</Text>
      <Text style={toastStyles.message}>{error.message}</Text>
      {error.action && (
        <TouchableOpacity onPress={error.action.handler}>
          <Text style={toastStyles.actionText}>{error.action.label}</Text>
        </TouchableOpacity>
      )}
      <TouchableOpacity onPress={onDismiss} style={toastStyles.closeButton}>
        <Text style={toastStyles.closeText}>✕</Text>
      </TouchableOpacity>
    </Animated.View>
  );
};

const toastStyles = StyleSheet.create({
  container: {
    position: 'absolute',
    top: 50,
    left: 15,
    right: 15,
    borderRadius: 12,
    padding: 14,
    flexDirection: 'row',
    alignItems: 'center',
    elevation: 10,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    zIndex: 9999,
  },
  icon: { fontSize: 20, marginRight: 8 },
  message: { flex: 1, color: 'white', fontSize: 14, fontWeight: '500' },
  actionText: { color: 'rgba(255,255,255,0.9)', fontWeight: 'bold', marginHorizontal: 8, textDecorationLine: 'underline' },
  closeButton: { marginLeft: 5, padding: 2 },
  closeText: { color: 'rgba(255,255,255,0.7)', fontSize: 16 },
});

// Error Context
interface ErrorContextType {
  showError: (error: AppError) => void;
  showSuccess: (message: string) => void;
  showWarning: (message: string) => void;
}

const ErrorContext = React.createContext<ErrorContextType>({
  showError: () => {},
  showSuccess: () => {},
  showWarning: () => {},
});

export const ErrorProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [currentError, setCurrentError] = useState<AppError | null>(null);

  const showError = useCallback((error: AppError) => {
    setCurrentError(error);
  }, []);

  const showSuccess = useCallback((message: string) => {
    setCurrentError({
      message,
      severity: 'success',
      autoHide: true,
      duration: 2000,
    });
  }, []);

  const showWarning = useCallback((message: string) => {
    setCurrentError({
      message,
      severity: 'warning',
      autoHide: true,
      duration: 3000,
    });
  }, []);

  return (
    <ErrorContext.Provider value={{ showError, showSuccess, showWarning }}>
      {children}
      {currentError && (
        <Toast
          error={currentError}
          onDismiss={() => setCurrentError(null)}
        />
      )}
    </ErrorContext.Provider>
  );
};

export const useError = () => React.useContext(ErrorContext);
```

---

## Workshop: Robust Error Handling

```typescript
import React, { useState, useEffect, useCallback, useRef } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  TextInput,
  ActivityIndicator,
} from 'react-native';

// Retry Logic
async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    delay?: number;
    onRetry?: (attempt: number, error: Error) => void;
  } = {}
): Promise<T> {
  const { maxRetries = 3, delay = 1000, onRetry } = options;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      
      const err = error instanceof Error ? error : new Error(String(error));
      onRetry?.(attempt, err);
      
      // Exponential backoff
      await new Promise(resolve => setTimeout(resolve, delay * Math.pow(2, attempt - 1)));
    }
  }
  
  throw new Error('Max retries exceeded');
}

// Form Validation with Errors
interface FormData {
  email: string;
  password: string;
  confirmPassword: string;
}

interface FormErrors {
  email?: string;
  password?: string;
  confirmPassword?: string;
  general?: string;
}

const validateForm = (data: FormData): FormErrors => {
  const errors: FormErrors = {};
  
  if (!data.email) {
    errors.email = 'กรุณาใส่ email';
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
    errors.email = 'รูปแบบ email ไม่ถูกต้อง';
  }
  
  if (!data.password) {
    errors.password = 'กรุณาใส่รหัสผ่าน';
  } else if (data.password.length < 8) {
    errors.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
  } else if (!/(?=.*[A-Z])(?=.*[0-9])/.test(data.password)) {
    errors.password = 'รหัสผ่านต้องมีตัวพิมพ์ใหญ่และตัวเลข';
  }
  
  if (data.password !== data.confirmPassword) {
    errors.confirmPassword = 'รหัสผ่านไม่ตรงกัน';
  }
  
  return errors;
};

// Main Workshop Component
const RobustErrorHandlingWorkshop: React.FC = () => {
  const [formData, setFormData] = useState<FormData>({
    email: '',
    password: '',
    confirmPassword: '',
  });
  const [formErrors, setFormErrors] = useState<FormErrors>({});
  const [loading, setLoading] = useState(false);
  const [submitResult, setSubmitResult] = useState<string>('');
  const [retryAttempts, setRetryAttempts] = useState(0);
  const isMountedRef = useRef(true);

  useEffect(() => {
    return () => { isMountedRef.current = false; };
  }, []);

  const updateField = useCallback((field: keyof FormData, value: string) => {
    setFormData(prev => ({ ...prev, [field]: value }));
    // Clear error เมื่อผู้ใช้พิมพ์
    if (formErrors[field]) {
      setFormErrors(prev => ({ ...prev, [field]: undefined }));
    }
  }, [formErrors]);

  const handleSubmit = async () => {
    // Validate
    const errors = validateForm(formData);
    if (Object.keys(errors).length > 0) {
      setFormErrors(errors);
      return;
    }
    
    setLoading(true);
    setSubmitResult('');
    setRetryAttempts(0);
    
    try {
      await withRetry(
        async () => {
          // จำลอง API call ที่อาจล้มเหลว
          const shouldFail = Math.random() < 0.6; // 60% chance ล้มเหลว
          
          if (shouldFail) {
            throw new NetworkError(500, 'Server Error');
          }
          
          await new Promise(resolve => setTimeout(resolve, 1000));
          return { success: true };
        },
        {
          maxRetries: 3,
          delay: 500,
          onRetry: (attempt, error) => {
            if (isMountedRef.current) {
              setRetryAttempts(attempt);
              console.log(`Retry attempt ${attempt}: ${error.message}`);
            }
          },
        }
      );
      
      if (isMountedRef.current) {
        setSubmitResult('สมัครสมาชิกสำเร็จ! 🎉');
      }
    } catch (error) {
      if (isMountedRef.current) {
        const message = handleError(error);
        setFormErrors(prev => ({ ...prev, general: message }));
        setSubmitResult('');
      }
    } finally {
      if (isMountedRef.current) {
        setLoading(false);
        setRetryAttempts(0);
      }
    }
  };

  return (
    <ErrorBoundary>
      <ScrollView style={workshopStyles.container}>
        <Text style={workshopStyles.title}>Error Handling Workshop</Text>
        <Text style={workshopStyles.subtitle}>ฟอร์มสมัครสมาชิกพร้อม Error Handling</Text>

        {/* General Error */}
        {formErrors.general && (
          <View style={workshopStyles.generalError}>
            <Text style={workshopStyles.generalErrorText}>
              ❌ {formErrors.general}
            </Text>
            <TouchableOpacity onPress={() => setFormErrors(prev => ({ ...prev, general: undefined }))}>
              <Text style={workshopStyles.dismissText}>✕</Text>
            </TouchableOpacity>
          </View>
        )}

        {/* Success */}
        {submitResult && (
          <View style={workshopStyles.successBox}>
            <Text style={workshopStyles.successText}>{submitResult}</Text>
          </View>
        )}

        {/* Email Field */}
        <View style={workshopStyles.field}>
          <Text style={workshopStyles.label}>Email</Text>
          <TextInput
            style={[workshopStyles.input, formErrors.email && workshopStyles.inputError]}
            value={formData.email}
            onChangeText={text => updateField('email', text)}
            placeholder="your@email.com"
            keyboardType="email-address"
            autoCapitalize="none"
          />
          {formErrors.email && (
            <Text style={workshopStyles.fieldError}>{formErrors.email}</Text>
          )}
        </View>

        {/* Password Field */}
        <View style={workshopStyles.field}>
          <Text style={workshopStyles.label}>รหัสผ่าน</Text>
          <TextInput
            style={[workshopStyles.input, formErrors.password && workshopStyles.inputError]}
            value={formData.password}
            onChangeText={text => updateField('password', text)}
            placeholder="อย่างน้อย 8 ตัว มีตัวพิมพ์ใหญ่และตัวเลข"
            secureTextEntry
          />
          {formErrors.password && (
            <Text style={workshopStyles.fieldError}>{formErrors.password}</Text>
          )}
        </View>

        {/* Confirm Password */}
        <View style={workshopStyles.field}>
          <Text style={workshopStyles.label}>ยืนยันรหัสผ่าน</Text>
          <TextInput
            style={[workshopStyles.input, formErrors.confirmPassword && workshopStyles.inputError]}
            value={formData.confirmPassword}
            onChangeText={text => updateField('confirmPassword', text)}
            placeholder="ใส่รหัสผ่านอีกครั้ง"
            secureTextEntry
          />
          {formErrors.confirmPassword && (
            <Text style={workshopStyles.fieldError}>{formErrors.confirmPassword}</Text>
          )}
        </View>

        {/* Submit Button */}
        <TouchableOpacity
          style={[workshopStyles.submitButton, loading && workshopStyles.submitButtonDisabled]}
          onPress={handleSubmit}
          disabled={loading}
        >
          {loading ? (
            <View style={workshopStyles.loadingRow}>
              <ActivityIndicator color="white" />
              {retryAttempts > 0 && (
                <Text style={workshopStyles.retryText}>
                  {' '}กำลังลองใหม่ครั้งที่ {retryAttempts}...
                </Text>
              )}
            </View>
          ) : (
            <Text style={workshopStyles.submitText}>สมัครสมาชิก</Text>
          )}
        </TouchableOpacity>

        {/* Error Handling Tips */}
        <View style={workshopStyles.tipsBox}>
          <Text style={workshopStyles.tipsTitle}>เทคนิค Error Handling ที่ใช้:</Text>
          <Text style={workshopStyles.tipItem}>• Validation ก่อน submit</Text>
          <Text style={workshopStyles.tipItem}>• Retry logic (3 ครั้ง) พร้อม exponential backoff</Text>
          <Text style={workshopStyles.tipItem}>• isMounted check ป้องกัน memory leak</Text>
          <Text style={workshopStyles.tipItem}>• User-friendly error messages</Text>
          <Text style={workshopStyles.tipItem}>• Field-level errors</Text>
          <Text style={workshopStyles.tipItem}>• Error Boundary ป้องกัน crash</Text>
        </View>
      </ScrollView>
    </ErrorBoundary>
  );
};

const workshopStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  title: { fontSize: 22, fontWeight: 'bold', padding: 20, paddingBottom: 5, color: '#333' },
  subtitle: { fontSize: 14, color: '#888', paddingHorizontal: 20, marginBottom: 20 },
  generalError: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: '#FFEBEE',
    margin: 15,
    padding: 14,
    borderRadius: 10,
    borderLeftWidth: 4,
    borderLeftColor: '#F44336',
  },
  generalErrorText: { color: '#C62828', fontSize: 14, flex: 1 },
  dismissText: { color: '#F44336', fontSize: 18, fontWeight: 'bold' },
  successBox: {
    backgroundColor: '#E8F5E9',
    margin: 15,
    padding: 14,
    borderRadius: 10,
    borderLeftWidth: 4,
    borderLeftColor: '#4CAF50',
  },
  successText: { color: '#2E7D32', fontSize: 15, fontWeight: '500' },
  field: { marginHorizontal: 15, marginBottom: 15 },
  label: { fontSize: 14, fontWeight: '600', color: '#555', marginBottom: 6 },
  input: {
    backgroundColor: 'white',
    borderWidth: 1,
    borderColor: '#e0e0e0',
    borderRadius: 10,
    padding: 13,
    fontSize: 15,
  },
  inputError: { borderColor: '#F44336', backgroundColor: '#FFF5F5' },
  fieldError: { color: '#F44336', fontSize: 12, marginTop: 5 },
  submitButton: {
    backgroundColor: '#2196F3',
    margin: 15,
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  submitButtonDisabled: { backgroundColor: '#90CAF9' },
  loadingRow: { flexDirection: 'row', alignItems: 'center' },
  retryText: { color: 'white', fontSize: 14 },
  submitText: { color: 'white', fontWeight: 'bold', fontSize: 17 },
  tipsBox: {
    backgroundColor: '#E3F2FD',
    margin: 15,
    padding: 15,
    borderRadius: 12,
    marginBottom: 30,
  },
  tipsTitle: { fontSize: 15, fontWeight: 'bold', color: '#1565C0', marginBottom: 10 },
  tipItem: { fontSize: 13, color: '#1976D2', marginBottom: 5, lineHeight: 20 },
});

export default RobustErrorHandlingWorkshop;
```

---

## Tips สรุป Error Handling

### 1. Hierarchy ของ Error Handling
```
Global Handler → Error Boundary → Try/Catch → User Feedback
```

### 2. Error Classification
```typescript
type AppErrorType =
  | 'network'      // ปัญหา network
  | 'auth'         // authentication
  | 'validation'   // ข้อมูลไม่ถูกต้อง
  | 'not_found'    // ไม่พบ resource
  | 'permission'   // ไม่มีสิทธิ์
  | 'unknown';     // ไม่ทราบสาเหตุ
```

### 3. User-friendly Messages
```typescript
const userMessages: Record<string, string> = {
  NetworkError: 'ตรวจสอบการเชื่อมต่ออินเทอร์เน็ต',
  AuthError: 'กรุณาเข้าสู่ระบบใหม่',
  NotFoundError: 'ไม่พบข้อมูลที่ต้องการ',
  // อย่าแสดง technical details ให้ผู้ใช้ทั่วไป!
};
```

---

## สรุป

Error Handling ที่ดีต้องการ:
- **Error Boundaries**: ป้องกัน crash ทั้ง component tree
- **Try/Catch**: จัดการ async errors อย่างเหมาะสม
- **Global Handler**: จัดการ unhandled errors
- **Retry Logic**: ลองใหม่เมื่อ network fail
- **User-friendly Messages**: อธิบายปัญหาอย่างเป็นมิตร
- **Crash Reporting**: ติดตาม errors ใน production
- **Validation**: ตรวจสอบข้อมูลก่อน submit
