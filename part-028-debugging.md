# Part 028: Debugging เบื้องต้น

## บทนำ

Debugging คือทักษะที่สำคัญที่สุดทักษะหนึ่งของนักพัฒนา ไม่ว่า code จะเขียนดีแค่ไหน ก็ยังมีโอกาสเจอ bug เสมอ ในบทนี้เราจะเรียนรู้เครื่องมือและเทคนิคต่างๆ ในการ debug React Native app

## สารบัญ

1. Developer Menu
2. Chrome DevTools
3. React Native Debugger
4. Flipper
5. Console.log Strategies
6. Error Boundaries
7. Workshop: Debug a Broken App

---

## 1. Developer Menu

Developer Menu เป็นจุดเริ่มต้นของการ debug ใน React Native

### วิธีเปิด Developer Menu

| Platform | วิธี |
|----------|------|
| iOS Simulator | Cmd + D หรือ Device > Shake |
| Android Emulator | Cmd + M (Mac) / Ctrl + M (Win) |
| Real Device (iOS) | เขย่าอุปกรณ์ |
| Real Device (Android) | เขย่าอุปกรณ์ |

### เมนูที่สำคัญ

```
Developer Menu
├── Reload                    # รีโหลด JS bundle
├── Open Debugger             # เปิด Chrome DevTools
├── Toggle Element Inspector  # ตรวจสอบ component tree
├── Performance Monitor       # ดู FPS, JS thread
├── Network Inspector         # ดู network requests
└── Settings                  # ตั้งค่า developer options
```

### Reload Methods

```javascript
// Reload ด้วย keyboard shortcuts
// iOS Simulator: Cmd + R
// Android Emulator: R + R

// Reload ด้วย code
import { DevSettings } from 'react-native';
DevSettings.reload();

// หรือใน terminal
// iOS: press 'i' ใน Metro terminal
// Android: press 'a' ใน Metro terminal
```

---

## 2. Chrome DevTools

### เปิด Chrome Debugger

1. เปิด Developer Menu
2. เลือก "Open Debugger" หรือ "Debug with Chrome"
3. Chrome จะเปิดหน้าต่างใหม่ที่ `localhost:8081/debugger-ui`
4. เปิด DevTools (Cmd + Option + I / F12)

### Console Tab

```javascript
// ✅ แสดงข้อมูลง่ายๆ
console.log('ข้อมูลทั่วไป', someValue);
console.warn('คำเตือน', warningMessage);
console.error('ข้อผิดพลาด', error);

// ✅ จัดกลุ่ม logs
console.group('User Actions');
console.log('เข้าสู่ระบบ:', user.name);
console.log('เวลา:', new Date().toLocaleString());
console.groupEnd();

// ✅ ดู object แบบละเอียด
console.dir(complexObject);

// ✅ แสดงเป็นตาราง
console.table([
  { id: 1, name: 'สินค้า A', price: 100 },
  { id: 2, name: 'สินค้า B', price: 200 },
]);

// ✅ วัดเวลา
console.time('dataLoad');
const data = await fetchData();
console.timeEnd('dataLoad');  // "dataLoad: 1234ms"

// ✅ Count
console.count('renderCount');  // "renderCount: 1", "renderCount: 2", ...

// ✅ Assert
console.assert(user.age >= 18, 'User must be 18 or older', user);

// ✅ Stack trace
console.trace('Where was this called?');
```

### Sources Tab (Breakpoints)

```javascript
// วิธีที่ 1: ใช้ debugger statement ใน code
const handleLogin = async (email, password) => {
  debugger;  // หยุดที่นี่
  
  try {
    const result = await loginAPI(email, password);
    debugger;  // หยุดหลัง API call
    navigation.navigate('Home');
  } catch (error) {
    debugger;  // หยุดเมื่อเกิด error
    console.error('Login failed:', error);
  }
};

// วิธีที่ 2: คลิก line number ใน Sources tab
// วิธีที่ 3: Conditional breakpoints (คลิกขวาที่ line number)
```

### Network Tab

