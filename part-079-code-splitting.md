# Part 079: Code Splitting และ Lazy Loading

## Code Splitting คืออะไร?

Code Splitting คือเทคนิคการแบ่ง JavaScript bundle เป็น chunks เล็กๆ เพื่อให้แอปโหลดเฉพาะ code ที่จำเป็นในแต่ละ screen แทนที่จะโหลด bundle ทั้งหมดตอนเริ่มต้น

---

## 1. Dynamic Imports

### 1.1 Basic Dynamic Import

```typescript
// BasicDynamicImport.ts

// ❌ Static import (โหลดทั้งหมดตอน startup)
import HeavyComponent from './HeavyComponent';
import AdvancedFeature from './AdvancedFeature';

// ✅ Dynamic import (โหลดเมื่อจำเป็น)
const loadHeavyComponent = () => import('./HeavyComponent');
const loadAdvancedFeature = () => import('./AdvancedFeature');

// ใช้งาน
async function showHeavyFeature() {
  const module = await loadHeavyComponent();
  const HeavyComponent = module.default;
  // ใช้งาน HeavyComponent
}
```

### 1.2 Dynamic Import กับ Module

```typescript
// DynamicModuleLoader.ts
type ModuleKey = 'calendar' | 'camera' | 'maps' | 'payment';

const moduleLoaders: Record<ModuleKey, () => Promise<any>> = {
  calendar: () => import('./features/calendar'),
  camera: () => import('./features/camera'),
  maps: () => import('./features/maps'),
  payment: () => import('./features/payment')
};

const moduleCache = new Map<ModuleKey, any>();

export async function loadModule(key: ModuleKey): Promise<any> {
  if (moduleCache.has(key)) {
    return moduleCache.get(key);
  }
  
  console.log(`Loading module: ${key}`);
  const start = performance.now();
  
  const module = await moduleLoaders[key]();
  moduleCache.set(key, module);
  
  console.log(`Module ${key} loaded in ${(performance.now() - start).toFixed(1)}ms`);
  return module;
}

// Preload modules ที่คาดว่าจะใช้เร็วๆ นี้
export function preloadModule(key: ModuleKey): void {
  if (!moduleCache.has(key)) {
    moduleLoaders[key]().then(module => {
      moduleCache.set(key, module);
    });
  }
}
```

---

## 2. React.lazy และ Suspense

### 2.1 Basic Lazy Loading

```tsx
// LazyRoutes.tsx
import React, { Suspense, lazy, useState } from 'react';
import { View, ActivityIndicator, Text, StyleSheet } from 'react-native';

// Lazy load components
const HomeScreen = lazy(() => import('./screens/HomeScreen'));
const ProfileScreen = lazy(() => import('./screens/ProfileScreen'));
const SettingsScreen = lazy(() => import('./screens/SettingsScreen'));
const AdvancedEditorScreen = lazy(() => import('./screens/AdvancedEditorScreen'));

// Loading fallback
const LoadingScreen: React.FC<{ message?: string }> = ({ message }) => (
  <View style={styles.loading}>
    <ActivityIndicator size="large" color="#007AFF" />
    {message && <Text style={styles.loadingText}>{message}</Text>}
  </View>
);

// Error boundary สำหรับ lazy components
class LazyLoadErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback?: React.ReactNode },
  { hasError: boolean; error: Error | null }
> {
  state = { hasError: false, error: null };
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <View style={styles.error}>
          <Text style={styles.errorText}>Failed to load screen</Text>
          <Text style={styles.errorDetail}>
            {(this.state.error as any)?.message}
          </Text>
        </View>
      );
    }
    return this.props.children;
  }
}

type Screen = 'home' | 'profile' | 'settings' | 'editor';

const LazyApp: React.FC = () => {
  const [currentScreen, setCurrentScreen] = useState<Screen>('home');
  
  const renderScreen = () => {
    switch (currentScreen) {
      case 'home': return <HomeScreen />;
      case 'profile': return <ProfileScreen />;
      case 'settings': return <SettingsScreen />;
      case 'editor': return <AdvancedEditorScreen />;
    }
  };
  
  return (
    <View style={styles.container}>
      <LazyLoadErrorBoundary>
        <Suspense fallback={<LoadingScreen message="กำลังโหลด..." />}>
          {renderScreen()}
        </Suspense>
      </LazyLoadErrorBoundary>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  loading: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center'
  },
  loadingText: {
    marginTop: 12,
    fontSize: 14,
    color: '#666'
  },
  error: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20
  },
  errorText: {
    fontSize: 18,
    color: '#FF3B30',
    marginBottom: 8
  },
  errorDetail: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center'
  }
});

export default LazyApp;
```

