# Part 077: Performance Profiling

## ทำไมต้อง Profile?

Performance Profiling คือกระบวนการวัดและวิเคราะห์ performance ของแอป เพื่อหา bottlenecks และปรับปรุงประสบการณ์ผู้ใช้ การ profile อย่างสม่ำเสมอช่วยให้เราทราบว่าแอปช้าที่จุดไหนและควรปรับปรุงอย่างไร

---

## 1. React DevTools Profiler

### 1.1 การเปิดใช้งาน

```bash
# ติดตั้ง React DevTools
npm install -g react-devtools

# เปิดใช้งาน
react-devtools
```

### 1.2 การอ่านผลลัพธ์

```tsx
// ProfilerExample.tsx
import React, { Profiler, ProfilerOnRenderCallback } from 'react';

const onRenderCallback: ProfilerOnRenderCallback = (
  id,           // ชื่อ component
  phase,        // "mount" หรือ "update"
  actualDuration, // เวลาที่ใช้ render จริง
  baseDuration,   // เวลาที่ใช้ถ้าไม่มี memoization
  startTime,
  commitTime,
  interactions
) => {
  console.log(`Component: ${id}`);
  console.log(`Phase: ${phase}`);
  console.log(`Actual duration: ${actualDuration}ms`);
  console.log(`Base duration: ${baseDuration}ms`);
  
  // Warning ถ้า render ช้า
  if (actualDuration > 16) { // 16ms = 60fps threshold
    console.warn(`⚠️ Slow render: ${id} took ${actualDuration}ms`);
  }
};

const ProfiledApp: React.FC = () => {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <MainContent />
    </Profiler>
  );
};

// Profile เฉพาะ component ที่สนใจ
const SlowList: React.FC<{ data: any[] }> = ({ data }) => {
  return (
    <Profiler id="SlowList" onRender={onRenderCallback}>
      <ExpensiveComponent data={data} />
    </Profiler>
  );
};
```

### 1.3 Performance Logger

```typescript
// PerformanceLogger.ts
interface RenderMetric {
  componentId: string;
  phase: 'mount' | 'update';
  actualDuration: number;
  baseDuration: number;
  timestamp: number;
}

class PerformanceLogger {
  private metrics: RenderMetric[] = [];
  private threshold = 16; // ms (60fps)
  
  log(
    id: string,
    phase: 'mount' | 'update',
    actualDuration: number,
    baseDuration: number
  ) {
    const metric: RenderMetric = {
      componentId: id,
      phase,
      actualDuration,
      baseDuration,
      timestamp: Date.now()
    };
    
    this.metrics.push(metric);
    
    if (actualDuration > this.threshold) {
      this.reportSlowRender(metric);
    }
  }
  
  private reportSlowRender(metric: RenderMetric) {
    console.warn(
      `[Performance] Slow render: ${metric.componentId}`,
      `\n  Phase: ${metric.phase}`,
      `\n  Actual: ${metric.actualDuration.toFixed(2)}ms`,
      `\n  Base: ${metric.baseDuration.toFixed(2)}ms`,
      `\n  Savings from memoization: ${(metric.baseDuration - metric.actualDuration).toFixed(2)}ms`
    );
  }
  
  getSummary() {
    const grouped = this.metrics.reduce((acc, metric) => {
      if (!acc[metric.componentId]) {
        acc[metric.componentId] = {
          count: 0,
          totalActual: 0,
          totalBase: 0,
          maxActual: 0
        };
      }
      
      const entry = acc[metric.componentId];
      entry.count++;
      entry.totalActual += metric.actualDuration;
      entry.totalBase += metric.baseDuration;
      entry.maxActual = Math.max(entry.maxActual, metric.actualDuration);
      
      return acc;
    }, {} as Record<string, any>);
    
    return Object.entries(grouped)
      .map(([id, data]) => ({
        id,
        renderCount: data.count,
        avgActual: data.totalActual / data.count,
        avgBase: data.totalBase / data.count,
        maxActual: data.maxActual,
        memoEfficiency: ((data.totalBase - data.totalActual) / data.totalBase * 100).toFixed(1) + '%'
      }))
      .sort((a, b) => b.avgActual - a.avgActual);
  }
  
  clear() {
    this.metrics = [];
  }
}

export const perfLogger = new PerformanceLogger();
```

---

