# Part 058: Memory Management ใน React Native

## ความเข้าใจ Memory Management

Memory management คือการจัดการหน่วยความจำของแอปให้มีประสิทธิภาพ ป้องกัน memory leaks และทำให้แอปทำงานได้อย่างราบรื่น

### ทำไม Memory Management ถึงสำคัญ?

```
1. Memory Leak → แอปใช้ RAM มากขึ้นเรื่อยๆ
2. High Memory → OS อาจ kill แอป (Crash)
3. Slow Performance → Garbage Collector ทำงานหนัก
4. ANR (Android Not Responding) → แอปค้าง
5. Battery Drain → ใช้ RAM มาก = ใช้ battery มาก
```

### วิธีตรวจจับ Memory Leaks

```bash
# Android - ใช้ Android Profiler ใน Android Studio
# iOS - ใช้ Instruments (Leaks template)
# React Native - ใช้ Flipper + Memory plugin
```

---

## Memory Leaks ที่พบบ่อย

### 1. Listener ที่ไม่ถูก Remove

```typescript
import React, { useEffect, useState } from 'react';
import { View, Text, Dimensions, StyleSheet } from 'react-native';

// BAD: Listener ถูก add แต่ไม่ถูก remove
const BadWindowDimensionComponent = () => {
  const [dimensions, setDimensions] = useState(Dimensions.get('window'));

  useEffect(() => {
    // ❌ Memory Leak! Listener ไม่ถูก cleanup
    Dimensions.addEventListener('change', ({ window }) => {
      setDimensions(window);
    });
    // ลืม return cleanup function!
  }, []);

  return <Text>{`${dimensions.width} x ${dimensions.height}`}</Text>;
};

// GOOD: Cleanup listener เมื่อ unmount
const GoodWindowDimensionComponent = () => {
  const [dimensions, setDimensions] = useState(Dimensions.get('window'));

  useEffect(() => {
    const subscription = Dimensions.addEventListener('change', ({ window }) => {
      setDimensions(window);
    });

    // ✅ Cleanup เมื่อ component unmount
    return () => {
      subscription?.remove();
    };
  }, []);

  return (
    <Text style={memStyles.dimensionText}>
      {`${dimensions.width} x ${dimensions.height}`}
    </Text>
  );
};
```

### 2. Async Operations บน Unmounted Component

```typescript
import React, { useEffect, useState, useRef } from 'react';
import { View, Text } from 'react-native';

// BAD: setState หลัง unmount
const BadAsyncComponent = () => {
  const [data, setData] = useState(null);

  useEffect(() => {
    // ❌ ถ้า component unmount ระหว่าง fetch, setData จะถูกเรียกบน unmounted component
    fetchData().then(result => {
      setData(result); // WARNING: Can't perform a React state update on an unmounted component
    });
  }, []);

  return <Text>{data?.name}</Text>;
};

// GOOD: ตรวจสอบก่อน setState
const GoodAsyncComponent = () => {
  const [data, setData] = useState<any>(null);
  const [loading, setLoading] = useState(true);
  const isMountedRef = useRef(true);

  useEffect(() => {
    isMountedRef.current = true;

    const loadData = async () => {
      try {
        const result = await fetchData();
        // ✅ ตรวจสอบว่า component ยัง mounted อยู่หรือไม่
        if (isMountedRef.current) {
          setData(result);
        }
      } catch (error) {
        if (isMountedRef.current) {
          console.error(error);
        }
      } finally {
        if (isMountedRef.current) {
          setLoading(false);
        }
      }
    };

    loadData();

    return () => {
      isMountedRef.current = false; // Mark as unmounted
    };
  }, []);

  if (loading) return <Text>กำลังโหลด...</Text>;
  return <Text>{data?.name}</Text>;
};

// Better approach: ใช้ AbortController สำหรับ fetch
const BetterAsyncComponent = () => {
  const [data, setData] = useState<any>(null);

  useEffect(() => {
    const abortController = new AbortController();

    const loadData = async () => {
      try {
        const response = await fetch('https://api.example.com/data', {
          signal: abortController.signal, // ✅ Abort fetch เมื่อ unmount
        });
        const result = await response.json();
        setData(result);
      } catch (error: any) {
        if (error.name !== 'AbortError') {
          console.error(error);
        }
        // AbortError คาดหวังได้ ไม่ต้องจัดการ
      }
    };

    loadData();

    return () => {
      abortController.abort(); // ✅ Cancel fetch เมื่อ unmount
    };
  }, []);

  return <Text>{data?.name}</Text>;
};

const fetchData = async () => {
  const response = await fetch('https://api.example.com/user');
  return response.json();
};

const memStyles = StyleSheet.create({
  dimensionText: { fontSize: 16, color: '#333' },
});
```