```javascript
// ใช้ Chrome DevTools Network tab เพื่อดู:
// - API requests และ responses
// - Headers ที่ส่ง/รับ
// - Timing ของแต่ละ request
// - Payload/Body ของ request

// ✅ Log network requests ด้วย interceptor
import axios from 'axios';

const instance = axios.create({ baseURL: 'https://api.example.com' });

// Request interceptor
instance.interceptors.request.use((config) => {
  console.group(`📤 ${config.method?.toUpperCase()} ${config.url}`);
  console.log('Headers:', config.headers);
  console.log('Data:', config.data);
  console.groupEnd();
  return config;
});

// Response interceptor
instance.interceptors.response.use(
  (response) => {
    console.group(`📥 ${response.status} ${response.config.url}`);
    console.log('Data:', response.data);
    console.groupEnd();
    return response;
  },
  (error) => {
    console.group(`❌ Error ${error.response?.status} ${error.config?.url}`);
    console.log('Error:', error.response?.data);
    console.groupEnd();
    return Promise.reject(error);
  }
);
```

---

## 3. React Native Debugger

React Native Debugger เป็น standalone app ที่ใช้ debug ได้ดีกว่า Chrome DevTools

### ติดตั้ง

```bash
# Mac
brew install --cask react-native-debugger

# หรือ download จาก
# https://github.com/jhen0409/react-native-debugger/releases
```

### Features ของ RN Debugger

1. **React DevTools** - ดู component tree, props, state
2. **Redux DevTools** - ดู Redux actions และ state
3. **Network Inspector** - ดู API calls
4. **Console** - Chrome DevTools console
5. **Breakpoints** - Source-level debugging

### ใช้กับ Redux

```javascript
import { createStore } from 'redux';

// เปิด Redux DevTools
const store = createStore(
  rootReducer,
  window.__REDUX_DEVTOOLS_EXTENSION__ && window.__REDUX_DEVTOOLS_EXTENSION__()
);
```

### React Query DevTools

```bash
npm install @tanstack/react-query-devtools
```

```jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,
    },
  },
});

const App = () => (
  <QueryClientProvider client={queryClient}>
    <MainApp />
    {__DEV__ && <ReactQueryDevtools initialIsOpen={false} />}
  </QueryClientProvider>
);
```

---

## 4. Flipper

Flipper เป็น powerful debugging platform จาก Facebook

### ติดตั้ง

```bash
# Download Flipper
# https://fbflipper.com/

# ติดตั้ง plugin ที่จำเป็น
# - React DevTools
# - Network
# - Databases
# - Crash Reporter
# - Logs
```

### Flipper Plugins ที่มีประโยชน์

```
Flipper Plugins
├── Layout        # Component tree + CSS
├── Network       # HTTP requests
├── Databases     # SQLite, MMKV, AsyncStorage
├── React DevTools# Component inspection
├── Crash Reporter# Crash logs
├── Logs          # Device logs
├── Hermes        # JS engine debugger
└── Redux         # Redux DevTools
```

### Custom Flipper Plugin

```javascript
import { addPlugin } from 'react-native-flipper';

// สร้าง custom plugin
addPlugin({
  getId() {
    return 'MyCustomPlugin';
  },
  onConnect(connection) {
    // ส่งข้อมูลไปยัง Flipper UI
    connection.send('newData', {
      data: 'Hello from React Native!',
      timestamp: Date.now(),
    });

    // รับ messages จาก Flipper UI
    connection.receive('customEvent', (data, respond) => {
      console.log('Received from Flipper:', data);
      respond({ success: true });
    });
  },
  onDisconnect() {
    console.log('Flipper disconnected');
  },
  runInBackground() {
    return false;
  },
});
```

### Network Inspection กับ Flipper

```javascript
import { NetworkPlugin } from 'react-native-flipper';

// Setup network inspection
NetworkPlugin.addNetworkInterceptor((request, response) => {
  console.log(`[Network] ${request.method} ${request.url}`);
  console.log('Response status:', response.status);
});
```

---

## 5. Console.log Strategies

### Custom Logger