## 2. Custom Performance Monitor

```tsx
// PerformanceMonitor.tsx
import React, { useState, useEffect, useRef } from 'react';
import {
  View,
  Text,
  StyleSheet,
  Animated,
  TouchableOpacity
} from 'react-native';

interface FrameData {
  fps: number;
  jsThreadBusy: boolean;
  uiThreadBusy: boolean;
}

const PerformanceOverlay: React.FC = () => {
  const [fps, setFps] = useState(60);
  const [visible, setVisible] = useState(true);
  const frameCount = useRef(0);
  const lastTimestamp = useRef(performance.now());
  const animFrame = useRef<number>();
  
  useEffect(() => {
    const measureFPS = (timestamp: number) => {
      frameCount.current++;
      
      const elapsed = timestamp - lastTimestamp.current;
      if (elapsed >= 1000) {
        const currentFPS = Math.round(
          frameCount.current * (1000 / elapsed)
        );
        setFps(currentFPS);
        frameCount.current = 0;
        lastTimestamp.current = timestamp;
      }
      
      animFrame.current = requestAnimationFrame(measureFPS);
    };
    
    animFrame.current = requestAnimationFrame(measureFPS);
    
    return () => {
      if (animFrame.current) {
        cancelAnimationFrame(animFrame.current);
      }
    };
  }, []);
  
  const getFPSColor = () => {
    if (fps >= 55) return '#34C759';
    if (fps >= 40) return '#FF9500';
    return '#FF3B30';
  };
  
  if (!visible) return null;
  
  return (
    <TouchableOpacity
      style={styles.overlay}
      onPress={() => setVisible(false)}
    >
      <Text style={[styles.fpsText, { color: getFPSColor() }]}>
        {fps} FPS
      </Text>
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  overlay: {
    position: 'absolute',
    top: 50,
    right: 10,
    backgroundColor: 'rgba(0,0,0,0.7)',
    padding: 8,
    borderRadius: 6,
    zIndex: 9999
  },
  fpsText: {
    fontFamily: 'monospace',
    fontSize: 14,
    fontWeight: 'bold'
  }
});

export default PerformanceOverlay;
```

---

## 3. JavaScript Thread Optimization

### 3.1 การหา JS Thread Bottlenecks

```typescript
// JSThreadProfiler.ts
export class JSThreadProfiler {
  private tasks: Map<string, number> = new Map();
  
  startTask(name: string) {
    this.tasks.set(name, performance.now());
  }
  
  endTask(name: string) {
    const start = this.tasks.get(name);
    if (start === undefined) return 0;
    
    const duration = performance.now() - start;
    this.tasks.delete(name);
    
    if (duration > 5) { // >5ms is significant on JS thread
      console.warn(`[JS Thread] Task "${name}" took ${duration.toFixed(2)}ms`);
    }
    
    return duration;
  }
  
  measureSync<T>(name: string, fn: () => T): T {
    this.startTask(name);
    const result = fn();
    this.endTask(name);
    return result;
  }
  
  async measureAsync<T>(name: string, fn: () => Promise<T>): Promise<T> {
    this.startTask(name);
    try {
      const result = await fn();
      this.endTask(name);
      return result;
    } catch (error) {
      this.endTask(name);
      throw error;
    }
  }
}

export const jsProfiler = new JSThreadProfiler();
```

### 3.2 Expensive Computation Optimization

```typescript
// HeavyComputation.ts
import { InteractionManager } from 'react-native';

// ❌ บล็อก JS thread
function processDataSync(data: number[]): number {
  return data.reduce((sum, n) => sum + Math.sqrt(n), 0);
}

// ✅ ทำหลัง interactions เสร็จ
async function processDataDeferred(data: number[]): Promise<number> {
  return new Promise((resolve) => {
    InteractionManager.runAfterInteractions(() => {
      const result = data.reduce((sum, n) => sum + Math.sqrt(n), 0);
      resolve(result);
    });
  });
}

// ✅ แบ่งงานเป็น chunks
async function processDataChunked(
  data: number[],
  chunkSize = 1000
): Promise<number> {
  let total = 0;
  
  for (let i = 0; i < data.length; i += chunkSize) {
    const chunk = data.slice(i, i + chunkSize);
    
    // Process chunk
    total += chunk.reduce((sum, n) => sum + Math.sqrt(n), 0);
    
    // ปล่อย JS thread ชั่วคราว
    await new Promise(resolve => setTimeout(resolve, 0));
  }
  
  return total;
}

// ✅ ใช้ Web Worker (ถ้า available)
async function processDataWorker(data: number[]): Promise<number> {
  // React Native ไม่มี Web Workers แต่ใช้ libraries เช่น react-native-threads
  return processDataChunked(data);
}
```

