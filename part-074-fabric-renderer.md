# Part 074: Fabric Renderer

## Fabric คืออะไร?

Fabric คือ rendering system ใหม่ของ React Native ที่แทนที่ legacy renderer เดิม โดยออกแบบมาเพื่อรองรับ React 18's Concurrent Mode และปรับปรุง performance ของ UI thread

## ปัญหาของ Legacy Renderer

1. **Async UI Updates**: ทุก UI update ต้องผ่าน bridge → ช้า
2. **No Synchronous Layout**: ไม่สามารถ read layout synchronously
3. **Poor Interop**: ทำงานร่วมกับ native views ได้ยาก
4. **Thread Issues**: UI thread และ JS thread communicate แบบ async

---

## 1. สถาปัตยกรรมของ Fabric

```
JavaScript Layer
      ↓
React Shadow Tree (C++)
      ↓
Yoga Layout Engine (C++)
      ↓
Native Views (iOS/Android)
```

### Shadow Tree คืออะไร?

```typescript
// Conceptual representation
interface FiberNode {
  type: string;
  props: Record<string, any>;
  children: FiberNode[];
  // Fabric-specific
  shadowNode: ShadowNode; // C++ object
}

interface ShadowNode {
  yogaNode: YogaNode;
  layoutMetrics: LayoutMetrics;
  viewProps: ViewProps;
}
```

---

## 2. Concurrent Mode

### 2.1 การใช้งาน Concurrent Features

```tsx
// ConcurrentApp.tsx
import React, { 
  useState, 
  useTransition, 
  useDeferredValue, 
  Suspense 
} from 'react';
import {
  View,
  Text,
  TextInput,
  FlatList,
  StyleSheet,
  ActivityIndicator
} from 'react-native';

// Simulated heavy component
const HeavyList: React.FC<{ items: string[] }> = ({ items }) => {
  return (
    <FlatList
      data={items}
      keyExtractor={(item, index) => `${item}-${index}`}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text>{item}</Text>
        </View>
      )}
    />
  );
};

const SearchScreen: React.FC = () => {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const [searchResults, setSearchResults] = useState<string[]>([]);
  
  // Deferred value ไม่ block UI
  const deferredQuery = useDeferredValue(query);
  
  const handleSearch = (text: string) => {
    setQuery(text); // Update input ทันที
    
    // ทำ search แบบ non-urgent
    startTransition(() => {
      const results = performHeavySearch(text);
      setSearchResults(results);
    });
  };
  
  return (
    <View style={styles.container}>
      <TextInput
        style={styles.input}
        value={query}
        onChangeText={handleSearch}
        placeholder="ค้นหา..."
      />
      
      {isPending && (
        <ActivityIndicator style={styles.loader} />
      )}
      
      <Suspense fallback={<ActivityIndicator />}>
        <HeavyList items={searchResults} />
      </Suspense>
    </View>
  );
};

// Heavy computation
function performHeavySearch(query: string): string[] {
  const allItems = Array.from({ length: 10000 }, (_, i) => `Item ${i}`);
  return allItems.filter(item => 
    item.toLowerCase().includes(query.toLowerCase())
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  input: {
    height: 44,
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    paddingHorizontal: 12,
    marginBottom: 8
  },
  loader: { marginVertical: 8 },
  item: {
    padding: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#eee'
  }
});

export default SearchScreen;
```

### 2.2 useTransition สำหรับ Navigation

