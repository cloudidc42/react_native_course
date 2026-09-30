# Part 019: Custom Hooks

## สารบัญ
1. สร้าง Custom Hook
2. useFetch Hook
3. useLocalStorage Hook
4. useDebounce Hook
5. useWindowDimensions Hook
6. Workshop: สร้าง Hooks Library

---

## 1. สร้าง Custom Hook

Custom Hook คือ JavaScript function ที่:
- ชื่อขึ้นต้นด้วย `use`
- สามารถใช้ React Hooks อื่นๆ ภายในได้
- ช่วย reuse stateful logic ระหว่าง components

### ทำไมต้องสร้าง Custom Hooks

```javascript
// ❌ ก่อนใช้ Custom Hook - logic ซ้ำในหลาย component
function Screen1() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [data, setData] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetch('/api/data1')
      .then(r => r.json())
      .then(d => { setData(d); setLoading(false); })
      .catch(e => { setError(e); setLoading(false); });
  }, []);
  // ... ซ้ำใน Screen2, Screen3, ...
}

// ✅ หลังใช้ Custom Hook - logic ใน Hook เดียว
function useFetch(url) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [data, setData] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(r => r.json())
      .then(d => { setData(d); setLoading(false); })
      .catch(e => { setError(e); setLoading(false); });
  }, [url]);
  
  return { loading, error, data };
}

function Screen1() {
  const { loading, error, data } = useFetch('/api/data1');
  // ...
}
```

### กฎของ Custom Hooks

```javascript
// ✅ ถูกต้อง: ชื่อขึ้นต้นด้วย use
function useCounter() { ... }
function useAuth() { ... }
function useFetch() { ... }

// ❌ ผิด: ไม่ขึ้นต้นด้วย use
function fetchData() { ... }  // ไม่ใช่ Hook!
function counterLogic() { ... }  // ไม่ใช่ Hook!

// Custom Hook แต่ละ instance มี state แยกกัน
function App() {
  const counter1 = useCounter(0);  // state แยกกัน
  const counter2 = useCounter(10); // state แยกกัน
  
  return (
    <View>
      <Text>{counter1.count}</Text>  // 0
      <Text>{counter2.count}</Text>  // 10
    </View>
  );
}
```

---

## 2. useFetch Hook

```javascript
// hooks/useFetch.js
import { useState, useEffect, useCallback, useRef } from 'react';

function useFetch(url, options = {}) {
  const [state, setState] = useState({
    data: null,
    loading: false,
    error: null,
  });
  
  const [trigger, setTrigger] = useState(0);
  const optionsRef = useRef(options);
  optionsRef.current = options;

  const refetch = useCallback(() => {
    setTrigger(t => t + 1);
  }, []);

  useEffect(() => {
    if (!url) {
      setState({ data: null, loading: false, error: null });
      return;
    }

    let isMounted = true;
    const controller = new AbortController();

    setState(prev => ({ ...prev, loading: true, error: null }));

    const fetchData = async () => {
      try {
        const response = await fetch(url, {
          ...optionsRef.current,
          signal: controller.signal,
        });

        if (!response.ok) {
          throw new Error(`HTTP Error: ${response.status} ${response.statusText}`);
        }

        const contentType = response.headers.get('content-type');
        const data = contentType?.includes('application/json')
          ? await response.json()
          : await response.text();

        if (isMounted) {
          setState({ data, loading: false, error: null });
        }
      } catch (error) {
        if (isMounted && error.name !== 'AbortError') {
          setState(prev => ({ ...prev, loading: false, error: error.message }));
        }
      }
    };

    fetchData();

    return () => {
      isMounted = false;
      controller.abort();
    };
  }, [url, trigger]);

  return { ...state, refetch };
}

export default useFetch;
```

### ตัวอย่างการใช้ useFetch

```javascript
// ใช้งานพื้นฐาน
function UserList() {
  const { data: users, loading, error, refetch } = useFetch(
    'https://jsonplaceholder.typicode.com/users'
  );

  if (loading) return <ActivityIndicator />;
  if (error) return (
    <View>
      <Text>Error: {error}</Text>
      <Button title="ลองใหม่" onPress={refetch} />
    </View>
  );

  return (
    <FlatList
      data={users}
      keyExtractor={item => item.id.toString()}
      renderItem={({ item }) => <Text>{item.name}</Text>}
    />
  );
}

// Dynamic URL
function PostsByUser({ userId }) {
  const { data, loading } = useFetch(
    userId ? `https://jsonplaceholder.typicode.com/posts?userId=${userId}` : null
  );
  
  if (!userId) return <Text>เลือก user ก่อน</Text>;
  if (loading) return <ActivityIndicator />;
  
  return <Text>{data?.length} โพสต์</Text>;
}
```

### useFetch พร้อม POST/PUT/DELETE

```javascript
// hooks/useApi.js
import { useState, useCallback } from 'react';