---

## 4. UI Thread Optimization

### 4.1 Animated API Performance

```tsx
// AnimationOptimized.tsx
import React, { useRef, useEffect } from 'react';
import { Animated, View, StyleSheet } from 'react-native';

const OptimizedAnimation: React.FC = () => {
  const scaleAnim = useRef(new Animated.Value(1)).current;
  const opacityAnim = useRef(new Animated.Value(1)).current;
  
  useEffect(() => {
    // ✅ useNativeDriver: true - runs on UI thread
    const animation = Animated.loop(
      Animated.sequence([
        Animated.parallel([
          Animated.spring(scaleAnim, {
            toValue: 1.2,
            useNativeDriver: true // สำคัญมาก!
          }),
          Animated.timing(opacityAnim, {
            toValue: 0.5,
            duration: 500,
            useNativeDriver: true
          })
        ]),
        Animated.parallel([
          Animated.spring(scaleAnim, {
            toValue: 1,
            useNativeDriver: true
          }),
          Animated.timing(opacityAnim, {
            toValue: 1,
            duration: 500,
            useNativeDriver: true
          })
        ])
      ])
    );
    
    animation.start();
    return () => animation.stop();
  }, []);
  
  return (
    <Animated.View
      style={[
        styles.box,
        {
          transform: [{ scale: scaleAnim }],
          opacity: opacityAnim
        }
      ]}
    />
  );
};

// ❌ ทำให้ใช้ JS thread
// const badAnimation = Animated.timing(value, {
//   toValue: 1,
//   useNativeDriver: false  // ใช้ JS thread = ช้ากว่า
// });

const styles = StyleSheet.create({
  box: {
    width: 100,
    height: 100,
    backgroundColor: '#007AFF',
    borderRadius: 8
  }
});

export default OptimizedAnimation;
```

### 4.2 Reanimated 2 for Better Performance

```tsx
// ReanimatedExample.tsx
import React from 'react';
import { View, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  runOnJS,
  useAnimatedGestureHandler
} from 'react-native-reanimated';
import { PanGestureHandler } from 'react-native-gesture-handler';

const DraggableCard: React.FC = () => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);
  
  const gestureHandler = useAnimatedGestureHandler({
    onStart: (_, ctx: any) => {
      ctx.startX = translateX.value;
      ctx.startY = translateY.value;
      scale.value = withSpring(1.1);
    },
    onActive: (event, ctx) => {
      translateX.value = ctx.startX + event.translationX;
      translateY.value = ctx.startY + event.translationY;
    },
    onEnd: () => {
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
      scale.value = withSpring(1);
    }
  });
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value }
    ]
  }));
  
  return (
    <PanGestureHandler onGestureEvent={gestureHandler}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <View style={styles.cardContent} />
      </Animated.View>
    </PanGestureHandler>
  );
};

const styles = StyleSheet.create({
  card: {
    width: 200,
    height: 120,
    backgroundColor: '#007AFF',
    borderRadius: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    elevation: 8
  },
  cardContent: {
    flex: 1,
    borderRadius: 12
  }
});

export default DraggableCard;
```

---

## 5. FlatList Optimization