```javascript
// utils/logger.js

const LOG_LEVELS = {
  DEBUG: 0,
  INFO: 1,
  WARN: 2,
  ERROR: 3,
  NONE: 4,
};

const LOG_LEVEL = __DEV__ ? LOG_LEVELS.DEBUG : LOG_LEVELS.ERROR;

const COLORS = {
  DEBUG: '#888',
  INFO: '#2196F3',
  WARN: '#FF9800',
  ERROR: '#F44336',
  SUCCESS: '#4CAF50',
};

class Logger {
  constructor(namespace) {
    this.namespace = namespace;
  }

  _shouldLog(level) {
    return LOG_LEVEL <= level;
  }

  _format(level, message, ...args) {
    const timestamp = new Date().toISOString().split('T')[1].split('.')[0];
    return [`[${timestamp}] [${level}] [${this.namespace}] ${message}`, ...args];
  }

  debug(message, ...args) {
    if (!this._shouldLog(LOG_LEVELS.DEBUG)) return;
    console.log(...this._format('DEBUG', message, ...args));
  }

  info(message, ...args) {
    if (!this._shouldLog(LOG_LEVELS.INFO)) return;
    console.info(...this._format('INFO', message, ...args));
  }

  warn(message, ...args) {
    if (!this._shouldLog(LOG_LEVELS.WARN)) return;
    console.warn(...this._format('WARN', message, ...args));
  }

  error(message, error, ...args) {
    if (!this._shouldLog(LOG_LEVELS.ERROR)) return;
    console.error(...this._format('ERROR', message, error, ...args));
    // ส่งไปยัง crash reporting service
    if (!__DEV__) {
      Sentry.captureException(error, { extra: { message, args } });
    }
  }

  success(message, ...args) {
    if (!this._shouldLog(LOG_LEVELS.DEBUG)) return;
    console.log(...this._format('SUCCESS ✓', message, ...args));
  }

  // Log API calls
  api(method, url, data) {
    this.debug(`API ${method.toUpperCase()} ${url}`, data);
  }

  // Performance measurement
  time(label) {
    if (__DEV__) console.time(`[${this.namespace}] ${label}`);
  }

  timeEnd(label) {
    if (__DEV__) console.timeEnd(`[${this.namespace}] ${label}`);
  }
}

// Factory function
export const createLogger = (namespace) => new Logger(namespace);

// ตัวอย่างการใช้งาน
// const logger = createLogger('AuthService');
// logger.info('เข้าสู่ระบบสำเร็จ', { userId: 123 });
// logger.error('เข้าสู่ระบบล้มเหลว', error, { email });
```

### Pretty Print Objects

```javascript
// utils/debugUtils.js

export const prettyLog = (label, value) => {
  if (!__DEV__) return;
  
  console.group(`📦 ${label}`);
  try {
    console.log(JSON.stringify(value, null, 2));
  } catch {
    console.log(value);
  }
  console.groupEnd();
};

export const logRender = (componentName, props) => {
  if (!__DEV__) return;
  console.log(`🔄 Render: ${componentName}`, props ? JSON.stringify(props) : '');
};

export const logStateChange = (stateName, prev, next) => {
  if (!__DEV__) return;
  console.group(`🔀 State Change: ${stateName}`);
  console.log('Before:', JSON.stringify(prev, null, 2));
  console.log('After:', JSON.stringify(next, null, 2));
  console.groupEnd();
};

export const logPerformance = (label) => {
  if (!__DEV__) return;
  
  const startTime = Date.now();
  return {
    end: () => {
      const duration = Date.now() - startTime;
      console.log(`⏱ ${label}: ${duration}ms`);
    },
  };
};

// ใช้งาน
const { end } = logPerformance('fetchUsers');
const users = await fetchUsers();
end(); // "⏱ fetchUsers: 234ms"
```

### Debug Hook

