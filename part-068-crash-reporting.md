# Part 068: Crash Reporting ใน React Native

## บทนำ

Crash Reporting คือกระบวนการตรวจจับ บันทึก และรายงานข้อผิดพลาดที่เกิดขึ้นในแอป เพื่อให้นักพัฒนาสามารถระบุและแก้ไขปัญหาได้อย่างรวดเร็ว ในบทนี้เราจะเรียนรู้การ implement crash reporting ด้วย Firebase Crashlytics และ Sentry

---

## 1. ทำความเข้าใจ Crash Reporting

### ประเภทของ Errors

```
Fatal Crashes:
- แอปพังทันที (force close)
- เกิดจาก uncaught exceptions
- ผู้ใช้ต้องเปิดแอปใหม่

Non-Fatal Errors:
- ข้อผิดพลาดที่แอปยังทำงานต่อได้
- API failures
- Data parsing errors
- ควร log ไว้เพื่อติดตาม

ANR (Application Not Responding) - Android:
- แอปค้างและไม่ตอบสนอง
- เกิดเมื่อ main thread ทำงานนานเกิน 5 วินาที
```

### ข้อมูลที่ Crash Report ควรมี

```
- Stack trace (ลำดับการเรียก functions ก่อน crash)
- Device info (model, OS version, app version)
- User info (ถ้ามี consent)
- Breadcrumbs (ลำดับ events ก่อน crash)
- Custom logs
- Environment (dev/staging/production)
```

---

## 2. Firebase Crashlytics

### ติดตั้ง Firebase Crashlytics

```bash
npm install @react-native-firebase/app @react-native-firebase/crashlytics
cd ios && pod install
```

### ตั้งค่า Crashlytics

**android/app/build.gradle**:
```groovy
apply plugin: 'com.google.firebase.crashlytics'

android {
    // ...
}

dependencies {
    // ...
    implementation platform('com.google.firebase:firebase-bom:32.0.0')
    implementation 'com.google.firebase:firebase-crashlytics'
}
```

**ios/YourApp/AppDelegate.mm**:
```objc
#import <Firebase.h>

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application 
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  [FIRApp configure];
  // ...
}
```

### การใช้งาน Crashlytics

```typescript
// src/crashReporting/crashlytics.ts
import crashlytics from '@react-native-firebase/crashlytics';
import { Platform } from 'react-native';
import DeviceInfo from 'react-native-device-info';

class CrashlyticsService {
  async initialize(): Promise<void> {
    await crashlytics().setCrashlyticsCollectionEnabled(
      !__DEV__ // เปิดใช้งานเฉพาะ production
    );
    
    // ตั้งค่า device info เริ่มต้น
    await this.setDefaultAttributes();
  }

  private async setDefaultAttributes(): Promise<void> {
    await Promise.all([
      crashlytics().setAttribute('platform', Platform.OS),
      crashlytics().setAttribute('app_version', DeviceInfo.getVersion()),
      crashlytics().setAttribute('build_number', DeviceInfo.getBuildNumber()),
      crashlytics().setAttribute('device_model', await DeviceInfo.getModel()),
      crashlytics().setAttribute('os_version', DeviceInfo.getSystemVersion()),
    ]);
  }

  setUserId(userId: string): void {
    crashlytics().setUserId(userId);
  }

  setAttribute(key: string, value: string): void {
    crashlytics().setAttribute(key, value);
  }

  setAttributes(attributes: Record<string, string>): void {
    crashlytics().setAttributes(attributes);
  }

  log(message: string): void {
    crashlytics().log(message);
  }

  recordError(error: Error, jsErrorName?: string): void {
    crashlytics().recordError(error, jsErrorName);
  }

  crash(): void {
    crashlytics().crash(); // สำหรับ testing เท่านั้น
  }
}

export default new CrashlyticsService();
```

### Global Error Handler