```tsx
// OptimizedFlatList.tsx
import React, { useCallback, useMemo } from 'react';
import {
  FlatList,
  View,
  Text,
  StyleSheet,
  ListRenderItem
} from 'react-native';

interface Item {
  id: string;
  title: string;
  subtitle: string;
  imageUrl: string;
}

// ✅ Extract ItemComponent เพื่อลด re-renders
const ListItem = React.memo<{ item: Item; onPress: (id: string) => void }>(
  ({ item, onPress }) => {
    return (
      <View style={styles.item}>
        <Text style={styles.title}>{item.title}</Text>
        <Text style={styles.subtitle}>{item.subtitle}</Text>
      </View>
    );
  },
  (prev, next) => prev.item.id === next.item.id // Custom comparison
);

const OptimizedFlatList: React.FC<{ data: Item[] }> = ({ data }) => {
  // ✅ useCallback เพื่อไม่ re-create function
  const keyExtractor = useCallback((item: Item) => item.id, []);
  
  const handlePress = useCallback((id: string) => {
    console.log('Pressed:', id);
  }, []);
  
  // ✅ renderItem ต้อง stable
  const renderItem: ListRenderItem<Item> = useCallback(
    ({ item }) => <ListItem item={item} onPress={handlePress} />,
    [handlePress]
  );
  
  // ✅ getItemLayout ถ้า item มี fixed height
  const getItemLayout = useCallback(
    (_: any, index: number) => ({
      length: 80,
      offset: 80 * index,
      index
    }),
    []
  );
  
  return (
    <FlatList
      data={data}
      keyExtractor={keyExtractor}
      renderItem={renderItem}
      getItemLayout={getItemLayout}
      
      // Optimization props
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      updateCellsBatchingPeriod={100}
      initialNumToRender={10}
      windowSize={10}
      
      // For large lists
      legacyImplementation={false}
    />
  );
};

const styles = StyleSheet.create({
  item: {
    height: 80,
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
    justifyContent: 'center'
  },
  title: {
    fontSize: 16,
    fontWeight: '600',
    marginBottom: 4
  },
  subtitle: {
    fontSize: 14,
    color: '#666'
  }
});

export default OptimizedFlatList;
```

---

## 6. Memory Profiling

```typescript
// MemoryMonitor.ts
import { Platform } from 'react-native';

export class MemoryMonitor {
  private snapshots: Array<{
    timestamp: number;
    usedHeap?: number;
    label: string;
  }> = [];
  
  takeSnapshot(label: string) {
    const snapshot = {
      timestamp: Date.now(),
      label,
      usedHeap: undefined as number | undefined
    };
    
    // ถ้า available
    if (typeof (performance as any).memory !== 'undefined') {
      snapshot.usedHeap = (performance as any).memory.usedJSHeapSize;
    }
    
    this.snapshots.push(snapshot);
    
    if (snapshot.usedHeap) {
      console.log(
        `[Memory] ${label}: ${(snapshot.usedHeap / 1024 / 1024).toFixed(2)}MB`
      );
    }
    
    return snapshot;
  }
  
  compareSnapshots(label1: string, label2: string) {
    const s1 = this.snapshots.find(s => s.label === label1);
    const s2 = this.snapshots.find(s => s.label === label2);
    
    if (!s1 || !s2 || !s1.usedHeap || !s2.usedHeap) {
      console.log('Cannot compare - missing heap data');
      return;
    }
    
    const diff = s2.usedHeap - s1.usedHeap;
    const diffMB = (diff / 1024 / 1024).toFixed(2);
    
    console.log(`Memory change from "${label1}" to "${label2}": ${diffMB}MB`);
    
    if (diff > 10 * 1024 * 1024) { // >10MB increase
      console.warn(`⚠️ Large memory increase detected: ${diffMB}MB`);
    }
    
    return { diff, diffMB };
  }
  
  detectLeaks() {
    if (this.snapshots.length < 3) return;
    
    const recent = this.snapshots.slice(-5);
    const hasHeapData = recent.every(s => s.usedHeap !== undefined);
    
    if (!hasHeapData) return;
    
    const trend = recent.reduce((acc, snapshot, i) => {
      if (i === 0) return acc;
      const prev = recent[i - 1];
      return acc + ((snapshot.usedHeap! - prev.usedHeap!) > 0 ? 1 : -1);
    }, 0);
    
    if (trend >= 4) { // Consistently growing
      console.error('🚨 Possible memory leak detected! Memory is consistently growing.');
    }
  }
}

export const memoryMonitor = new MemoryMonitor();
```

---

## Workshop: Profile และ Optimize

### Step 1: Setup Profiling