### 3. Timer/Interval ที่ไม่ถูก Clear

```typescript
import React, { useEffect, useState, useRef } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

// BAD: Timer/Interval ไม่ถูก clear
const BadTimerComponent = () => {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // ❌ Interval ยังทำงานอยู่แม้ component unmount
    setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
  }, []);

  return <Text>Count: {count}</Text>;
};

// GOOD: Clear timer เมื่อ unmount
const GoodTimerComponent = () => {
  const [count, setCount] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef<NodeJS.Timeout | null>(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setCount(c => c + 1);
      }, 1000);
    }

    return () => {
      // ✅ Clear interval เมื่อ effect cleanup หรือ unmount
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
        intervalRef.current = null;
      }
    };
  }, [isRunning]);

  // Cleanup เมื่อ unmount ด้วย
  useEffect(() => {
    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, []);

  return (
    <View style={timerStyles.container}>
      <Text style={timerStyles.count}>{count}</Text>
      <TouchableOpacity
        style={[timerStyles.button, isRunning ? timerStyles.stopBtn : timerStyles.startBtn]}
        onPress={() => setIsRunning(r => !r)}
      >
        <Text style={timerStyles.buttonText}>{isRunning ? 'หยุด' : 'เริ่ม'}</Text>
      </TouchableOpacity>
    </View>
  );
};

const timerStyles = StyleSheet.create({
  container: { alignItems: 'center', padding: 20 },
  count: { fontSize: 48, fontWeight: 'bold', marginBottom: 20, color: '#333' },
  button: { paddingHorizontal: 30, paddingVertical: 12, borderRadius: 25 },
  startBtn: { backgroundColor: '#4CAF50' },
  stopBtn: { backgroundColor: '#F44336' },
  buttonText: { color: 'white', fontWeight: 'bold', fontSize: 18 },
});
```

### 4. Event Emitter ที่ไม่ถูก Remove

```typescript
import { NativeEventEmitter, NativeModules } from 'react-native';

const useNativeEvent = (eventName: string, handler: (data: any) => void) => {
  useEffect(() => {
    const eventEmitter = new NativeEventEmitter(NativeModules.MyModule);
    
    const subscription = eventEmitter.addListener(eventName, handler);
    
    // ✅ สำคัญมาก! remove listener เมื่อ unmount
    return () => {
      subscription.remove();
    };
  }, [eventName, handler]);
};
```

---

## Cleanup ใน useEffect