```typescript
// src/crashReporting/errorHandler.ts
import { Platform } from 'react-native';
import CrashlyticsService from './crashlytics';

// เก็บ original handlers
const originalConsoleError = console.error;
const originalConsoleWarn = console.warn;

export const setupErrorHandlers = () => {
  // Handle JavaScript errors
  const ErrorUtils = (global as any).ErrorUtils;
  
  if (ErrorUtils) {
    const originalHandler = ErrorUtils.getGlobalHandler();
    
    ErrorUtils.setGlobalHandler((error: Error, isFatal: boolean) => {
      // Log breadcrumb
      CrashlyticsService.log(`[${isFatal ? 'FATAL' : 'ERROR'}] ${error.message}`);
      
      // Set error info
      CrashlyticsService.setAttribute('is_fatal', String(isFatal));
      
      if (isFatal) {
        // Record crash
        CrashlyticsService.recordError(error, 'Fatal JavaScript Error');
      } else {
        // Record non-fatal error
        CrashlyticsService.recordError(error, 'Non-Fatal JavaScript Error');
      }
      
      // Call original handler
      originalHandler && originalHandler(error, isFatal);
    });
  }

  // Handle Promise rejections
  const tracking = (global as any).HermesInternal?.getRuntimeProperties?.();
  
  if (tracking) {
    // Hermes engine tracks unhandled promise rejections
    (global as any).__onUnhandledPromiseRejection = (
      id: number,
      error: Error
    ) => {
      CrashlyticsService.log(`Unhandled Promise Rejection: ${error.message}`);
      CrashlyticsService.recordError(error, 'Unhandled Promise Rejection');
    };
  }

  // Override console.error to capture in crashlytics
  console.error = (...args: unknown[]) => {
    const message = args.map(a => 
      typeof a === 'object' ? JSON.stringify(a) : String(a)
    ).join(' ');
    
    CrashlyticsService.log(`[console.error] ${message}`);
    originalConsoleError.apply(console, args);
  };
};
```

### Error Boundary Component

```typescript
// src/components/ErrorBoundary.tsx
import React, { Component, ErrorInfo } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
} from 'react-native';
import CrashlyticsService from '../crashReporting/crashlytics';

interface Props {
  children: React.ReactNode;
  fallback?: React.ComponentType<{ error: Error; retry: () => void }>;
  onError?: (error: Error, info: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
  errorInfo: ErrorInfo | null;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null,
    };
  }

  static getDerivedStateFromError(error: Error): Partial<State> {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    // Log to Crashlytics
    CrashlyticsService.log(`ErrorBoundary caught: ${error.message}`);
    CrashlyticsService.setAttribute('component_stack', 
      errorInfo.componentStack?.substring(0, 500) || '');
    CrashlyticsService.recordError(error, 'React Error Boundary');
    
    // Update state
    this.setState({ errorInfo });
    
    // Call custom handler
    this.props.onError?.(error, errorInfo);
  }

  handleRetry = () => {
    this.setState({ hasError: false, error: null, errorInfo: null });
  };

  render() {
    const { hasError, error } = this.state;
    const { children, fallback: Fallback } = this.props;

    if (hasError && error) {
      if (Fallback) {
        return <Fallback error={error} retry={this.handleRetry} />;
      }

      return (
        <View style={styles.container}>
          <Text style={styles.emoji}>😵</Text>
          <Text style={styles.title}>Something went wrong</Text>
          <Text style={styles.message}>
            We're sorry! The app encountered an unexpected error.
          </Text>
          
          {__DEV__ && (
            <ScrollView style={styles.errorDetails}>
              <Text style={styles.errorMessage}>{error.message}</Text>
              <Text style={styles.errorStack}>{error.stack}</Text>
            </ScrollView>
          )}
          
          <TouchableOpacity style={styles.retryButton} onPress={this.handleRetry}>
            <Text style={styles.retryText}>Try Again</Text>
          </TouchableOpacity>
        </View>
      );
    }

    return children;
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
    backgroundColor: '#fff',
  },
  emoji: {
    fontSize: 64,
    marginBottom: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
    textAlign: 'center',
  },
  message: {
    fontSize: 16,
    color: '#666',
    textAlign: 'center',
    marginBottom: 24,
    lineHeight: 24,
  },
  errorDetails: {
    maxHeight: 200,
    width: '100%',
    backgroundColor: '#f5f5f5',
    borderRadius: 8,
    padding: 12,
    marginBottom: 24,
  },
  errorMessage: {
    fontSize: 14,
    color: '#f44336',
    marginBottom: 8,
    fontFamily: 'monospace',
  },
  errorStack: {
    fontSize: 12,
    color: '#666',
    fontFamily: 'monospace',
  },
  retryButton: {
    backgroundColor: '#2196F3',
    paddingVertical: 12,
    paddingHorizontal: 32,
    borderRadius: 8,
  },
  retryText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default ErrorBoundary;
```