```javascript
// hooks/useDebugValue.js
import { useEffect, useRef } from 'react';

// Hook สำหรับ debug props/state changes
export const useDebugChanges = (values, name = 'Component') => {
  const prevValues = useRef(values);

  useEffect(() => {
    if (!__DEV__) return;

    const changes = Object.entries(values).reduce((acc, [key, value]) => {
      if (prevValues.current[key] !== value) {
        acc[key] = { from: prevValues.current[key], to: value };
      }
      return acc;
    }, {});

    if (Object.keys(changes).length > 0) {
      console.group(`🔄 ${name} changed`);
      Object.entries(changes).forEach(([key, { from, to }]) => {
        console.log(`  ${key}: ${JSON.stringify(from)} → ${JSON.stringify(to)}`);
      });
      console.groupEnd();
    }

    prevValues.current = values;
  });
};

// useWhyDidYouUpdate - ดูว่า prop ไหนทำให้ re-render
export const useWhyDidYouUpdate = (name, props) => {
  const previousProps = useRef({});

  useEffect(() => {
    if (!__DEV__) return;

    if (previousProps.current) {
      const allKeys = Object.keys({ ...previousProps.current, ...props });
      const changedProps = {};

      allKeys.forEach((key) => {
        if (previousProps.current[key] !== props[key]) {
          changedProps[key] = {
            from: previousProps.current[key],
            to: props[key],
          };
        }
      });

      if (Object.keys(changedProps).length) {
        console.log('[why-did-you-update]', name, changedProps);
      }
    }

    previousProps.current = props;
  });
};

// ใช้งาน
const MyComponent = (props) => {
  useWhyDidYouUpdate('MyComponent', props);
  // ...
};
```

---

## 6. Error Boundaries

Error Boundaries ช่วยจับ runtime errors และแสดง fallback UI

```jsx
import React from 'react';
import { View, Text, TouchableOpacity, ScrollView, StyleSheet } from 'react-native';

// Class Component สำหรับ Error Boundary
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null,
    };
  }

  // เรียกเมื่อเกิด error ใน subtree
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // เรียกหลัง render เพื่อ log error
  componentDidCatch(error, errorInfo) {
    console.error('ErrorBoundary caught an error:', error, errorInfo);
    this.setState({ errorInfo });

    // ส่งไปยัง crash reporting
    if (!__DEV__) {
      // Sentry.captureException(error, { extra: errorInfo });
    }
  }

  handleReset = () => {
    this.setState({ hasError: false, error: null, errorInfo: null });
    this.props.onReset?.();
  };

  render() {
    if (this.state.hasError) {
      // Custom fallback UI
      if (this.props.fallback) {
        return this.props.fallback(this.state.error, this.handleReset);
      }

      return (
        <View style={errStyles.container}>
          <Text style={errStyles.emoji}>😔</Text>
          <Text style={errStyles.title}>เกิดข้อผิดพลาด</Text>
          <Text style={errStyles.message}>
            เกิดข้อผิดพลาดที่ไม่คาดคิด กรุณาลองใหม่อีกครั้ง
          </Text>

          {__DEV__ && (
            <ScrollView style={errStyles.details}>
              <Text style={errStyles.errorText}>
                {this.state.error?.toString()}
              </Text>
              <Text style={errStyles.stackText}>
                {this.state.errorInfo?.componentStack}
              </Text>
            </ScrollView>
          )}

          <TouchableOpacity style={errStyles.button} onPress={this.handleReset}>
            <Text style={errStyles.buttonText}>ลองใหม่</Text>
          </TouchableOpacity>
        </View>
      );
    }

    return this.props.children;
  }
}

const errStyles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
    backgroundColor: '#fff',
  },
  emoji: { fontSize: 64, marginBottom: 16 },
  title: { fontSize: 24, fontWeight: 'bold', color: '#1a1a1a', marginBottom: 8 },
  message: { fontSize: 16, color: '#666', textAlign: 'center', marginBottom: 24, lineHeight: 24 },
  details: {
    backgroundColor: '#f5f5f5',
    borderRadius: 8,
    padding: 12,
    maxHeight: 200,
    width: '100%',
    marginBottom: 24,
  },
  errorText: { fontFamily: 'monospace', fontSize: 12, color: '#FF3B30' },
  stackText: { fontFamily: 'monospace', fontSize: 11, color: '#666', marginTop: 8 },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 32,
    paddingVertical: 14,
    borderRadius: 12,
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});

// ============================================
// Nested Error Boundaries Pattern
// ============================================

// App Level - จับ errors ทั้งหมด
const AppErrorBoundary = ({ children }) => (
  <ErrorBoundary
    fallback={(error, reset) => (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <Text>App crashed: {error?.message}</Text>
        <TouchableOpacity onPress={reset}>
          <Text>Restart App</Text>
        </TouchableOpacity>
      </View>
    )}
  >
    {children}
  </ErrorBoundary>
);

// Screen Level - จับ errors เฉพาะหน้า
const ScreenErrorBoundary = ({ children, screenName }) => (
  <ErrorBoundary
    fallback={(error, reset) => (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center', padding: 20 }}>
        <Text style={{ fontSize: 18, fontWeight: 'bold', marginBottom: 8 }}>
          {screenName} เกิดข้อผิดพลาด
        </Text>
        <Text style={{ color: '#666', marginBottom: 16 }}>
          {__DEV__ ? error?.message : 'กรุณาลองใหม่'}
        </Text>
        <TouchableOpacity
          style={{ backgroundColor: '#007AFF', padding: 14, borderRadius: 10 }}
          onPress={reset}
        >
          <Text style={{ color: '#fff' }}>รีโหลดหน้านี้</Text>
        </TouchableOpacity>
      </View>
    )}
  >
    {children}
  </ErrorBoundary>
);

// Component Level - จับ errors เฉพาะ component
const ComponentErrorBoundary = ({ children, name = 'Component' }) => (
  <ErrorBoundary
    fallback={() => (
      <View style={{ padding: 8, backgroundColor: '#FFF3E0', borderRadius: 8 }}>
        <Text style={{ color: '#E65100', fontSize: 12 }}>
          ⚠️ {name} ไม่สามารถแสดงผลได้
        </Text>
      </View>
    )}
  >
    {children}
  </ErrorBoundary>
);

// ============================================
// Hook-based Error Handling
// ============================================

// Custom hook สำหรับ async error handling
const useAsyncError = () => {
  const [error, setError] = useState(null);

  if (error) throw error;  // Re-throw ให้ ErrorBoundary จับ

  const handleError = useCallback((err) => {
    setError(err);
  }, []);

  return handleError;
};

// ใช้งาน
const MyComponent = () => {
  const throwError = useAsyncError();

  const fetchData = async () => {
    try {
      await someAsyncOperation();
    } catch (error) {
      throwError(error); // จะถูก ErrorBoundary จับ
    }
  };

  return <View />;
};
```

