# Part 078: Hermes Engine

## Hermes คืออะไร?

Hermes คือ JavaScript engine ที่ Facebook พัฒนาขึ้นเพื่อ React Native โดยเฉพาะ ออกแบบมาเพื่อ optimize สำหรับ mobile devices โดยเน้น startup time, memory usage, และ app size

## ทำไม Hermes ถึงเร็วกว่า?

### Standard JS Engine (V8/JSC)
```
Source Code → Parse → AST → Compile to Bytecode → Execute
                              (ทำตอน runtime)
```

### Hermes
```
Source Code → Parse → AST → Compile to Bytecode → Ship with app
                              (ทำตอน build time)
                                    ↓
                              Execute Bytecode (เร็วกว่า!)
```

---

## 1. การ Enable Hermes

### Android

```properties
# android/gradle.properties
hermesEnabled=true
```

```groovy
// android/app/build.gradle
android {
  defaultConfig {
    // ...
  }
  
  buildTypes {
    release {
      // Hermes enabled automatically via gradle.properties
    }
  }
}
```

### iOS

```ruby
# ios/Podfile
use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => true
)
```

### ตรวจสอบว่า Hermes ทำงานอยู่

```typescript
// CheckHermes.ts
import { HermesInternal } from 'react-native';

export function isHermesEnabled(): boolean {
  return !!(global as any).HermesInternal;
}

export function getHermesVersion(): string | null {
  const hermesInternal = (global as any).HermesInternal;
  if (!hermesInternal) return null;
  return hermesInternal.getRuntimeProperties?.()?.['OSS Release Version'] ?? null;
}

// Component to display status
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

export const HermesStatus: React.FC = () => {
  const enabled = isHermesEnabled();
  const version = getHermesVersion();
  
  return (
    <View style={styles.container}>
      <Text style={styles.label}>JavaScript Engine:</Text>
      <Text style={[styles.value, enabled ? styles.hermes : styles.jsc]}>
        {enabled ? `Hermes ${version ?? ''}` : 'JSC (JavaScriptCore)'}
      </Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 8
  },
  label: { fontSize: 14, color: '#666', marginRight: 8 },
  value: { fontSize: 14, fontWeight: '600' },
  hermes: { color: '#007AFF' },
  jsc: { color: '#FF9500' }
});
```

---

## 2. Bytecode Compilation

### คือการแปลง JavaScript เป็น Bytecode ก่อน Ship

```bash
# ดู bytecode ที่ Hermes สร้าง
hermes -emit-binary -out output.hbc input.js

# ตรวจสอบ bytecode
hermes -dump-bytecode output.hbc
```

### Bundle Optimization

```javascript
// metro.config.js
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: true, // ✅ Hermes ชอบสิ่งนี้
      },
    }),
  },
  serializer: {
    customSerializer: (entryPoint, preModules, graph, options) => {
      // Custom serialization for Hermes
      return require('@react-native/metro-config')
        .createHermesSerializer()(entryPoint, preModules, graph, options);
    }
  }
};
```

---

## 3. Hermes Debugger

### 3.1 การ Debug ด้วย Chrome DevTools

```bash
# เปิด React Native dev server
npx react-native start

# เปิด Hermes debugger
# URL: chrome://inspect/#devices
# หรือ about:inspect
```

### 3.2 Flipper Integration

```typescript
// ใน __DEV__ mode, Hermes รองรับ Flipper debugging
if (__DEV__) {
  // Hermes will expose debugging via Flipper automatically
  console.log('Running in development mode with Hermes debugger available');
}
```

### 3.3 เปิดใช้ Source Maps

```javascript
// metro.config.js
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        // Enable source maps for debugging
        generateSourceMaps: true,
      },
    }),
  },
};
```

---

## 4. Performance Comparison