### Breadcrumbs System

```typescript
// src/crashReporting/breadcrumbs.ts
import crashlytics from '@react-native-firebase/crashlytics';

type BreadcrumbType = 'navigation' | 'user_action' | 'network' | 'error' | 'state_change';

interface Breadcrumb {
  type: BreadcrumbType;
  message: string;
  data?: Record<string, string>;
  timestamp: number;
}

class BreadcrumbTracker {
  private breadcrumbs: Breadcrumb[] = [];
  private maxBreadcrumbs = 20;

  add(
    type: BreadcrumbType,
    message: string,
    data?: Record<string, string | number | boolean>
  ): void {
    const breadcrumb: Breadcrumb = {
      type,
      message,
      data: data ? Object.entries(data).reduce((acc, [k, v]) => ({
        ...acc, [k]: String(v)
      }), {}) : undefined,
      timestamp: Date.now(),
    };

    this.breadcrumbs.push(breadcrumb);
    
    // Keep only last maxBreadcrumbs
    if (this.breadcrumbs.length > this.maxBreadcrumbs) {
      this.breadcrumbs.shift();
    }

    // Log to Crashlytics
    const logMessage = `[${type.toUpperCase()}] ${message}`;
    crashlytics().log(logMessage);
  }

  addNavigation(from: string, to: string): void {
    this.add('navigation', `Navigate: ${from} → ${to}`, { from, to });
  }

  addUserAction(action: string, context?: string): void {
    this.add('user_action', action, context ? { context } : undefined);
  }

  addNetworkRequest(url: string, method: string, statusCode: number): void {
    this.add('network', `${method} ${url}`, {
      status: statusCode,
      url,
      method,
    });
  }

  addError(error: string, context?: string): void {
    this.add('error', error, context ? { context } : undefined);
  }

  addStateChange(key: string, oldValue: string, newValue: string): void {
    this.add('state_change', `${key}: ${oldValue} → ${newValue}`, {
      key, oldValue, newValue,
    });
  }

  getAll(): Breadcrumb[] {
    return [...this.breadcrumbs];
  }

  clear(): void {
    this.breadcrumbs = [];
  }
}

export const breadcrumbs = new BreadcrumbTracker();
```

---

## 3. Sentry

### ติดตั้ง Sentry

```bash
npm install @sentry/react-native
cd ios && pod install

# Android - เพิ่มใน android/app/build.gradle
apply plugin: 'io.sentry.android.gradle'
```

### ตั้งค่า Sentry