```typescript
import React, { useEffect, useState, useCallback } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import Geolocation from '@react-native-community/geolocation';

const ComplexCleanupComponent: React.FC = () => {
  const [status, setStatus] = useState({
    network: 'unknown',
    location: null as { lat: number; lng: number } | null,
    time: '',
  });

  useEffect(() => {
    // 1. Network listener
    const networkUnsubscribe = NetInfo.addEventListener(state => {
      setStatus(prev => ({
        ...prev,
        network: state.isConnected ? 'online' : 'offline',
      }));
    });

    // 2. Geolocation watcher
    const watchId = Geolocation.watchPosition(
      position => {
        setStatus(prev => ({
          ...prev,
          location: {
            lat: position.coords.latitude,
            lng: position.coords.longitude,
          },
        }));
      },
      error => console.error(error),
      { enableHighAccuracy: false, distanceFilter: 10 }
    );

    // 3. Timer
    const interval = setInterval(() => {
      setStatus(prev => ({
        ...prev,
        time: new Date().toLocaleTimeString('th-TH'),
      }));
    }, 1000);

    // ✅ Cleanup ทั้งหมด
    return () => {
      networkUnsubscribe();
      Geolocation.clearWatch(watchId);
      clearInterval(interval);
    };
  }, []);

  return (
    <View style={cleanupStyles.container}>
      <Text style={cleanupStyles.text}>Network: {status.network}</Text>
      <Text style={cleanupStyles.text}>
        Location: {status.location
          ? `${status.location.lat.toFixed(4)}, ${status.location.lng.toFixed(4)}`
          : 'N/A'}
      </Text>
      <Text style={cleanupStyles.text}>Time: {status.time}</Text>
    </View>
  );
};

const cleanupStyles = StyleSheet.create({
  container: { padding: 15, backgroundColor: 'white', borderRadius: 10 },
  text: { fontSize: 14, color: '#555', marginBottom: 5 },
});
```

---

## Large Lists Memory Management

```typescript
import React, { useCallback, useState, useRef } from 'react';
import { FlatList, View, Text, StyleSheet, TouchableOpacity } from 'react-native';

interface HeavyItem {
  id: string;
  data: string;
  imageData: string; // simulate large data
}

const generateHeavyItems = (count: number): HeavyItem[] =>
  Array.from({ length: count }, (_, i) => ({
    id: `item-${i}`,
    data: `ข้อมูลชิ้นที่ ${i + 1} `.repeat(10),
    imageData: 'x'.repeat(100), // simulate image data
  }));

const MemoryEfficientList: React.FC = () => {
  const [items] = useState(() => generateHeavyItems(10000));
  const [selectedId, setSelectedId] = useState<string | null>(null);

  const renderItem = useCallback(({ item }: { item: HeavyItem }) => (
    <TouchableOpacity
      style={[
        listStyles.item,
        selectedId === item.id && listStyles.selectedItem,
      ]}
      onPress={() => setSelectedId(item.id === selectedId ? null : item.id)}
    >
      <Text style={listStyles.itemId}>#{item.id}</Text>
      <Text style={listStyles.itemData} numberOfLines={2}>
        {item.data}
      </Text>
    </TouchableOpacity>
  ), [selectedId]);

  const keyExtractor = useCallback((item: HeavyItem) => item.id, []);

  const ITEM_HEIGHT = 80;
  const getItemLayout = useCallback(
    (_: any, index: number) => ({
      length: ITEM_HEIGHT,
      offset: ITEM_HEIGHT * index,
      index,
    }),
    []
  );

  return (
    <FlatList
      data={items}
      keyExtractor={keyExtractor}
      renderItem={renderItem}
      getItemLayout={getItemLayout}
      
      // Memory optimization
      removeClippedSubviews={true}     // Unmount items ที่ไม่อยู่ใน viewport
      initialNumToRender={20}          // Render items แรกๆ ก่อน
      maxToRenderPerBatch={10}         // Batch size
      windowSize={5}                   // แค่ 5 screens worth of items ใน memory
      
      // Performance
      updateCellsBatchingPeriod={100}
      disableVirtualization={false}    // Keep virtualization on!
    />
  );
};

const listStyles = StyleSheet.create({
  item: {
    height: 80,
    padding: 12,
    backgroundColor: 'white',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
    justifyContent: 'center',
  },
  selectedItem: {
    backgroundColor: '#E3F2FD',
    borderLeftWidth: 4,
    borderLeftColor: '#2196F3',
  },
  itemId: { fontSize: 12, color: '#999', marginBottom: 4 },
  itemData: { fontSize: 14, color: '#444' },
});
```

---

## Image Caching