```tsx
// HermesBenchmark.tsx
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  ScrollView,
  StyleSheet
} from 'react-native';

interface BenchmarkResult {
  name: string;
  duration: number;
  iterations: number;
}

const HermesBenchmark: React.FC = () => {
  const [results, setResults] = useState<BenchmarkResult[]>([]);
  const [running, setRunning] = useState(false);
  
  const benchmark = useCallback((
    name: string,
    fn: () => void,
    iterations: number
  ): BenchmarkResult => {
    const start = performance.now();
    for (let i = 0; i < iterations; i++) {
      fn();
    }
    return {
      name,
      duration: performance.now() - start,
      iterations
    };
  }, []);
  
  const runBenchmarks = useCallback(async () => {
    setRunning(true);
    const newResults: BenchmarkResult[] = [];
    
    // 1. String operations
    newResults.push(benchmark(
      'String concatenation',
      () => {
        let s = '';
        for (let i = 0; i < 100; i++) s += 'a';
      },
      10000
    ));
    
    // 2. Array operations
    newResults.push(benchmark(
      'Array map/filter',
      () => {
        const arr = Array.from({ length: 100 }, (_, i) => i);
        arr.map(x => x * 2).filter(x => x > 50);
      },
      10000
    ));
    
    // 3. Object creation
    newResults.push(benchmark(
      'Object creation',
      () => {
        const obj = { a: 1, b: 'hello', c: [1, 2, 3], d: { nested: true } };
        JSON.stringify(obj);
      },
      10000
    ));
    
    // 4. Math operations
    newResults.push(benchmark(
      'Math operations',
      () => {
        let sum = 0;
        for (let i = 0; i < 1000; i++) {
          sum += Math.sqrt(i) * Math.sin(i);
        }
      },
      1000
    ));
    
    // 5. Regular expressions
    newResults.push(benchmark(
      'RegExp matching',
      () => {
        const text = 'Hello World React Native Hermes Performance Test';
        /\b\w+\b/g.exec(text);
      },
      10000
    ));
    
    // 6. JSON parse/stringify
    newResults.push(benchmark(
      'JSON parse/stringify',
      () => {
        const data = { id: 1, name: 'test', items: [1, 2, 3, 4, 5] };
        JSON.parse(JSON.stringify(data));
      },
      10000
    ));
    
    // 7. Closure creation
    newResults.push(benchmark(
      'Closure creation',
      () => {
        const fns = Array.from({ length: 10 }, (_, i) => (x: number) => x + i);
        fns.forEach(fn => fn(1));
      },
      10000
    ));
    
    setResults(newResults);
    setRunning(false);
  }, [benchmark]);
  
  const isHermes = !!(global as any).HermesInternal;
  
  return (
    <ScrollView style={styles.container}>
      <Text style={styles.title}>Hermes Benchmark</Text>
      
      <View style={styles.engineInfo}>
        <Text style={styles.engineLabel}>Engine: </Text>
        <Text style={[styles.engineName, isHermes ? styles.hermes : styles.jsc]}>
          {isHermes ? 'Hermes' : 'JavaScriptCore'}
        </Text>
      </View>
      
      <TouchableOpacity
        style={[styles.button, running && styles.buttonDisabled]}
        onPress={runBenchmarks}
        disabled={running}
      >
        <Text style={styles.buttonText}>
          {running ? 'Running...' : 'Run Benchmarks'}
        </Text>
      </TouchableOpacity>
      
      {results.map((result, index) => (
        <View key={index} style={styles.resultCard}>
          <Text style={styles.resultName}>{result.name}</Text>
          <View style={styles.resultRow}>
            <Text style={styles.resultDetail}>
              {result.iterations.toLocaleString()} iterations
            </Text>
            <Text style={styles.resultTime}>
              {result.duration.toFixed(2)}ms
            </Text>
          </View>
          <Text style={styles.resultOps}>
            {Math.round(result.iterations / (result.duration / 1000)).toLocaleString()} ops/sec
          </Text>
        </View>
      ))}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 22, fontWeight: 'bold', marginBottom: 8 },
  engineInfo: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 16,
    padding: 12,
    backgroundColor: '#f0f0f0',
    borderRadius: 8
  },
  engineLabel: { fontSize: 16, color: '#666' },
  engineName: { fontSize: 16, fontWeight: '700' },
  hermes: { color: '#007AFF' },
  jsc: { color: '#FF9500' },
  button: {
    backgroundColor: '#007AFF',
    padding: 14,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 16
  },
  buttonDisabled: { backgroundColor: '#ccc' },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  resultCard: {
    backgroundColor: '#fff',
    padding: 14,
    borderRadius: 8,
    marginBottom: 8,
    borderWidth: 1,
    borderColor: '#e0e0e0'
  },
  resultName: { fontSize: 15, fontWeight: '600', marginBottom: 6 },
  resultRow: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 4 },
  resultDetail: { fontSize: 13, color: '#666' },
  resultTime: { fontSize: 13, fontWeight: '500' },
  resultOps: { fontSize: 13, color: '#34C759', fontWeight: '600' }
});

export default HermesBenchmark;
```

---

## 5. Hermes Limitations และ Workarounds

### 5.1 Features ที่ไม่รองรับ (เวอร์ชันเก่า)

```typescript
// ❌ Proxy ใน Hermes เวอร์ชันเก่า (แก้ไขแล้วใน versions ใหม่)
// const proxy = new Proxy({}, handler);

// ✅ Workaround
class ObservableObject {
  private _data: Record<string, any> = {};
  private _observers: Map<string, Function[]> = new Map();
  
  set(key: string, value: any) {
    this._data[key] = value;
    this._observers.get(key)?.forEach(fn => fn(value));
  }
  
  get(key: string) {
    return this._data[key];
  }
  
  observe(key: string, fn: Function) {
    if (!this._observers.has(key)) {
      this._observers.set(key, []);
    }
    this._observers.get(key)!.push(fn);
  }
}
```