### 2.2 Advanced Suspense Pattern

```tsx
// AdvancedSuspense.tsx
import React, { Suspense, lazy, useTransition, useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

// Lazy load with prefetch
function lazyWithPrefetch<T extends React.ComponentType<any>>(
  factory: () => Promise<{ default: T }>
) {
  let componentPromise: Promise<{ default: T }> | null = null;
  
  const LazyComponent = lazy(() => {
    if (!componentPromise) {
      componentPromise = factory();
    }
    return componentPromise;
  });
  
  const prefetch = () => {
    if (!componentPromise) {
      componentPromise = factory();
    }
  };
  
  return { LazyComponent, prefetch };
}

const { LazyComponent: HeavyChart, prefetch: prefetchChart } = lazyWithPrefetch(
  () => import('./components/HeavyChart')
);

const { LazyComponent: VideoPlayer, prefetch: prefetchVideo } = lazyWithPrefetch(
  () => import('./components/VideoPlayer')
);

const SmartScreen: React.FC = () => {
  const [showChart, setShowChart] = useState(false);
  const [isPending, startTransition] = useTransition();
  
  const handleChartHover = () => {
    // Prefetch when user hovers (anticipate need)
    prefetchChart();
  };
  
  const handleShowChart = () => {
    startTransition(() => {
      setShowChart(true);
    });
  };
  
  return (
    <View style={styles.container}>
      <TouchableOpacity
        style={[styles.button, isPending && styles.pending]}
        onPress={handleShowChart}
        onLongPress={handleChartHover}
      >
        <Text style={styles.buttonText}>
          {isPending ? 'กำลังโหลด...' : 'แสดง Chart'}
        </Text>
      </TouchableOpacity>
      
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart data={[1, 2, 3, 4, 5]} />
        </Suspense>
      )}
    </View>
  );
};

const ChartSkeleton: React.FC = () => (
  <View style={styles.skeleton}>
    <View style={styles.skeletonBar} />
    <View style={[styles.skeletonBar, { height: 120 }]} />
    <View style={[styles.skeletonBar, { height: 80 }]} />
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  button: {
    backgroundColor: '#007AFF',
    padding: 14,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 16
  },
  pending: { opacity: 0.7 },
  buttonText: { color: '#fff', fontSize: 16 },
  skeleton: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    gap: 8,
    height: 150,
    padding: 8
  },
  skeletonBar: {
    flex: 1,
    height: 100,
    backgroundColor: '#e0e0e0',
    borderRadius: 4
  }
});

export default SmartScreen;
```

---

## 3. Bundle Analysis

### 3.1 Metro Bundle Analyzer

```bash
# Generate bundle stats
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/main.bundle \
  --sourcemap-output /tmp/main.bundle.map

# Analyze bundle
npx source-map-explorer /tmp/main.bundle /tmp/main.bundle.map
```

### 3.2 Custom Bundle Analyzer