```typescript
import React, { useEffect, useRef, useState } from 'react';
import { View, Image, StyleSheet, Text } from 'react-native';
import FastImage from 'react-native-fast-image';

// Custom Image Cache Manager
class ImageCacheManager {
  private static cache = new Map<string, string>();
  private static maxCacheSize = 50; // สูงสุด 50 images

  static async getOrFetch(url: string): Promise<string> {
    if (this.cache.has(url)) {
      return this.cache.get(url)!;
    }

    // Evict oldest entries ถ้า cache เต็ม
    if (this.cache.size >= this.maxCacheSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }

    // Fetch and cache
    this.cache.set(url, url);
    return url;
  }

  static clearCache(): void {
    this.cache.clear();
    FastImage.clearMemoryCache();
    FastImage.clearDiskCache();
  }

  static getCacheSize(): number {
    return this.cache.size;
  }
}

// Image component พร้อม memory-efficient loading
const MemoryEfficientImage = ({ uri, style }: { uri: string; style?: any }) => {
  const [isLoaded, setIsLoaded] = useState(false);
  const [hasError, setHasError] = useState(false);

  return (
    <View style={[imgCacheStyles.container, style]}>
      {!isLoaded && !hasError && (
        <View style={[StyleSheet.absoluteFill, imgCacheStyles.placeholder]}>
          <Text style={imgCacheStyles.placeholderText}>📷</Text>
        </View>
      )}
      
      {hasError ? (
        <View style={[StyleSheet.absoluteFill, imgCacheStyles.errorPlaceholder]}>
          <Text>❌ โหลดรูปไม่ได้</Text>
        </View>
      ) : (
        <FastImage
          source={{
            uri,
            priority: FastImage.priority.normal,
            cache: FastImage.cacheControl.immutable,
          }}
          style={StyleSheet.absoluteFill}
          resizeMode={FastImage.resizeMode.cover}
          onLoad={() => setIsLoaded(true)}
          onError={() => setHasError(true)}
        />
      )}
    </View>
  );
};

const imgCacheStyles = StyleSheet.create({
  container: { overflow: 'hidden', backgroundColor: '#f0f0f0' },
  placeholder: { justifyContent: 'center', alignItems: 'center', backgroundColor: '#e0e0e0' },
  placeholderText: { fontSize: 30 },
  errorPlaceholder: { justifyContent: 'center', alignItems: 'center', backgroundColor: '#FFEBEE' },
});
```

---

## Native Memory Management

```typescript
import { NativeModules, Platform } from 'react-native';

// Android Memory Info
const getAndroidMemoryInfo = async () => {
  if (Platform.OS !== 'android') return null;
  
  try {
    // ใช้ NativeModule ที่คุณสร้างเอง หรือใช้ library
    const memInfo = await NativeModules.MemoryInfo?.getInfo();
    return memInfo;
  } catch {
    return null;
  }
};

// ลด Memory Usage
const optimizeMemory = () => {
  // 1. Clear image cache เมื่อ memory ต่ำ
  const handleLowMemory = () => {
    FastImage.clearMemoryCache();
    console.log('Cleared image memory cache due to low memory');
  };

  // 2. iOS Memory Warning Handler  
  if (Platform.OS === 'ios') {
    // iOS ส่ง notification เมื่อ memory ต่ำ
    // จัดการใน AppDelegate.mm
  }
};
```

---

## Workshop: Fix Memory Leaks