### 5.2 Eval ไม่ทำงาน

```typescript
// ❌ eval ไม่ทำงานใน Hermes (disabled for security)
// eval('1 + 1');

// ✅ Workaround: ใช้ Function constructor ก็ไม่ได้
// แก้ไขด้วยการ avoid eval โดยสิ้นเชิง

// ❌ Dynamic code execution
// const fn = new Function('x', 'return x * 2');

// ✅ Static code
const double = (x: number) => x * 2;
```

### 5.3 Performance ที่ต่างออกไป

```typescript
// Hermes optimize บางอย่างต่างจาก V8
// ตรวจสอบ RegExp behavior

// ✅ Hermes มี optimization สำหรับ common patterns
const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// Compile regex ครั้งเดียว (Hermes จะ cache)
export function validateEmail(email: string): boolean {
  return EMAIL_REGEX.test(email);
}
```

---

## 6. Hermes สำหรับ Production

### 6.1 Proguard Rules (Android)

```pro
# android/app/proguard-rules.pro
-keep class com.facebook.hermes.unicode.** { *; }
-keep class com.facebook.jni.** { *; }
```

### 6.2 Bundle Size Comparison

```bash
# ตรวจสอบ bundle size
# Before Hermes
ls -la android/app/build/outputs/apk/release/*.apk

# After Hermes (ควรเล็กกว่า)
ls -la android/app/build/outputs/apk/release/*.apk

# ดู bytecode bundle
ls -la android/app/src/main/assets/index.android.bundle
```

### 6.3 Crash Reporting กับ Hermes

```typescript
// CrashHandler.ts
import { ErrorUtils } from 'react-native';

// Setup global error handler
const originalHandler = ErrorUtils.getGlobalHandler();

ErrorUtils.setGlobalHandler((error: Error, isFatal: boolean) => {
  const isHermes = !!(global as any).HermesInternal;
  
  // Log extra Hermes info
  if (isHermes) {
    console.error('Hermes crash:', {
      message: error.message,
      stack: error.stack,
      isFatal,
      engine: 'Hermes'
    });
  }
  
  // Call original handler
  originalHandler?.(error, isFatal);
});

// Setup error boundaries
import React from 'react';

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class HermesAwareErrorBoundary extends React.Component<
  { children: React.ReactNode },
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = { hasError: false, error: null };
  
  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    const isHermes = !!(global as any).HermesInternal;
    console.error('ErrorBoundary caught error:', error, {
      ...info,
      engine: isHermes ? 'Hermes' : 'JSC'
    });
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <Text>Something went wrong</Text>
          <Text>{this.state.error?.message}</Text>
        </View>
      );
    }
    return this.props.children;
  }
}
```

---

## 7. Hermes Garbage Collection

```typescript
// GCOptimization.ts
// Hermes มี GC ที่ optimize สำหรับ mobile

// ✅ ปล่อย references เพื่อช่วย GC
function processLargeData(data: any[]) {
  let result = null;
  
  try {
    // Process data
    result = data.map(item => ({ ...item, processed: true }));
  } finally {
    // ล้าง reference ที่ไม่ต้องการ
    data = null as any;
  }
  
  return result;
}

// ✅ ใช้ WeakRef สำหรับ optional references (Hermes รองรับ)
const cache = new Map<string, WeakRef<any>>();

function getCached(key: string) {
  const ref = cache.get(key);
  if (ref) {
    const value = ref.deref();
    if (value !== undefined) return value;
    cache.delete(key); // Clean up dead references
  }
  return null;
}

function setCached(key: string, value: any) {
  cache.set(key, new WeakRef(value));
}
```

---

## Tips สำหรับการใช้ Hermes

### 1. Enable ก่อน Production
```bash
# Test Hermes ตั้งแต่ development
# อย่ารอ switch ตอน production
```

### 2. ตรวจสอบ Library Compatibility
```typescript
// บาง libraries อาจมีปัญหากับ Hermes
// ตรวจสอบ issues บน GitHub ก่อนใช้
```

### 3. Source Maps สำหรับ Crash Reports
```javascript
// metro.config.js - Enable source maps
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        generateSourceMaps: true
      }
    })
  }
};
```

### 4. Memory Profiling
```typescript
// ใช้ Hermes profiler ใน Flipper
// Memory tab → Profile heap
```

---

## สรุป

Hermes ช่วยให้ React Native app:
1. **Startup เร็วขึ้น** - bytecode ไม่ต้อง parse ใหม่
2. **Memory ใช้น้อยลง** - compact bytecode
3. **App size เล็กลง** - optimize bundle
4. **Consistent performance** - predictable GC

แนะนำให้ enable Hermes ใน production เสมอ เพราะ Facebook ใช้ใน production apps ของตัวเองแล้ว