```tsx
// NavigationWithTransition.tsx
import React, { useState, useTransition } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

type Screen = 'home' | 'profile' | 'settings';

const NavigationWithTransition: React.FC = () => {
  const [currentScreen, setCurrentScreen] = useState<Screen>('home');
  const [isPending, startTransition] = useTransition();
  
  const navigateTo = (screen: Screen) => {
    startTransition(() => {
      setCurrentScreen(screen);
    });
  };
  
  const renderScreen = () => {
    switch (currentScreen) {
      case 'home':
        return <HomeScreen />;
      case 'profile':
        return <ProfileScreen />;
      case 'settings':
        return <SettingsScreen />;
    }
  };
  
  return (
    <View style={styles.container}>
      <View style={[styles.content, isPending && styles.pending]}>
        {renderScreen()}
      </View>
      
      <View style={styles.tabBar}>
        {(['home', 'profile', 'settings'] as Screen[]).map(screen => (
          <TouchableOpacity
            key={screen}
            style={[
              styles.tab,
              currentScreen === screen && styles.activeTab
            ]}
            onPress={() => navigateTo(screen)}
          >
            <Text style={styles.tabText}>{screen}</Text>
          </TouchableOpacity>
        ))}
      </View>
    </View>
  );
};

const HomeScreen = () => <View><Text>Home</Text></View>;
const ProfileScreen = () => <View><Text>Profile</Text></View>;
const SettingsScreen = () => <View><Text>Settings</Text></View>;

const styles = StyleSheet.create({
  container: { flex: 1 },
  content: { flex: 1 },
  pending: { opacity: 0.7 },
  tabBar: {
    flexDirection: 'row',
    borderTopWidth: 1,
    borderTopColor: '#ccc'
  },
  tab: {
    flex: 1,
    padding: 16,
    alignItems: 'center'
  },
  activeTab: { backgroundColor: '#e8f0fe' },
  tabText: { fontSize: 14 }
});

export default NavigationWithTransition;
```

---

## 3. Synchronous Layout

### 3.1 การใช้ onLayout

```tsx
// SyncLayoutExample.tsx
import React, { useState, useCallback, useRef } from 'react';
import {
  View,
  Text,
  ScrollView,
  StyleSheet,
  LayoutChangeEvent,
  findNodeHandle
} from 'react-native';

interface LayoutInfo {
  width: number;
  height: number;
  x: number;
  y: number;
}

const SyncLayoutExample: React.FC = () => {
  const [containerLayout, setContainerLayout] = useState<LayoutInfo | null>(null);
  const [childLayouts, setChildLayouts] = useState<Record<string, LayoutInfo>>({});
  const containerRef = useRef<View>(null);
  
  const handleContainerLayout = useCallback((event: LayoutChangeEvent) => {
    const { width, height, x, y } = event.nativeEvent.layout;
    setContainerLayout({ width, height, x, y });
  }, []);
  
  const handleChildLayout = useCallback((id: string) => {
    return (event: LayoutChangeEvent) => {
      const { width, height, x, y } = event.nativeEvent.layout;
      setChildLayouts(prev => ({
        ...prev,
        [id]: { width, height, x, y }
      }));
    };
  }, []);
  
  const measureInWindow = () => {
    containerRef.current?.measureInWindow((x, y, width, height) => {
      console.log('Position in window:', { x, y, width, height });
    });
  };
  
  return (
    <ScrollView>
      <View
        ref={containerRef}
        style={styles.container}
        onLayout={handleContainerLayout}
      >
        {containerLayout && (
          <Text style={styles.info}>
            Container: {containerLayout.width.toFixed(0)}x{containerLayout.height.toFixed(0)}
          </Text>
        )}
        
        {['A', 'B', 'C'].map(id => (
          <View
            key={id}
            style={styles.child}
            onLayout={handleChildLayout(id)}
          >
            <Text>Child {id}</Text>
            {childLayouts[id] && (
              <Text style={styles.smallInfo}>
                {childLayouts[id].width.toFixed(0)}x{childLayouts[id].height.toFixed(0)}
              </Text>
            )}
          </View>
        ))}
      </View>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#f0f0f0'
  },
  info: {
    fontSize: 14,
    color: '#666',
    marginBottom: 8
  },
  child: {
    backgroundColor: '#fff',
    padding: 16,
    margin: 4,
    borderRadius: 8,
    borderWidth: 1,
    borderColor: '#ddd'
  },
  smallInfo: {
    fontSize: 12,
    color: '#999'
  }
});

export default SyncLayoutExample;
```

### 3.2 Fabric's Synchronous Measurement

```typescript
// FabricMeasurement.ts
import { UIManager, findNodeHandle } from 'react-native';

// ใน Fabric, การ measure เป็น synchronous
function measureSynchronous(ref: React.RefObject<any>) {
  const node = findNodeHandle(ref.current);
  
  if (!node) return null;
  
  // Fabric supports synchronous layout reads
  return new Promise<{x: number; y: number; width: number; height: number}>(
    (resolve) => {
      UIManager.measure(node, (x, y, width, height, pageX, pageY) => {
        resolve({ x: pageX, y: pageY, width, height });
      });
    }
  );
}

// Custom Hook สำหรับ auto measurement
import { useRef, useCallback, useState } from 'react';

export function useLayout() {
  const ref = useRef<any>(null);
  const [layout, setLayout] = useState<{
    x: number;
    y: number;
    width: number;
    height: number;
  } | null>(null);
  
  const measure = useCallback(async () => {
    const result = await measureSynchronous(ref);
    if (result) setLayout(result);
    return result;
  }, []);
  
  return { ref, layout, measure };
}
```