```typescript
// BundleAnalyzer.ts
import fs from 'fs';
import path from 'path';

interface ModuleSize {
  name: string;
  size: number;
  percentage: number;
}

async function analyzeBundleSize(bundlePath: string): Promise<ModuleSize[]> {
  const content = fs.readFileSync(bundlePath, 'utf8');
  const totalSize = Buffer.byteLength(content, 'utf8');
  
  // Simple analysis (production tools are more sophisticated)
  const moduleRegex = /\/\* ([^*]+) \*\//g;
  const modules: Map<string, number> = new Map();
  
  let match;
  let lastIndex = 0;
  let lastName = '';
  
  while ((match = moduleRegex.exec(content)) !== null) {
    if (lastName) {
      const size = match.index - lastIndex;
      modules.set(lastName, (modules.get(lastName) || 0) + size);
    }
    lastName = match[1];
    lastIndex = match.index;
  }
  
  return Array.from(modules.entries())
    .map(([name, size]) => ({
      name,
      size,
      percentage: (size / totalSize) * 100
    }))
    .sort((a, b) => b.size - a.size)
    .slice(0, 20);
}
```

---

## 4. RAM Bundles

### 4.1 การ Enable RAM Bundles

```javascript
// metro.config.js
module.exports = {
  serializer: {
    // Enable RAM bundles for better startup
    processModuleFilter: (module) => {
      // ไม่ใส่ node_modules บางตัวใน initial bundle
      if (module.path.includes('node_modules/lodash')) {
        return false;
      }
      return true;
    }
  },
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        inlineRequires: true, // สำคัญสำหรับ RAM bundles
      },
    }),
  }
};
```

### 4.2 Indexed RAM Bundle

```javascript
// package.json scripts
{
  "scripts": {
    "bundle:ios": "react-native bundle --platform ios --dev false --entry-file index.js --bundle-output ios/main.jsbundle --indexed-ram-bundle",
    "bundle:android": "react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/app/src/main/assets/index.android.bundle --indexed-ram-bundle"
  }
}
```

---

## 5. Workshop: Optimize Bundle

### Step 1: วิเคราะห์ Bundle Size

```bash
# ดู bundle size ปัจจุบัน
npx react-native bundle \
  --platform ios \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/bundle.js \
  --sourcemap-output /tmp/bundle.js.map

# ดูขนาด
du -sh /tmp/bundle.js

# Analyze
npx source-map-explorer /tmp/bundle.js /tmp/bundle.js.map
```

### Step 2: Lazy Load Heavy Modules

```typescript
// BeforeOptimization.ts - ❌
import moment from 'moment'; // 300KB!
import lodash from 'lodash'; // 500KB!
import * as icons from 'react-native-vector-icons'; // Heavy!

// AfterOptimization.ts - ✅
// Use lightweight alternatives or dynamic imports

// แทน moment ด้วย date-fns (tree-shakeable)
import { format, parseISO } from 'date-fns';

// แทน lodash.get ด้วย native
const get = (obj: any, path: string, defaultValue?: any) => {
  return path.split('.').reduce((acc, key) => acc?.[key], obj) ?? defaultValue;
};

// Dynamic import สำหรับ rarely used features
const loadMoment = () => import('moment');
const loadHeavyLibrary = () => import('./heavyLibrary');
```

### Step 3: Tree Shaking Configuration

```javascript
// babel.config.js
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: [
    // Enable tree shaking
    ['babel-plugin-transform-imports', {
      'lodash': {
        transform: 'lodash/${member}',
        preventFullImport: true
      }
    }],
    // Remove unused imports in production
    ...(process.env.NODE_ENV === 'production' ? [
      ['babel-plugin-preval', {}]
    ] : [])
  ]
};
```

### Step 4: Implement Lazy Routes