```typescript
// src/crashReporting/sentry.ts
import * as Sentry from '@sentry/react-native';
import { Platform } from 'react-native';
import DeviceInfo from 'react-native-device-info';

export const initializeSentry = () => {
  Sentry.init({
    dsn: process.env.SENTRY_DSN || '',
    
    // Performance Monitoring
    tracesSampleRate: __DEV__ ? 1.0 : 0.1,
    
    // Session Tracking
    enableAutoSessionTracking: true,
    sessionTrackingIntervalMillis: 30000,
    
    // Environment
    environment: process.env.ENVIRONMENT || 'development',
    release: `${DeviceInfo.getBundleId()}@${DeviceInfo.getVersion()}+${DeviceInfo.getBuildNumber()}`,
    dist: DeviceInfo.getBuildNumber(),
    
    // Debug
    debug: __DEV__,
    
    // Filtering
    beforeSend(event) {
      // Filter development errors
      if (__DEV__) return null;
      
      // Filter known non-critical errors
      if (event.exception?.values?.[0]?.type === 'NetworkError') {
        return null;
      }
      
      return event;
    },
    
    // Breadcrumbs
    beforeBreadcrumb(breadcrumb) {
      // Filter sensitive data from breadcrumbs
      if (breadcrumb.category === 'http' && breadcrumb.data?.url) {
        const url = breadcrumb.data.url as string;
        if (url.includes('/auth/') || url.includes('/payment/')) {
          return null; // ไม่เก็บ auth และ payment requests
        }
      }
      return breadcrumb;
    },
    
    // Integrations
    integrations: [
      new Sentry.ReactNativeTracing({
        enableNativeFramesTracking: Platform.OS === 'ios',
        enableStallTracking: true,
      }),
    ],
  });
};
```

### Sentry User Context

```typescript
// src/crashReporting/sentryUser.ts
import * as Sentry from '@sentry/react-native';

interface UserContext {
  id: string;
  email?: string;
  username?: string;
  plan?: string;
}

export const setSentryUser = (user: UserContext): void => {
  Sentry.setUser({
    id: user.id,
    email: user.email, // เฉพาะถ้ามี consent
    username: user.username,
    data: {
      plan: user.plan,
    },
  });
};

export const clearSentryUser = (): void => {
  Sentry.setUser(null);
};

export const setSentryContext = (
  contextName: string,
  data: Record<string, unknown>
): void => {
  Sentry.setContext(contextName, data);
};

export const addSentryBreadcrumb = (
  message: string,
  category?: string,
  level?: Sentry.SeverityLevel,
  data?: Record<string, unknown>
): void => {
  Sentry.addBreadcrumb({
    message,
    category: category || 'app',
    level: level || 'info',
    data,
    timestamp: Date.now() / 1000,
  });
};

export const captureException = (
  error: Error,
  context?: Record<string, unknown>
): string => {
  return Sentry.captureException(error, {
    extra: context,
  });
};

export const captureMessage = (
  message: string,
  level: Sentry.SeverityLevel = 'info'
): string => {
  return Sentry.captureMessage(message, level);
};
```

### Sentry Performance Monitoring

```typescript
// src/crashReporting/sentryPerformance.ts
import * as Sentry from '@sentry/react-native';

// Transaction สำหรับติดตาม performance
export const startTransaction = (
  name: string,
  operation: string = 'custom'
): Sentry.TransactionType => {
  return Sentry.startTransaction({
    name,
    op: operation,
  });
};

// Span สำหรับติดตาม sub-operations
export const measureAsync = async <T>(
  operation: string,
  fn: () => Promise<T>,
  parentSpan?: Sentry.Span
): Promise<T> => {
  const span = parentSpan?.startChild({ op: operation }) || 
    Sentry.getCurrentHub().getScope()?.getTransaction()?.startChild({ op: operation });
  
  try {
    const result = await fn();
    span?.setStatus('ok');
    return result;
  } catch (error) {
    span?.setStatus('internal_error');
    throw error;
  } finally {
    span?.finish();
  }
};

// ตัวอย่าง: ติดตาม API call performance
export const trackedFetch = async (
  url: string,
  options?: RequestInit
): Promise<Response> => {
  const span = Sentry.getCurrentHub()
    .getScope()
    ?.getTransaction()
    ?.startChild({
      op: 'http.client',
      description: `${options?.method || 'GET'} ${url}`,
    });

  try {
    const response = await fetch(url, options);
    span?.setStatus(response.ok ? 'ok' : 'error');
    span?.setData('http.status_code', response.status);
    return response;
  } catch (error) {
    span?.setStatus('internal_error');
    throw error;
  } finally {
    span?.finish();
  }
};
```