function useApi(baseUrl) {
  const [state, setState] = useState({
    loading: false,
    error: null,
    data: null,
  });

  const request = useCallback(async (endpoint, method = 'GET', body = null) => {
    setState(prev => ({ ...prev, loading: true, error: null }));

    try {
      const response = await fetch(`${baseUrl}${endpoint}`, {
        method,
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
        body: body ? JSON.stringify(body) : undefined,
      });

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new Error(errorData.message || `HTTP ${response.status}`);
      }

      const data = response.status === 204 ? null : await response.json();
      setState({ loading: false, error: null, data });
      return data;
    } catch (error) {
      setState(prev => ({ ...prev, loading: false, error: error.message }));
      throw error;
    }
  }, [baseUrl]);

  const get = useCallback((endpoint) => request(endpoint, 'GET'), [request]);
  const post = useCallback((endpoint, body) => request(endpoint, 'POST', body), [request]);
  const put = useCallback((endpoint, body) => request(endpoint, 'PUT', body), [request]);
  const patch = useCallback((endpoint, body) => request(endpoint, 'PATCH', body), [request]);
  const del = useCallback((endpoint) => request(endpoint, 'DELETE'), [request]);

  return { ...state, get, post, put, patch, del };
}

export default useApi;

// ใช้งาน
function CreatePostForm() {
  const { loading, error, post } = useApi('https://jsonplaceholder.typicode.com');
  const [title, setTitle] = useState('');

  const handleSubmit = async () => {
    try {
      const newPost = await post('/posts', { title, body: '...', userId: 1 });
      console.log('Created:', newPost);
      Alert.alert('สำเร็จ', 'สร้างโพสต์แล้ว!');
    } catch (err) {
      // error handled by hook
    }
  };

  return (
    <View>
      {error && <Text style={{ color: 'red' }}>{error}</Text>}
      <TextInput value={title} onChangeText={setTitle} placeholder="หัวข้อ" />
      <Button title={loading ? 'กำลังสร้าง...' : 'สร้าง'} onPress={handleSubmit} disabled={loading} />
    </View>
  );
}
```

---

## 3. useLocalStorage Hook (AsyncStorage)

```javascript
// hooks/useLocalStorage.js
import { useState, useEffect, useCallback } from 'react';
import AsyncStorage from '@react-native-async-storage/async-storage';

function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(initialValue);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // โหลดค่าจาก storage
  useEffect(() => {
    const loadValue = async () => {
      try {
        const item = await AsyncStorage.getItem(key);
        if (item !== null) {
          setStoredValue(JSON.parse(item));
        }
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    loadValue();
  }, [key]);

  // บันทึกค่า
  const setValue = useCallback(async (value) => {
    try {
      const valueToStore = value instanceof Function
        ? value(storedValue)
        : value;
      
      setStoredValue(valueToStore);
      await AsyncStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (err) {
      setError(err.message);
    }
  }, [key, storedValue]);

  // ลบค่า
  const removeValue = useCallback(async () => {
    try {
      await AsyncStorage.removeItem(key);
      setStoredValue(initialValue);
    } catch (err) {
      setError(err.message);
    }
  }, [key, initialValue]);

  return [storedValue, setValue, { loading, error, removeValue }];
}

export default useLocalStorage;

// ใช้งาน
function ThemeToggle() {
  const [theme, setTheme, { loading }] = useLocalStorage('@theme', 'light');

  if (loading) return null;

  return (
    <View>
      <Text>Theme: {theme}</Text>
      <Switch
        value={theme === 'dark'}
        onValueChange={(isDark) => setTheme(isDark ? 'dark' : 'light')}
      />
    </View>
  );
}

// ใช้กับ object
function useUserSettings() {
  return useLocalStorage('@user_settings', {
    notifications: true,
    language: 'th',
    fontSize: 16,
  });
}
```

---

## 4. useDebounce Hook

```javascript
// hooks/useDebounce.js
import { useState, useEffect } from 'react';

// Debounce value
function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Debounce callback
function useDebouncedCallback(callback, delay = 500) {
  const timerRef = useRef(null);

  const debouncedCallback = useCallback((...args) => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    timerRef.current = setTimeout(() => {
      callback(...args);
    }, delay);
  }, [callback, delay]);

  useEffect(() => {
    return () => {
      if (timerRef.current) {
        clearTimeout(timerRef.current);
      }
    };
  }, []);

  return debouncedCallback;
}

export { useDebounce, useDebouncedCallback };

// ใช้งาน - Search
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const debouncedQuery = useDebounce(query, 400);

  useEffect(() => {
    if (debouncedQuery) {
      searchApi(debouncedQuery).then(setResults);
    } else {
      setResults([]);
    }
  }, [debouncedQuery]);

  return (
    <View>
      <TextInput
        value={query}
        onChangeText={setQuery}
        placeholder="ค้นหา..."
      />
      <FlatList data={results} renderItem={...} />
    </View>
  );
}