---

## 7. Workshop: Debug a Broken App

```jsx
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  ActivityIndicator,
  StyleSheet,
  Alert,
} from 'react-native';

// ============================================
// 🐛 Buggy Version (ก่อน debug)
// ============================================

/*
const BuggyApp = () => {
  const [users, setUsers] = useState([]);
  
  // Bug 1: Missing dependency ใน useEffect
  useEffect(() => {
    fetchUsers(userId);  // userId ไม่ได้อยู่ใน dependency array!
  }, []);

  // Bug 2: ไม่มี loading state
  const fetchUsers = async (id) => {
    const response = await fetch(`/api/users/${id}`);
    const data = response.json();  // Bug 3: Missing await!
    setUsers(data.users);          // Bug 4: data อาจเป็น undefined
  };

  // Bug 5: ไม่ handle error
  return (
    <FlatList
      data={users}
      renderItem={({ item }) => (
        <Text>{item.name}</Text>      // Bug 6: item อาจเป็น null
      )}
    />
  );
};
*/

// ============================================
// ✅ Fixed Version (หลัง debug)
// ============================================

// Step 1: Debug utilities
const DEBUG = {
  log: (msg, data) => __DEV__ && console.log(`[App] ${msg}`, data ?? ''),
  warn: (msg, data) => __DEV__ && console.warn(`[App] ⚠️ ${msg}`, data ?? ''),
  error: (msg, err) => {
    console.error(`[App] ❌ ${msg}`, err);
    if (!__DEV__) {
      // Report to Sentry
    }
  },
};

// Step 2: Type-safe API call
const fetchUsersFromAPI = async (page = 1) => {
  DEBUG.log('fetchUsers start', { page });

  const response = await fetch(`https://jsonplaceholder.typicode.com/users?_page=${page}&_limit=5`);

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const data = await response.json();  // ✅ ใช้ await
  DEBUG.log('fetchUsers success', { count: data.length });
  return data;
};