---

## 4. Component Lifecycle Changes

### 4.1 New Lifecycle

```tsx
// LifecycleExample.tsx
import React, { 
  useEffect, 
  useLayoutEffect, 
  useRef, 
  useState 
} from 'react';
import { View, Text, Animated } from 'react-native';

const LifecycleExample: React.FC = () => {
  const animValue = useRef(new Animated.Value(0)).current;
  const [mounted, setMounted] = useState(false);
  
  // useLayoutEffect - runs before paint (synchronous in Fabric)
  useLayoutEffect(() => {
    // Safe to measure here
    setMounted(true);
    
    return () => {
      // Cleanup before unmount
      setMounted(false);
    };
  }, []);
  
  // useEffect - runs after paint
  useEffect(() => {
    if (mounted) {
      Animated.spring(animValue, {
        toValue: 1,
        useNativeDriver: true
      }).start();
    }
    
    return () => {
      animValue.setValue(0);
    };
  }, [mounted]);
  
  return (
    <Animated.View
      style={{
        opacity: animValue,
        transform: [{
          translateY: animValue.interpolate({
            inputRange: [0, 1],
            outputRange: [20, 0]
          })
        }]
      }}
    >
      <Text>Animated with Fabric</Text>
    </Animated.View>
  );
};

export default LifecycleExample;
```

### 4.2 Fabric Component Registration (iOS)

```objectivec
// FabricViewManager.mm
#import <React/RCTViewManager.h>
#import <React/RCTFabricComponentsPlugins.h>

@interface CustomFabricViewManager : RCTViewManager
@end

@implementation CustomFabricViewManager

RCT_EXPORT_MODULE(CustomFabricView)

- (UIView *)view {
  return [[UIView alloc] init];
}

RCT_EXPORT_VIEW_PROPERTY(color, UIColor)
RCT_EXPORT_VIEW_PROPERTY(cornerRadius, CGFloat)

@end

// Fabric registration
Class<RCTComponentViewProtocol> CustomFabricViewCls(void) {
  return CustomFabricViewManager.class;
}
```

---

## 5. Workshop: Migrate to Fabric

### Step 1: ตรวจสอบ Compatibility

```bash
# ตรวจสอบว่า dependencies รองรับ Fabric
npx react-native info

# ตรวจสอบ libraries ที่ใช้
npx react-native-libraries --check-fabric
```

### Step 2: Enable Fabric ใน iOS

```ruby
# ios/Podfile
use_react_native!(
  :path => config[:reactNativePath],
  # เปิด New Architecture
  :fabric_enabled => true,
  :hermes_enabled => true
)
```

### Step 3: Enable Fabric ใน Android

```kotlin
// android/gradle.properties
newArchEnabled=true
hermesEnabled=true
```

### Step 4: สร้าง Fabric Native Component

```typescript
// NativeCustomView.ts - Spec สำหรับ Fabric Component
import type { ViewProps } from 'react-native';
import type { HostComponent } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

interface NativeProps extends ViewProps {
  color?: string;
  cornerRadius?: Float;
  onCustomEvent?: DirectEventHandler<Readonly<{
    value: string;
  }>>;
}

export default codegenNativeComponent<NativeProps>('CustomView') as HostComponent<NativeProps>;
```

### Step 5: iOS Native Implementation สำหรับ Fabric