// ใช้งาน - Auto Save
function NoteEditor() {
  const [note, setNote] = useLocalStorage('@current_note', '');
  const [saveStatus, setSaveStatus] = useState('saved');

  const saveNote = useDebouncedCallback(async (text) => {
    setSaveStatus('saving');
    await AsyncStorage.setItem('@note', text);
    setSaveStatus('saved');
  }, 1000);

  const handleChange = (text) => {
    setNote(text);
    setSaveStatus('unsaved');
    saveNote(text);
  };

  return (
    <View>
      <Text style={{ color: saveStatus === 'saved' ? 'green' : 'orange' }}>
        {saveStatus === 'saving' ? '💾 กำลังบันทึก...' :
         saveStatus === 'saved' ? '✅ บันทึกแล้ว' : '✏️ ยังไม่บันทึก'}
      </Text>
      <TextInput
        value={note}
        onChangeText={handleChange}
        multiline
        style={{ flex: 1, padding: 16 }}
      />
    </View>
  );
}
```

---

## 5. useWindowDimensions Hook

```javascript
// hooks/useWindowDimensions.js
import { useState, useEffect } from 'react';
import { Dimensions } from 'react-native';

function useWindowDimensions() {
  const [dimensions, setDimensions] = useState(() => {
    const { width, height } = Dimensions.get('window');
    return {
      width,
      height,
      isLandscape: width > height,
      isPortrait: width < height,
      isTablet: width >= 768,
      scale: Dimensions.get('window').scale,
      fontScale: Dimensions.get('window').fontScale,
    };
  });

  useEffect(() => {
    const subscription = Dimensions.addEventListener('change', ({ window, screen }) => {
      setDimensions({
        width: window.width,
        height: window.height,
        isLandscape: window.width > window.height,
        isPortrait: window.width < window.height,
        isTablet: window.width >= 768,
        scale: window.scale,
        fontScale: window.fontScale,
        screenWidth: screen.width,
        screenHeight: screen.height,
      });
    });

    return () => subscription?.remove();
  }, []);

  return dimensions;
}

export default useWindowDimensions;

// ใช้งาน
function ResponsiveLayout() {
  const { width, isTablet, isLandscape } = useWindowDimensions();

  const numColumns = isTablet ? (isLandscape ? 4 : 3) : 2;
  const cardWidth = (width - 24 - (numColumns - 1) * 8) / numColumns;

  return (
    <FlatList
      numColumns={numColumns}
      key={numColumns}  // Force re-render เมื่อ columns เปลี่ยน
      data={products}
      renderItem={({ item }) => (
        <ProductCard style={{ width: cardWidth }} product={item} />
      )}
    />
  );
}
```

---

## Workshop: สร้าง Hooks Library

### โครงสร้าง

```
hooks/
├── index.js          // Export ทุก hook
├── useFetch.js
├── useLocalStorage.js
├── useDebounce.js
├── useWindowDimensions.js
├── useCounter.js
├── useToggle.js
├── useTimer.js
├── useForm.js
└── useNetworkStatus.js
```

### useCounter Hook

```javascript
// hooks/useCounter.js
import { useState, useCallback } from 'react';