// Step 3: Proper component
const DebuggedApp = () => {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  // ✅ Debug render count
  const renderCount = useRef(0);
  renderCount.current++;
  DEBUG.log(`Render #${renderCount.current}`, { usersCount: users.length, loading, error: error?.message });

  const loadUsers = useCallback(async (pageNum = 1, reset = false) => {
    if (loading) {
      DEBUG.warn('Already loading, skipping...');
      return;
    }

    setLoading(true);
    setError(null);

    try {
      const newUsers = await fetchUsersFromAPI(pageNum);

      if (newUsers.length === 0) {
        setHasMore(false);
        return;
      }

      setUsers((prev) => reset ? newUsers : [...prev, ...newUsers]);
      setPage(pageNum);

      DEBUG.log('Users loaded', { total: reset ? newUsers.length : users.length + newUsers.length });
    } catch (err) {
      DEBUG.error('Failed to load users', err);
      setError(err);
      Alert.alert(
        '❌ โหลดข้อมูลไม่สำเร็จ',
        `${err.message}\n\nกรุณาตรวจสอบการเชื่อมต่ออินเทอร์เน็ต`,
        [
          { text: 'ยกเลิก', style: 'cancel' },
          { text: 'ลองใหม่', onPress: () => loadUsers(pageNum, reset) },
        ]
      );
    } finally {
      setLoading(false);
    }
  }, [loading, users.length]);

  // ✅ Proper dependency array
  useEffect(() => {
    DEBUG.log('useEffect mount - loading initial users');
    loadUsers(1, true);

    return () => {
      DEBUG.log('useEffect cleanup');
    };
  }, []); // eslint-disable-line react-hooks/exhaustive-deps

  const handleLoadMore = useCallback(() => {
    if (!loading && hasMore) {
      loadUsers(page + 1);
    }
  }, [loading, hasMore, page, loadUsers]);

  const renderUser = useCallback(({ item, index }) => {
    // ✅ Guard against null/undefined
    if (!item) {
      DEBUG.warn('Received null item at index', index);
      return null;
    }

    return (
      <UserCard user={item} />
    );
  }, []);

  const keyExtractor = useCallback((item) => {
    // ✅ Ensure unique key
    if (!item?.id) {
      DEBUG.warn('User without id:', item);
      return Math.random().toString();
    }
    return String(item.id);
  }, []);

  // ✅ Error state
  if (error && users.length === 0) {
    return (
      <View style={debugStyles.errorState}>
        <Text style={debugStyles.errorIcon}>😔</Text>
        <Text style={debugStyles.errorTitle}>โหลดข้อมูลไม่สำเร็จ</Text>
        <Text style={debugStyles.errorMsg}>{error.message}</Text>
        <TouchableOpacity
          style={debugStyles.retryBtn}
          onPress={() => loadUsers(1, true)}
        >
          <Text style={debugStyles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={{ flex: 1, backgroundColor: '#f5f5f5' }}>
      {/* Debug Panel (DEV only) */}
      {__DEV__ && (
        <View style={debugStyles.devPanel}>
          <Text style={debugStyles.devText}>
            🔧 DEV | renders: {renderCount.current} | users: {users.length} | page: {page}
          </Text>
          <TouchableOpacity onPress={() => loadUsers(1, true)}>
            <Text style={debugStyles.devRefresh}>↺</Text>
          </TouchableOpacity>
        </View>
      )}

      <FlatList
        data={users}
        renderItem={renderUser}
        keyExtractor={keyExtractor}
        onEndReached={handleLoadMore}
        onEndReachedThreshold={0.5}
        contentContainerStyle={{ padding: 16, gap: 12, paddingBottom: 32 }}
        refreshing={loading && page === 1}
        onRefresh={() => loadUsers(1, true)}
        ListFooterComponent={() => (
          loading && page > 1 ? (
            <View style={debugStyles.loadingMore}>
              <ActivityIndicator color="#007AFF" />
              <Text style={{ color: '#666', marginLeft: 8 }}>กำลังโหลดเพิ่มเติม...</Text>
            </View>
          ) : null
        )}
        ListEmptyComponent={() => (
          !loading ? (
            <View style={debugStyles.empty}>
              <Text style={{ fontSize: 48 }}>📭</Text>
              <Text style={{ color: '#666', marginTop: 12 }}>ไม่มีข้อมูล</Text>
            </View>
          ) : null
        )}
      />

      {/* Initial Loading */}
      {loading && page === 1 && users.length === 0 && (
        <View style={debugStyles.loadingOverlay}>
          <ActivityIndicator size="large" color="#007AFF" />
          <Text style={{ color: '#666', marginTop: 12 }}>กำลังโหลด...</Text>
        </View>
      )}
    </View>
  );
};

// User Card Component
const UserCard = React.memo(({ user }) => {
  // ✅ Defense against missing fields
  const name = user?.name ?? 'Unknown User';
  const email = user?.email ?? 'No email';
  const company = user?.company?.name ?? 'No company';

  return (
    <View style={debugStyles.card}>
      <View style={debugStyles.avatar}>
        <Text style={debugStyles.avatarText}>
          {name.charAt(0).toUpperCase()}
        </Text>
      </View>
      <View style={{ flex: 1 }}>
        <Text style={debugStyles.name}>{name}</Text>
        <Text style={debugStyles.email}>{email}</Text>
        <Text style={debugStyles.company}>{company}</Text>
      </View>
    </View>
  );
});

const debugStyles = StyleSheet.create({
  card: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    gap: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
  },
  avatar: {
    width: 50,
    height: 50,
    borderRadius: 25,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
  avatarText: { color: '#fff', fontSize: 20, fontWeight: 'bold' },
  name: { fontSize: 16, fontWeight: '600', color: '#1a1a1a' },
  email: { fontSize: 13, color: '#666', marginTop: 2 },
  company: { fontSize: 12, color: '#999', marginTop: 1 },
  devPanel: {
    backgroundColor: '#1a1a1a',
    paddingVertical: 4,
    paddingHorizontal: 16,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  devText: { color: '#00FF00', fontSize: 10, fontFamily: 'monospace' },
  devRefresh: { color: '#00FF00', fontSize: 18 },
  errorState: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
  },
  errorIcon: { fontSize: 64, marginBottom: 16 },
  errorTitle: { fontSize: 20, fontWeight: 'bold', marginBottom: 8 },
  errorMsg: { color: '#666', textAlign: 'center', marginBottom: 24 },
  retryBtn: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 32,
    paddingVertical: 14,
    borderRadius: 12,
  },
  retryText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  loadingMore: {
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  empty: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 40,
  },
  loadingOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(255,255,255,0.8)',
    justifyContent: 'center',
    alignItems: 'center',
  },
});