```swift
// CustomViewComponentView.swift
import UIKit

@objc class CustomViewComponentView: RCTViewComponentView {
  
  private var contentView: UIView = UIView()
  
  required init?(coder: NSCoder) {
    fatalError("init(coder:) has not been implemented")
  }
  
  override init(frame: CGRect) {
    super.init(frame: frame)
    setupView()
  }
  
  private func setupView() {
    addSubview(contentView)
    contentView.translatesAutoresizingMaskIntoConstraints = false
    NSLayoutConstraint.activate([
      contentView.topAnchor.constraint(equalTo: topAnchor),
      contentView.leadingAnchor.constraint(equalTo: leadingAnchor),
      contentView.trailingAnchor.constraint(equalTo: trailingAnchor),
      contentView.bottomAnchor.constraint(equalTo: bottomAnchor)
    ])
  }
  
  // Update props
  override func updateProps(
    _ props: Props,
    oldProps: Props?
  ) {
    super.updateProps(props, oldProps: oldProps)
    
    guard let viewProps = props as? CustomViewProps else { return }
    
    if let colorString = viewProps.color {
      contentView.backgroundColor = UIColor(hexString: colorString)
    }
    
    contentView.layer.cornerRadius = CGFloat(viewProps.cornerRadius)
  }
  
  // Static registration
  @objc static func componentDescriptorProvider() -> ComponentDescriptorProvider {
    return concreteComponentDescriptorProvider<CustomViewComponentDescriptor>()
  }
}
```

### Step 6: JavaScript Usage

```tsx
// CustomViewComponent.tsx
import React from 'react';
import { StyleSheet, Platform } from 'react-native';
import CustomView from './NativeCustomView';

const CustomViewComponent: React.FC<{
  color?: string;
  cornerRadius?: number;
  onPress?: () => void;
}> = ({ color = '#007AFF', cornerRadius = 8, onPress }) => {
  return (
    <CustomView
      style={styles.view}
      color={color}
      cornerRadius={cornerRadius}
      onCustomEvent={(event) => {
        console.log('Native event:', event.nativeEvent);
        onPress?.();
      }}
    />
  );
};

const styles = StyleSheet.create({
  view: {
    width: 200,
    height: 100
  }
});

export default CustomViewComponent;
```

---

## Performance Comparison

```typescript
// BenchmarkExample.tsx
import React, { useState, useCallback, useRef } from 'react';
import { View, Text, FlatList, StyleSheet, TouchableOpacity } from 'react-native';

const ITEMS_COUNT = 1000;

const BenchmarkScreen: React.FC = () => {
  const [items, setItems] = useState<number[]>([]);
  const startTime = useRef<number>(0);
  const [renderTime, setRenderTime] = useState<number | null>(null);
  
  const runBenchmark = useCallback(() => {
    startTime.current = Date.now();
    const newItems = Array.from({ length: ITEMS_COUNT }, (_, i) => i);
    setItems(newItems);
  }, []);
  
  const handleRenderComplete = useCallback(() => {
    if (startTime.current > 0) {
      const elapsed = Date.now() - startTime.current;
      setRenderTime(elapsed);
      startTime.current = 0;
    }
  }, []);
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Fabric Benchmark</Text>
      
      {renderTime !== null && (
        <Text style={styles.result}>
          Render time: {renderTime}ms
        </Text>
      )}
      
      <TouchableOpacity style={styles.button} onPress={runBenchmark}>
        <Text style={styles.buttonText}>Run Benchmark ({ITEMS_COUNT} items)</Text>
      </TouchableOpacity>
      
      <FlatList
        data={items}
        keyExtractor={item => item.toString()}
        onLayout={handleRenderComplete}
        renderItem={({ item }) => (
          <View style={styles.item}>
            <Text>Item {item}</Text>
          </View>
        )}
        windowSize={5}
        maxToRenderPerBatch={20}
        updateCellsBatchingPeriod={100}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 20, fontWeight: 'bold', marginBottom: 8 },
  result: { fontSize: 16, color: '#34C759', marginBottom: 8 },
  button: {
    backgroundColor: '#007AFF',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 16
  },
  buttonText: { color: '#fff', fontSize: 16 },
  item: {
    padding: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#eee'
  }
});

export default BenchmarkScreen;
```

---

## Tips และ Best Practices สำหรับ Fabric

1. **ใช้ useLayoutEffect** แทน componentDidMount เพื่อ measurement
2. **Avoid inline styles** ที่ซับซ้อน - ใช้ StyleSheet.create
3. **Native Driver** สำหรับ animations ให้เร็วขึ้น
4. **Virtualization** สำหรับ long lists
5. **Memo/PureComponent** เพื่อลด re-renders

## สรุป

Fabric Renderer ช่วยให้ React Native:
- รองรับ React Concurrent Mode
- ปรับปรุง thread safety
- ทำ synchronous layout measurements
- เพิ่ม interoperability กับ native code
- ปรับปรุง performance โดยรวม