function useCounter(initialValue = 0, { min, max, step = 1 } = {}) {
  const [count, setCount] = useState(initialValue);

  const increment = useCallback(() => {
    setCount(c => {
      const next = c + step;
      return max !== undefined ? Math.min(next, max) : next;
    });
  }, [step, max]);

  const decrement = useCallback(() => {
    setCount(c => {
      const next = c - step;
      return min !== undefined ? Math.max(next, min) : next;
    });
  }, [step, min]);

  const reset = useCallback(() => setCount(initialValue), [initialValue]);

  const setValue = useCallback((value) => {
    setCount(() => {
      let v = typeof value === 'function' ? value(count) : value;
      if (min !== undefined) v = Math.max(v, min);
      if (max !== undefined) v = Math.min(v, max);
      return v;
    });
  }, [count, min, max]);

  return {
    count,
    increment,
    decrement,
    reset,
    setValue,
    isMin: min !== undefined && count <= min,
    isMax: max !== undefined && count >= max,
  };
}

export default useCounter;
```

### useToggle Hook

```javascript
// hooks/useToggle.js
import { useState, useCallback } from 'react';

function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => setValue(v => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return [value, { toggle, setTrue, setFalse }];
}

export default useToggle;

// ใช้งาน
function PasswordInput() {
  const [showPassword, { toggle }] = useToggle(false);

  return (
    <View style={{ flexDirection: 'row' }}>
      <TextInput
        secureTextEntry={!showPassword}
        placeholder="รหัสผ่าน"
        style={{ flex: 1 }}
      />
      <TouchableOpacity onPress={toggle}>
        <Ionicons name={showPassword ? 'eye-off' : 'eye'} size={22} />
      </TouchableOpacity>
    </View>
  );
}
```

### useTimer Hook

```javascript
// hooks/useTimer.js
import { useState, useEffect, useRef, useCallback } from 'react';

function useTimer(initialSeconds = 0, { autoStart = false, onComplete } = {}) {
  const [seconds, setSeconds] = useState(initialSeconds);
  const [isRunning, setIsRunning] = useState(autoStart);
  const intervalRef = useRef(null);
  const onCompleteRef = useRef(onComplete);
  onCompleteRef.current = onComplete;

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(prev => {
          if (prev <= 1) {
            setIsRunning(false);
            onCompleteRef.current?.();
            return 0;
          }
          return prev - 1;
        });
      }, 1000);
    }

    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, [isRunning]);

  const start = useCallback(() => setIsRunning(true), []);
  const pause = useCallback(() => setIsRunning(false), []);
  const reset = useCallback(() => {
    setIsRunning(false);
    setSeconds(initialSeconds);
  }, [initialSeconds]);

  const restart = useCallback(() => {
    setSeconds(initialSeconds);
    setIsRunning(true);
  }, [initialSeconds]);

  // Format เป็น MM:SS
  const formatted = `${String(Math.floor(seconds / 60)).padStart(2, '0')}:${String(seconds % 60).padStart(2, '0')}`;

  return { seconds, formatted, isRunning, start, pause, reset, restart };
}

export default useTimer;

// ใช้งาน - Countdown Timer
function OTPTimer({ onExpire }) {
  const { formatted, seconds, restart } = useTimer(60, {
    autoStart: true,
    onComplete: onExpire,
  });

  return (
    <View style={{ alignItems: 'center' }}>
      <Text style={{ fontSize: 32, fontWeight: 'bold', color: seconds < 10 ? '#E53935' : '#212121' }}>
        {formatted}
      </Text>
      {seconds === 0 && (
        <TouchableOpacity onPress={restart}>
          <Text style={{ color: '#6200EE' }}>ส่งรหัสใหม่</Text>
        </TouchableOpacity>
      )}
    </View>
  );
}
```

### useForm Hook

```javascript
// hooks/useForm.js
import { useState, useCallback } from 'react';