export default DebuggedApp;
```

---

## Tips และ Best Practices

### 1. Debug vs Production
```javascript
// ✅ ใช้ __DEV__ guard
if (__DEV__) {
  console.log('debug info');
}

// ✅ หรือปิด console ใน production
if (!__DEV__) {
  console.log = () => {};
  console.warn = () => {};
  console.error = () => {};
}
```

### 2. Meaningful Error Messages
```javascript
// ❌ ไม่มีประโยชน์
throw new Error('Error');

// ✅ มีข้อมูลครบ
throw new Error(`Login failed: ${response.status} - ${errorMessage} (userId: ${userId})`);
```

### 3. Performance Debugging
```javascript
// ตรวจสอบ re-renders ที่ไม่จำเป็น
const MyComponent = React.memo(({ value }) => {
  // ✅ Memoize expensive calculations
  const result = useMemo(() => expensiveCalc(value), [value]);
  
  // ✅ Memoize callbacks
  const handlePress = useCallback(() => {
    doSomething(value);
  }, [value]);

  return <View />;
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Developer Menu และ reload methods
- Chrome DevTools สำหรับ console และ breakpoints
- React Native Debugger สำหรับ Redux
- Flipper สำหรับ advanced debugging
- Console.log strategies และ custom logger
- Error Boundaries สำหรับ graceful error handling
- Workshop: Debug และ fix broken app

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Testing เบื้องต้น