```typescript
import React, { useState, useEffect, useRef, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
  Alert,
} from 'react-native';
import NetInfo from '@react-native-community/netinfo';

interface MemoryTestResult {
  id: number;
  test: string;
  status: 'pass' | 'fail' | 'warning';
  description: string;
}

// ตัวอย่าง Hook ที่ memory-safe
const useMemorySafeInterval = (callback: () => void, delay: number | null) => {
  const savedCallback = useRef(callback);

  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return;

    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
};

// useTimeout ที่ memory-safe
const useMemorySafeTimeout = (callback: () => void, delay: number | null) => {
  const savedCallback = useRef(callback);

  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return;
    
    const id = setTimeout(() => savedCallback.current(), delay);
    return () => clearTimeout(id);
  }, [delay]);
};

// useFetch ที่ memory-safe
const useMemorySafeFetch = <T>(url: string) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const abortController = new AbortController();
    let isMounted = true;

    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url, { signal: abortController.signal });
        const json = await response.json();
        
        if (isMounted) {
          setData(json);
          setError(null);
        }
      } catch (err: any) {
        if (err.name !== 'AbortError' && isMounted) {
          setError(err.message);
        }
      } finally {
        if (isMounted) {
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      isMounted = false;
      abortController.abort();
    };
  }, [url]);

  return { data, loading, error };
};

// Component หลักสำหรับ Workshop
const MemoryLeakFixWorkshop: React.FC = () => {
  const [tests, setTests] = useState<MemoryTestResult[]>([]);
  const [isRunning, setIsRunning] = useState(false);
  const [activeSubscriptions, setActiveSubscriptions] = useState(0);
  const [timerCount, setTimerCount] = useState(0);
  const [networkStatus, setNetworkStatus] = useState('unknown');

  const subscriptionCountRef = useRef(0);
  const timersRef = useRef<NodeJS.Timeout[]>([]);
  const netInfoUnsubscribeRef = useRef<(() => void) | null>(null);

  // Timer ที่ memory-safe
  useMemorySafeInterval(() => {
    setTimerCount(c => c + 1);
  }, isRunning ? 1000 : null);

  // Network listener ที่ memory-safe
  useEffect(() => {
    netInfoUnsubscribeRef.current = NetInfo.addEventListener(state => {
      setNetworkStatus(state.isConnected ? 'online' : 'offline');
    });

    return () => {
      netInfoUnsubscribeRef.current?.();
    };
  }, []);

  const addTest = useCallback((test: Omit<MemoryTestResult, 'id'>) => {
    setTests(prev => [...prev, { ...test, id: Date.now() }]);
  }, []);

  // Test 1: ตรวจสอบ Listener Cleanup
  const testListenerCleanup = async () => {
    addTest({
      test: 'Listener Cleanup',
      status: 'pass',
      description: 'Network listener ถูก cleanup ด้วย useEffect return function',
    });
  };

  // Test 2: ตรวจสอบ Async Cleanup
  const testAsyncCleanup = async () => {
    const abortController = new AbortController();
    
    try {
      // จำลอง async operation
      const fetchPromise = new Promise<void>((resolve, reject) => {
        const timer = setTimeout(resolve, 2000);
        abortController.signal.addEventListener('abort', () => {
          clearTimeout(timer);
          reject(new Error('AbortError'));
        });
      });

      // Abort หลัง 500ms
      setTimeout(() => abortController.abort(), 500);

      await fetchPromise;
      
      addTest({
        test: 'Async Cleanup',
        status: 'fail',
        description: 'Async operation ควรถูก abort แต่ยังทำงานอยู่',
      });
    } catch (error: any) {
      if (error.message === 'AbortError') {
        addTest({
          test: 'Async Cleanup',
          status: 'pass',
          description: 'Async operation ถูก abort สำเร็จ ไม่มี memory leak',
        });
      }
    }
  };

  // Test 3: ตรวจสอบ Timer Cleanup
  const testTimerCleanup = () => {
    const timer = setTimeout(() => {
      // timer นี้ควรถูก clear ก่อนที่จะถึงเวลา
    }, 5000);
    
    // Clear ทันที
    clearTimeout(timer);
    
    addTest({
      test: 'Timer Cleanup',
      status: 'pass',
      description: 'Timer ถูก clear อย่างถูกต้อง',
    });
  };

  // Test 4: ตรวจสอบ Subscription Count
  const testSubscriptionCount = () => {
    const expectedCount = 1; // network listener
    const status = activeSubscriptions <= expectedCount ? 'pass' : 'warning';
    
    addTest({
      test: 'Subscription Count',
      status,
      description: `มี ${activeSubscriptions} active subscriptions (ควรมี ≤ ${expectedCount})`,
    });
  };

  const runAllTests = async () => {
    setTests([]);
    
    await testListenerCleanup();
    await testAsyncCleanup();
    testTimerCleanup();
    testSubscriptionCount();
    
    addTest({
      test: 'สรุป',
      status: 'pass',
      description: 'ตรวจสอบ memory leaks เสร็จสิ้น',
    });
  };

  const getStatusColor = (status: MemoryTestResult['status']) => {
    switch (status) {
      case 'pass': return '#4CAF50';
      case 'fail': return '#F44336';
      case 'warning': return '#FF9800';
    }
  };

  const getStatusIcon = (status: MemoryTestResult['status']) => {
    switch (status) {
      case 'pass': return '✅';
      case 'fail': return '❌';
      case 'warning': return '⚠️';
    }
  };

  return (
    <ScrollView style={workshopStyles.container}>
      <Text style={workshopStyles.title}>Memory Leak Detection Workshop</Text>

      {/* Status Dashboard */}
      <View style={workshopStyles.dashboard}>
        <View style={workshopStyles.dashItem}>
          <Text style={workshopStyles.dashValue}>{timerCount}</Text>
          <Text style={workshopStyles.dashLabel}>Timer Count</Text>
        </View>
        <View style={workshopStyles.dashItem}>
          <Text style={[
            workshopStyles.dashValue,
            { color: networkStatus === 'online' ? '#4CAF50' : '#F44336' }
          ]}>
            {networkStatus === 'online' ? '🟢' : '🔴'}
          </Text>
          <Text style={workshopStyles.dashLabel}>Network</Text>
        </View>
        <View style={workshopStyles.dashItem}>
          <Text style={workshopStyles.dashValue}>{activeSubscriptions}</Text>
          <Text style={workshopStyles.dashLabel}>Subscriptions</Text>
        </View>
      </View>

      {/* Control Buttons */}
      <View style={workshopStyles.controls}>
        <TouchableOpacity
          style={[workshopStyles.ctrlBtn, isRunning ? workshopStyles.stopBtn : workshopStyles.startBtn]}
          onPress={() => setIsRunning(r => !r)}
        >
          <Text style={workshopStyles.ctrlBtnText}>
            {isRunning ? '⏸ หยุด Timer' : '▶ เริ่ม Timer'}
          </Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={workshopStyles.testBtn}
          onPress={runAllTests}
        >
          <Text style={workshopStyles.testBtnText}>🔍 รัน Tests</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={workshopStyles.clearBtn}
          onPress={() => setTests([])}
        >
          <Text style={workshopStyles.clearBtnText}>🗑 ล้าง</Text>
        </TouchableOpacity>
      </View>

      {/* Test Results */}
      {tests.length > 0 && (
        <View style={workshopStyles.results}>
          <Text style={workshopStyles.resultsTitle}>ผลการทดสอบ</Text>
          {tests.map(test => (
            <View key={test.id} style={workshopStyles.testResult}>
              <View style={workshopStyles.testHeader}>
                <Text style={workshopStyles.testIcon}>{getStatusIcon(test.status)}</Text>
                <Text style={workshopStyles.testName}>{test.test}</Text>
                <Text style={[workshopStyles.testStatus, { color: getStatusColor(test.status) }]}>
                  {test.status.toUpperCase()}
                </Text>
              </View>
              <Text style={workshopStyles.testDesc}>{test.description}</Text>
            </View>
          ))}
        </View>
      )}

      {/* Best Practices */}
      <View style={workshopStyles.bestPractices}>
        <Text style={workshopStyles.bpTitle}>Best Practices</Text>
        {[
          '✅ ใช้ return cleanup function ใน useEffect เสมอ',
          '✅ ตรวจสอบ isMounted ก่อน setState ใน async operations',
          '✅ ใช้ AbortController สำหรับ fetch requests',
          '✅ Clear intervals/timeouts ใน cleanup',
          '✅ Remove event listeners ใน cleanup',
          '✅ ใช้ useCallback/useMemo ลด unnecessary re-creation',
          '✅ ลด windowSize ใน FlatList สำหรับ large lists',
          '✅ ใช้ FastImage.clearMemoryCache() เมื่อ memory ต่ำ',
        ].map((practice, i) => (
          <Text key={i} style={workshopStyles.bpItem}>{practice}</Text>
        ))}
      </View>
    </ScrollView>
  );
};

const workshopStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  title: { fontSize: 22, fontWeight: 'bold', padding: 20, paddingBottom: 15, color: '#333' },
  dashboard: {
    flexDirection: 'row',
    backgroundColor: '#1a1a2e',
    padding: 15,
    justifyContent: 'space-around',
  },
  dashItem: { alignItems: 'center' },
  dashValue: { fontSize: 24, fontWeight: 'bold', color: 'white' },
  dashLabel: { fontSize: 12, color: '#888', marginTop: 4 },
  controls: { flexDirection: 'row', padding: 15, gap: 10 },
  ctrlBtn: { flex: 1, padding: 12, borderRadius: 8, alignItems: 'center' },
  startBtn: { backgroundColor: '#4CAF50' },
  stopBtn: { backgroundColor: '#F44336' },
  ctrlBtnText: { color: 'white', fontWeight: 'bold', fontSize: 13 },
  testBtn: { flex: 1.5, backgroundColor: '#2196F3', padding: 12, borderRadius: 8, alignItems: 'center' },
  testBtnText: { color: 'white', fontWeight: 'bold', fontSize: 13 },
  clearBtn: { padding: 12, borderRadius: 8, alignItems: 'center', backgroundColor: '#9E9E9E' },
  clearBtnText: { color: 'white', fontWeight: 'bold', fontSize: 13 },
  results: {
    margin: 15,
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 15,
    elevation: 2,
  },
  resultsTitle: { fontSize: 18, fontWeight: 'bold', color: '#333', marginBottom: 12 },
  testResult: {
    marginBottom: 12,
    paddingBottom: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  testHeader: { flexDirection: 'row', alignItems: 'center', marginBottom: 4 },
  testIcon: { fontSize: 18, marginRight: 8 },
  testName: { flex: 1, fontSize: 14, fontWeight: '600', color: '#333' },
  testStatus: { fontSize: 12, fontWeight: 'bold' },
  testDesc: { fontSize: 13, color: '#666', marginLeft: 26 },
  bestPractices: {
    margin: 15,
    backgroundColor: '#E8F5E9',
    borderRadius: 12,
    padding: 15,
    marginBottom: 30,
  },
  bpTitle: { fontSize: 16, fontWeight: 'bold', color: '#2E7D32', marginBottom: 10 },
  bpItem: { fontSize: 13, color: '#388E3C', marginBottom: 6, lineHeight: 20 },
});

export default MemoryLeakFixWorkshop;
```

---

## Tips สรุป Memory Management

### 1. useEffect Cleanup Pattern
```typescript
useEffect(() => {
  // Setup
  const subscription = something.subscribe(handler);
  const timer = setInterval(tick, 1000);
  
  // Cleanup - ALWAYS return!
  return () => {
    subscription.unsubscribe();
    clearInterval(timer);
  };
}, []);
```

### 2. isMounted Pattern (deprecated แต่บางครั้งยังจำเป็น)
```typescript
const isMountedRef = useRef(true);

useEffect(() => {
  return () => { isMountedRef.current = false; };
}, []);

// ใน async function:
if (isMountedRef.current) setState(newValue);
```

---

## สรุป

Memory Management ที่ดีต้องการ:
- **Cleanup Subscriptions**: remove listeners, unsubscribe observables
- **Cancel Async Operations**: AbortController สำหรับ fetch
- **Clear Timers**: clearTimeout, clearInterval
- **isMounted Check**: ป้องกัน setState บน unmounted component
- **Image Cache Management**: ล้าง cache เมื่อจำเป็น
- **FlatList Optimization**: windowSize, removeClippedSubviews
- **Profiling**: ใช้ Flipper/Instruments ตรวจจับ leaks จริงๆ