function useForm(initialValues = {}, validationSchema = {}) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [submitting, setSubmitting] = useState(false);

  const validate = useCallback((fieldValues = values) => {
    const newErrors = {};
    
    Object.keys(validationSchema).forEach(field => {
      const rules = validationSchema[field];
      const value = fieldValues[field];
      
      if (rules.required && (!value || !value.toString().trim())) {
        newErrors[field] = rules.required === true ? 'จำเป็นต้องกรอก' : rules.required;
      } else if (rules.minLength && value && value.length < rules.minLength) {
        newErrors[field] = `ต้องมีอย่างน้อย ${rules.minLength} ตัวอักษร`;
      } else if (rules.maxLength && value && value.length > rules.maxLength) {
        newErrors[field] = `ต้องไม่เกิน ${rules.maxLength} ตัวอักษร`;
      } else if (rules.pattern && value && !rules.pattern.test(value)) {
        newErrors[field] = rules.patternMessage || 'รูปแบบไม่ถูกต้อง';
      } else if (rules.validate && value) {
        const error = rules.validate(value, fieldValues);
        if (error) newErrors[field] = error;
      }
    });
    
    return newErrors;
  }, [values, validationSchema]);

  const handleChange = useCallback((field) => (value) => {
    setValues(prev => ({ ...prev, [field]: value }));
    if (touched[field]) {
      const newErrors = validate({ ...values, [field]: value });
      setErrors(prev => ({ ...prev, [field]: newErrors[field] }));
    }
  }, [values, touched, validate]);

  const handleBlur = useCallback((field) => () => {
    setTouched(prev => ({ ...prev, [field]: true }));
    const newErrors = validate();
    setErrors(prev => ({ ...prev, [field]: newErrors[field] }));
  }, [validate]);

  const handleSubmit = useCallback((onSubmit) => async () => {
    const allTouched = Object.keys(values).reduce(
      (acc, key) => ({ ...acc, [key]: true }), {}
    );
    setTouched(allTouched);

    const newErrors = validate();
    setErrors(newErrors);

    if (Object.keys(newErrors).length === 0) {
      setSubmitting(true);
      try {
        await onSubmit(values);
      } finally {
        setSubmitting(false);
      }
    }
  }, [values, validate]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
    setSubmitting(false);
  }, [initialValues]);

  const setFieldValue = useCallback((field, value) => {
    setValues(prev => ({ ...prev, [field]: value }));
  }, []);

  const isValid = Object.keys(validate()).length === 0;

  return {
    values,
    errors,
    touched,
    submitting,
    isValid,
    handleChange,
    handleBlur,
    handleSubmit,
    reset,
    setFieldValue,
  };
}

export default useForm;
```

### useNetworkStatus Hook

```javascript
// hooks/useNetworkStatus.js
import { useState, useEffect } from 'react';
import NetInfo from '@react-native-community/netinfo';

function useNetworkStatus() {
  const [networkState, setNetworkState] = useState({
    isConnected: true,
    isInternetReachable: true,
    type: 'unknown',
    isWifi: false,
    isCellular: false,
    details: null,
  });

  useEffect(() => {
    // ตรวจสอบสถานะ network เริ่มต้น
    NetInfo.fetch().then(state => {
      setNetworkState({
        isConnected: state.isConnected,
        isInternetReachable: state.isInternetReachable,
        type: state.type,
        isWifi: state.type === 'wifi',
        isCellular: state.type === 'cellular',
        details: state.details,
      });
    });

    // Subscribe ฟัง network changes
    const unsubscribe = NetInfo.addEventListener(state => {
      setNetworkState({
        isConnected: state.isConnected,
        isInternetReachable: state.isInternetReachable,
        type: state.type,
        isWifi: state.type === 'wifi',
        isCellular: state.type === 'cellular',
        details: state.details,
      });
    });

    return () => unsubscribe();
  }, []);

  return networkState;
}

export default useNetworkStatus;

// ใช้งาน
function AppWithNetworkStatus() {
  const { isConnected, isInternetReachable, type } = useNetworkStatus();

  return (
    <View style={{ flex: 1 }}>
      {!isConnected && (
        <View style={{
          backgroundColor: '#E53935',
          padding: 8,
          alignItems: 'center',
        }}>
          <Text style={{ color: '#FFF' }}>
            ไม่มีการเชื่อมต่ออินเทอร์เน็ต
          </Text>
        </View>
      )}
      {isConnected && !isInternetReachable && (
        <View style={{
          backgroundColor: '#FFA000',
          padding: 8,
          alignItems: 'center',
        }}>
          <Text style={{ color: '#FFF' }}>
            เชื่อมต่อแล้วแต่ไม่สามารถเข้าอินเทอร์เน็ตได้
          </Text>
        </View>
      )}
      {/* App content */}
    </View>
  );
}
```

### index.js - Export ทุก Hooks

```javascript
// hooks/index.js
export { default as useFetch } from './useFetch';
export { default as useLocalStorage } from './useLocalStorage';
export { useDebounce, useDebouncedCallback } from './useDebounce';
export { default as useWindowDimensions } from './useWindowDimensions';
export { default as useCounter } from './useCounter';
export { default as useToggle } from './useToggle';
export { default as useTimer } from './useTimer';
export { default as useForm } from './useForm';
export { default as useNetworkStatus } from './useNetworkStatus';
```

### ตัวอย่างการใช้งาน Hooks Library ทั้งหมด

```javascript
// screens/DemoScreen.js
import React from 'react';
import { View, Text, TextInput, TouchableOpacity, StyleSheet } from 'react-native';
import {
  useCounter,
  useToggle,
  useTimer,
  useForm,
  useNetworkStatus,
} from '../hooks';