---

## 4. Error Logging

### Centralized Error Logger

```typescript
// src/crashReporting/errorLogger.ts
import crashlytics from '@react-native-firebase/crashlytics';
import * as Sentry from '@sentry/react-native';
import { breadcrumbs } from './breadcrumbs';

type ErrorLevel = 'debug' | 'info' | 'warning' | 'error' | 'fatal';

interface ErrorLogOptions {
  level?: ErrorLevel;
  tags?: Record<string, string>;
  extra?: Record<string, unknown>;
  user?: { id: string };
}

class ErrorLogger {
  log(
    error: Error | string,
    options: ErrorLogOptions = {}
  ): void {
    const { level = 'error', tags, extra } = options;
    
    const errorObj = typeof error === 'string' ? new Error(error) : error;
    const message = errorObj.message;

    // Breadcrumb
    breadcrumbs.addError(message, JSON.stringify(extra));

    // Crashlytics
    crashlytics().log(`[${level.toUpperCase()}] ${message}`);
    if (level === 'fatal' || level === 'error') {
      crashlytics().recordError(errorObj);
    }

    // Sentry
    Sentry.withScope(scope => {
      scope.setLevel(level as Sentry.SeverityLevel);
      if (tags) scope.setTags(tags);
      if (extra) scope.setExtras(extra);
      
      Sentry.captureException(errorObj);
    });

    // Console in dev
    if (__DEV__) {
      const consoleMethod = level === 'warning' ? 'warn' : 'error';
      console[consoleMethod](`[ErrorLogger] ${message}`, errorObj, extra);
    }
  }

  debug(message: string, extra?: Record<string, unknown>): void {
    if (!__DEV__) return; // Debug เฉพาะ development
    console.log(`[DEBUG] ${message}`, extra);
  }

  info(message: string, extra?: Record<string, unknown>): void {
    this.log(message, { level: 'info', extra });
  }

  warning(message: string, extra?: Record<string, unknown>): void {
    this.log(message, { level: 'warning', extra });
  }

  error(error: Error | string, extra?: Record<string, unknown>): void {
    this.log(error, { level: 'error', extra });
  }

  fatal(error: Error | string, extra?: Record<string, unknown>): void {
    this.log(error, { level: 'fatal', extra });
  }
}

export const logger = new ErrorLogger();
```

### API Error Handling

```typescript
// src/api/apiClient.ts
import axios, { AxiosInstance, AxiosError, AxiosResponse } from 'axios';
import { logger } from '../crashReporting/errorLogger';
import { breadcrumbs } from '../crashReporting/breadcrumbs';

const createApiClient = (baseURL: string): AxiosInstance => {
  const client = axios.create({
    baseURL,
    timeout: 30000,
    headers: {
      'Content-Type': 'application/json',
    },
  });

  // Request interceptor
  client.interceptors.request.use(
    config => {
      breadcrumbs.addNetworkRequest(
        config.url || '',
        config.method?.toUpperCase() || 'GET',
        0
      );
      return config;
    },
    error => {
      logger.error(error, { context: 'Request interceptor' });
      return Promise.reject(error);
    }
  );

  // Response interceptor
  client.interceptors.response.use(
    (response: AxiosResponse) => {
      breadcrumbs.addNetworkRequest(
        response.config.url || '',
        response.config.method?.toUpperCase() || 'GET',
        response.status
      );
      return response;
    },
    (error: AxiosError) => {
      const statusCode = error.response?.status;
      const url = error.config?.url || 'unknown';
      
      breadcrumbs.addNetworkRequest(url, 'HTTP', statusCode || 0);
      
      if (statusCode) {
        if (statusCode >= 500) {
          logger.error(error, {
            context: 'Server Error',
            url,
            statusCode,
          });
        } else if (statusCode === 401) {
          logger.info('Unauthorized request', { url });
        } else if (statusCode === 404) {
          logger.warning('Resource not found', { url });
        }
      } else if (error.code === 'ECONNABORTED') {
        logger.warning('Request timeout', { url });
      } else {
        logger.error(error, { context: 'Network Error', url });
      }
      
      return Promise.reject(error);
    }
  );

  return client;
};

export default createApiClient;
```