```tsx
// ProfilingApp.tsx
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  FlatList,
  StyleSheet,
  ScrollView
} from 'react-native';
import { perfLogger } from './JSThreadProfiler';
import { memoryMonitor } from './MemoryMonitor';

const ITEM_COUNT = 500;

interface DataItem {
  id: string;
  value: number;
  computed: number;
}

const generateData = (count: number): DataItem[] => {
  return Array.from({ length: count }, (_, i) => ({
    id: `item-${i}`,
    value: Math.random() * 1000,
    computed: 0
  }));
};

const ProcessedItem = React.memo<{ item: DataItem }>(({ item }) => (
  <View style={styles.item}>
    <Text>{item.id}: {item.computed.toFixed(2)}</Text>
  </View>
));

const ProfiledScreen: React.FC = () => {
  const [data, setData] = useState<DataItem[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [metrics, setMetrics] = useState<string[]>([]);
  
  const addMetric = useCallback((text: string) => {
    setMetrics(prev => [`[${new Date().toISOString().substr(11, 8)}] ${text}`, ...prev.slice(0, 19)]);
  }, []);
  
  const runTest = useCallback(async () => {
    setIsLoading(true);
    memoryMonitor.takeSnapshot('before-generate');
    addMetric('Starting test...');
    
    const duration = await perfLogger.measureAsync('generateData', async () => {
      const rawData = generateData(ITEM_COUNT);
      
      // Process in chunks
      const processed: DataItem[] = [];
      const chunkSize = 50;
      
      for (let i = 0; i < rawData.length; i += chunkSize) {
        const chunk = rawData.slice(i, i + chunkSize);
        chunk.forEach(item => {
          item.computed = Math.sqrt(item.value) * Math.PI;
        });
        processed.push(...chunk);
        
        // Yield to JS thread
        await new Promise(r => setTimeout(r, 0));
      }
      
      return processed;
    });
    
    addMetric(`Data processing: ${duration.toFixed(1)}ms`);
    memoryMonitor.takeSnapshot('after-generate');
    
    const comparison = memoryMonitor.compareSnapshots(
      'before-generate',
      'after-generate'
    );
    
    if (comparison) {
      addMetric(`Memory delta: ${comparison.diffMB}MB`);
    }
    
    setIsLoading(false);
  }, [addMetric]);
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Performance Profiler</Text>
      
      <TouchableOpacity
        style={styles.button}
        onPress={runTest}
        disabled={isLoading}
      >
        <Text style={styles.buttonText}>
          {isLoading ? 'Running...' : 'Run Benchmark'}
        </Text>
      </TouchableOpacity>
      
      <ScrollView style={styles.metricsContainer}>
        {metrics.map((metric, i) => (
          <Text key={i} style={styles.metric}>{metric}</Text>
        ))}
      </ScrollView>
      
      <FlatList
        data={data}
        keyExtractor={item => item.id}
        renderItem={({ item }) => <ProcessedItem item={item} />}
        maxToRenderPerBatch={10}
        windowSize={5}
        removeClippedSubviews={true}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 20, fontWeight: 'bold', marginBottom: 12 },
  button: {
    backgroundColor: '#007AFF',
    padding: 14,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 12
  },
  buttonText: { color: '#fff', fontSize: 16 },
  metricsContainer: {
    backgroundColor: '#1a1a1a',
    borderRadius: 8,
    padding: 8,
    maxHeight: 150,
    marginBottom: 12
  },
  metric: {
    color: '#00ff00',
    fontFamily: 'monospace',
    fontSize: 11,
    marginBottom: 2
  },
  item: {
    padding: 8,
    borderBottomWidth: 1,
    borderBottomColor: '#eee'
  }
});

export default ProfiledScreen;
```

---

## Tips สำหรับ Performance

### Quick Wins
1. ใช้ `useNativeDriver: true` กับทุก animation
2. `React.memo` กับ `useCallback` สำหรับ pure components
3. `getItemLayout` ใน FlatList ถ้ารู้ขนาด item
4. `removeClippedSubviews` สำหรับ long lists
5. `InteractionManager.runAfterInteractions` สำหรับ heavy work

### Common Issues
- setState ใน loops → batch updates ด้วย unstable_batchedUpdates
- Anonymous functions ใน render → useCallback
- Inline styles ที่เปลี่ยนตลอด → StyleSheet.create
- Heavy computation บน JS thread → defer หรือ chunk

## สรุป

การ profile และ optimize React Native app ต้องทำอย่างต่อเนื่อง โดยวัดก่อนและหลังการปรับปรุงเสมอ ใช้ tools ที่เหมาะสมสำหรับแต่ละปัญหา เช่น React DevTools สำหรับ component renders, Systrace สำหรับ Android UI, และ Instruments สำหรับ iOS