export default function DemoScreen() {
  // Counter
  const { count, increment, decrement, reset: resetCounter } = useCounter(0, {
    min: 0,
    max: 10,
    step: 1,
  });

  // Toggle
  const [isDark, { toggle: toggleDark }] = useToggle(false);

  // Timer
  const { formatted, isRunning, start, pause, reset: resetTimer } = useTimer(60);

  // Network
  const { isConnected, type } = useNetworkStatus();

  // Form
  const { values, errors, touched, handleChange, handleBlur, handleSubmit, submitting } = useForm(
    { email: '', message: '' },
    {
      email: {
        required: 'กรุณากรอกอีเมล',
        pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
        patternMessage: 'รูปแบบอีเมลไม่ถูกต้อง',
      },
      message: {
        required: 'กรุณากรอกข้อความ',
        minLength: 10,
      },
    }
  );

  const onSubmit = handleSubmit(async (formValues) => {
    console.log('Form submitted:', formValues);
    await new Promise(r => setTimeout(r, 1000));
    alert('ส่งสำเร็จ!');
  });

  return (
    <View style={[styles.container, isDark && styles.dark]}>
      {/* Counter Demo */}
      <View style={styles.section}>
        <Text style={[styles.sectionTitle, isDark && styles.textLight]}>Counter</Text>
        <View style={styles.row}>
          <TouchableOpacity style={styles.btn} onPress={decrement}>
            <Text style={styles.btnText}>-</Text>
          </TouchableOpacity>
          <Text style={[styles.countText, isDark && styles.textLight]}>{count}</Text>
          <TouchableOpacity style={styles.btn} onPress={increment}>
            <Text style={styles.btnText}>+</Text>
          </TouchableOpacity>
        </View>
        <TouchableOpacity onPress={resetCounter}>
          <Text style={styles.linkText}>รีเซ็ต</Text>
        </TouchableOpacity>
      </View>

      {/* Timer Demo */}
      <View style={styles.section}>
        <Text style={[styles.sectionTitle, isDark && styles.textLight]}>Timer</Text>
        <Text style={[styles.timerText, isDark && styles.textLight]}>{formatted}</Text>
        <View style={styles.row}>
          {isRunning
            ? <TouchableOpacity style={styles.btn} onPress={pause}>
                <Text style={styles.btnText}>หยุด</Text>
              </TouchableOpacity>
            : <TouchableOpacity style={styles.btn} onPress={start}>
                <Text style={styles.btnText}>เริ่ม</Text>
              </TouchableOpacity>
          }
          <TouchableOpacity style={[styles.btn, styles.grayBtn]} onPress={resetTimer}>
            <Text style={styles.btnText}>รีเซ็ต</Text>
          </TouchableOpacity>
        </View>
      </View>

      {/* Network Demo */}
      <View style={styles.section}>
        <Text style={[styles.sectionTitle, isDark && styles.textLight]}>Network</Text>
        <Text style={[
          styles.networkStatus,
          { color: isConnected ? '#4CAF50' : '#E53935' }
        ]}>
          {isConnected ? '✅ เชื่อมต่อแล้ว' : '❌ ไม่มีสัญญาณ'} ({type})
        </Text>
      </View>

      {/* Form Demo */}
      <View style={styles.section}>
        <Text style={[styles.sectionTitle, isDark && styles.textLight]}>Form</Text>
        <TextInput
          style={[styles.input, touched.email && errors.email && styles.inputError]}
          placeholder="อีเมล"
          value={values.email}
          onChangeText={handleChange('email')}
          onBlur={handleBlur('email')}
          keyboardType="email-address"
          autoCapitalize="none"
        />
        {touched.email && errors.email && (
          <Text style={styles.errorText}>{errors.email}</Text>
        )}
        <TextInput
          style={[styles.input, styles.textarea, touched.message && errors.message && styles.inputError]}
          placeholder="ข้อความ (อย่างน้อย 10 ตัวอักษร)"
          value={values.message}
          onChangeText={handleChange('message')}
          onBlur={handleBlur('message')}
          multiline
        />
        {touched.message && errors.message && (
          <Text style={styles.errorText}>{errors.message}</Text>
        )}
        <TouchableOpacity
          style={[styles.submitBtn, submitting && styles.disabledBtn]}
          onPress={onSubmit}
          disabled={submitting}
        >
          <Text style={styles.submitText}>
            {submitting ? 'กำลังส่ง...' : 'ส่งข้อความ'}
          </Text>
        </TouchableOpacity>
      </View>

      {/* Dark Mode Toggle */}
      <TouchableOpacity style={styles.darkToggle} onPress={toggleDark}>
        <Text style={styles.darkToggleText}>
          {isDark ? '☀️ Light Mode' : '🌙 Dark Mode'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: '#F5F5F5' },
  dark: { backgroundColor: '#121212' },
  section: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    padding: 16,
    marginBottom: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.08,
    shadowRadius: 3,
    elevation: 2,
  },
  sectionTitle: { fontSize: 16, fontWeight: 'bold', marginBottom: 12 },
  textLight: { color: '#FFF' },
  row: { flexDirection: 'row', alignItems: 'center', gap: 12 },
  btn: {
    backgroundColor: '#6200EE',
    width: 44,
    height: 44,
    borderRadius: 22,
    alignItems: 'center',
    justifyContent: 'center',
  },
  grayBtn: { backgroundColor: '#999' },
  btnText: { color: '#FFF', fontSize: 20, fontWeight: 'bold' },
  countText: { fontSize: 32, fontWeight: 'bold', minWidth: 50, textAlign: 'center' },
  linkText: { color: '#6200EE', marginTop: 8, textAlign: 'center' },
  timerText: { fontSize: 40, fontWeight: 'bold', textAlign: 'center', marginBottom: 12 },
  networkStatus: { fontSize: 15 },
  input: {
    borderWidth: 1,
    borderColor: '#DDD',
    borderRadius: 8,
    padding: 12,
    fontSize: 15,
    marginBottom: 4,
  },
  inputError: { borderColor: '#E53935' },
  textarea: { height: 80, textAlignVertical: 'top' },
  errorText: { color: '#E53935', fontSize: 12, marginBottom: 8 },
  submitBtn: {
    backgroundColor: '#6200EE',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginTop: 8,
  },
  disabledBtn: { backgroundColor: '#CCC' },
  submitText: { color: '#FFF', fontWeight: '600', fontSize: 15 },
  darkToggle: {
    backgroundColor: '#6200EE',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  darkToggleText: { color: '#FFF', fontWeight: '600' },
});
```

---

## Tips และ Best Practices

```
✅ DO:
- ตั้งชื่อ Hook ขึ้นต้นด้วย use เสมอ
- แต่ละ Hook ควรทำงานหน้าเดียว (Single Responsibility)
- Return object แทน array เมื่อมีหลาย values
- Document parameters และ return values ด้วย JSDoc
- Test custom hooks แยกต่างหาก

❌ DON'T:
- ไม่เรียก Hook แบบ conditional
- ไม่ return JSX จาก Hook (ใช้ component แทน)
- ไม่ทำ side effects โดยตรงใน body ของ Hook

📦 npm packages ที่น่าสนใจ:
- react-query: Data fetching hooks
- react-hook-form: Form hooks
- @tanstack/react-query: Advanced data fetching
```

---

## สรุป

Custom Hooks ช่วย reuse logic และทำให้ code สะอาดขึ้น:

1. **สร้าง Custom Hook** - principles และ patterns
2. **useFetch** - data fetching hook ที่สมบูรณ์
3. **useLocalStorage** - persistence hook
4. **useDebounce** - debounce value และ callback
5. **useWindowDimensions** - responsive hook
6. **Workshop** - Hooks library สมบูรณ์พร้อม demo