---

## Workshop: Crash Reporting Setup

### Complete Setup

```typescript
// src/crashReporting/setup.ts
import { setupErrorHandlers } from './errorHandler';
import CrashlyticsService from './crashlytics';
import { initializeSentry } from './sentry';
import { sessionTracker } from '../analytics/session';

export const initializeCrashReporting = async (
  userId?: string,
  userProperties?: Record<string, string>
): Promise<void> => {
  // Initialize services
  await CrashlyticsService.initialize();
  initializeSentry();
  
  // Setup global error handlers
  setupErrorHandlers();
  
  // Set user if available
  if (userId) {
    CrashlyticsService.setUserId(userId);
    
    if (userProperties) {
      CrashlyticsService.setAttributes(userProperties);
    }
  }
  
  // Start session tracking
  await sessionTracker.startSession();
  
  console.log('Crash reporting initialized');
};
```

### Test Crash (สำหรับ Development)

```typescript
// ใน Development settings screen
import crashlytics from '@react-native-firebase/crashlytics';
import * as Sentry from '@sentry/react-native';

const TestCrashButtons: React.FC = () => {
  if (!__DEV__) return null;
  
  return (
    <View>
      <TouchableOpacity onPress={() => crashlytics().crash()}>
        <Text>Test Firebase Crash</Text>
      </TouchableOpacity>
      
      <TouchableOpacity onPress={() => {
        throw new Error('Test Sentry Error');
      }}>
        <Text>Test Sentry Error</Text>
      </TouchableOpacity>
      
      <TouchableOpacity onPress={() => {
        Sentry.nativeCrash();
      }}>
        <Text>Test Sentry Native Crash</Text>
      </TouchableOpacity>
    </View>
  );
};
```

---

## Tips และ Best Practices

### 1. ไม่เก็บข้อมูลส่วนตัว

```typescript
// ลบ PII ก่อน log
const sanitizeError = (error: Error): Error => {
  const sanitizedMessage = error.message
    .replace(/email=[\w@.]+/gi, 'email=***')
    .replace(/password=\S+/gi, 'password=***')
    .replace(/token=\S+/gi, 'token=***');
    
  const sanitizedError = new Error(sanitizedMessage);
  sanitizedError.stack = error.stack;
  return sanitizedError;
};
```

### 2. Error Grouping

```typescript
// เพิ่ม fingerprint เพื่อ group errors
Sentry.withScope(scope => {
  scope.setFingerprint(['api-error', endpoint, String(statusCode)]);
  Sentry.captureException(error);
});
```

### 3. Alert on Critical Errors

```yaml
# Sentry Alert Rule
When:
  - event.level equals fatal
  - frequency > 10 in 1 minute

Then:
  - Send Slack notification
  - Send email to team
  - Create PagerDuty incident
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Firebase Crashlytics**: Setup และ crash tracking
2. **Error Boundary**: ดัก React errors
3. **Breadcrumbs**: ติดตาม events ก่อน crash
4. **Sentry**: Performance monitoring และ error tracking
5. **Error Logger**: Centralized logging system
6. **Best Practices**: Privacy, grouping, alerting