```tsx
// LazyRouter.tsx
import React, { Suspense, lazy, useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  BackHandler,
  Alert
} from 'react-native';
import { useEffect } from 'react';

// ✅ Lazy load screens
const screens = {
  home: lazy(() => import('./screens/Home')),
  products: lazy(() => import('./screens/Products')),
  cart: lazy(() => import('./screens/Cart')),
  checkout: lazy(() => import('./screens/Checkout')),
  profile: lazy(() => import('./screens/Profile')),
  settings: lazy(() => import('./screens/Settings')),
  help: lazy(() => import('./screens/Help'))
};

type ScreenKey = keyof typeof screens;

const LoadingFallback: React.FC<{ name: string }> = ({ name }) => (
  <View style={styles.loading}>
    <ActivityIndicator size="large" color="#007AFF" />
    <Text style={styles.loadingText}>กำลังโหลด {name}...</Text>
  </View>
);

const LazyRouter: React.FC = () => {
  const [currentScreen, setCurrentScreen] = useState<ScreenKey>('home');
  const [history, setHistory] = useState<ScreenKey[]>(['home']);
  
  const navigate = useCallback((screen: ScreenKey) => {
    setCurrentScreen(screen);
    setHistory(prev => [...prev, screen]);
  }, []);
  
  const goBack = useCallback(() => {
    if (history.length > 1) {
      const newHistory = history.slice(0, -1);
      setHistory(newHistory);
      setCurrentScreen(newHistory[newHistory.length - 1]);
      return true;
    }
    return false;
  }, [history]);
  
  useEffect(() => {
    const backHandler = BackHandler.addEventListener(
      'hardwareBackPress',
      goBack
    );
    return () => backHandler.remove();
  }, [goBack]);
  
  const Screen = screens[currentScreen];
  
  return (
    <View style={styles.container}>
      <Suspense fallback={<LoadingFallback name={currentScreen} />}>
        <Screen navigate={navigate} goBack={goBack} />
      </Suspense>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  loading: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    gap: 12
  },
  loadingText: {
    fontSize: 14,
    color: '#666'
  }
});

export default LazyRouter;
```

### Step 5: Measure Improvements

```typescript
// BundleMetrics.ts
interface BundleMetrics {
  totalSize: number;
  initialChunk: number;
  lazyChunks: number;
  startupTime: number;
}

export class BundleOptimizer {
  private startTime: number = performance.now();
  
  measureStartup(): number {
    const time = performance.now() - this.startTime;
    console.log(`App startup time: ${time.toFixed(0)}ms`);
    return time;
  }
  
  async measureChunkLoad(chunkName: string, loader: () => Promise<any>): Promise<number> {
    const start = performance.now();
    await loader();
    const time = performance.now() - start;
    console.log(`Chunk "${chunkName}" loaded in ${time.toFixed(0)}ms`);
    return time;
  }
  
  reportMetrics(metrics: Partial<BundleMetrics>) {
    console.group('Bundle Metrics');
    if (metrics.totalSize) {
      console.log(`Total bundle: ${(metrics.totalSize / 1024).toFixed(1)}KB`);
    }
    if (metrics.initialChunk) {
      console.log(`Initial chunk: ${(metrics.initialChunk / 1024).toFixed(1)}KB`);
    }
    if (metrics.startupTime) {
      console.log(`Startup time: ${metrics.startupTime.toFixed(0)}ms`);
    }
    console.groupEnd();
  }
}
```

---

## 6. Inline Requires

```javascript
// metro.config.js
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        // Inline requires - โหลด module เมื่อ function ถูกเรียก ไม่ใช่ตอน import
        inlineRequires: true,
        // หรือ exclude บาง modules
        inlineRequires: {
          blockList: {
            // ไม่ inline require libraries นี้
            [require.resolve('./src/App')]: false
          }
        }
      },
    }),
  },
};
```

---

## Tips สำหรับ Code Splitting

1. **Analyze ก่อน** - วิเคราะห์ bundle ก่อน optimize
2. **Measure ก่อนหลัง** - วัดผลเสมอ ไม่ใช่แค่ assume
3. **Lazy load screens** - screen ที่ไม่ได้ใช้บ่อยควร lazy load
4. **Preload intelligently** - preload content ที่คาดว่าจะดูถัดไป
5. **Keep initial bundle small** - <500KB สำหรับ initial chunk

## สรุป

Code Splitting และ Lazy Loading ช่วย:
1. ลด initial bundle size
2. เพิ่มความเร็ว startup
3. ประหยัด memory ใน device
4. ปรับปรุง user experience โดยรวม
