# Part 076: New Architecture ของ React Native

## New Architecture คืออะไร?

New Architecture ของ React Native ประกอบด้วยการเปลี่ยนแปลงสำคัญ 3 อย่าง:
1. **JSI** (JavaScript Interface) - แทนที่ Bridge
2. **Fabric** - Renderer ใหม่
3. **Turbo Modules** - Native Modules ใหม่

---

## 1. Bridge vs Bridgeless

### Legacy Bridge Architecture

```
┌─────────────────────────────────────────────────────┐
│                   JavaScript Thread                   │
│  ┌──────────────────────────────────────────────┐   │
│  │          React Component Tree                │   │
│  └──────────────────────┬───────────────────────┘   │
│                          │                            │
└──────────────────────────┼────────────────────────────┘
                           │ JSON serialization (slow)
                           ↓
┌─────────────────────────────────────────────────────┐
│                     Bridge Queue                      │
│  (asynchronous, batched, potential bottleneck)        │
└─────────────────────────┬───────────────────────────┘
                           │
┌──────────────────────────┼────────────────────────────┐
│                   Native Thread                        │
│  ┌──────────────────────────────────────────────┐   │
│  │              Native Modules                  │   │
│  │              Native Views                    │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### New Architecture (Bridgeless)

```
┌─────────────────────────────────────────────────────┐
│                   JavaScript Thread                   │
│  ┌──────────────────────────────────────────────┐   │
│  │     React + Fabric Renderer                  │   │
│  └──────────────────────┬───────────────────────┘   │
│                          │                            │
│                     JSI (C++)                        │
│                     ↓       ↓                         │
│               Turbo Modules  Fabric                   │
│                          │                            │
└──────────────────────────┼────────────────────────────┘
                           │ direct C++ calls (fast)
┌──────────────────────────┼────────────────────────────┐
│                   Native Thread                        │
│  ┌──────────────────────────────────────────────┐   │
│  │         Turbo Native Modules                 │   │
│  │         Fabric Native Views                  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## 2. Migration Guide

### Step 1: ตรวจสอบ Dependencies

```bash
# ตรวจสอบ libraries ที่ใช้ว่ารองรับ New Architecture
npx react-native-libraries --check-new-architecture

# หรือตรวจสอบ manually
cat package.json | grep -E '"(react-native|@react-native)'
```

### Step 2: อัปเดต React Native

```bash
# อัปเดต React Native เป็น version ที่รองรับ
npx react-native upgrade

# ตรวจสอบ version
npx react-native --version
```

### Step 3: Enable New Architecture

```bash
# สร้างโปรเจกต์ใหม่ด้วย New Architecture
npx react-native@latest init MyApp --version latest
```

```properties
# android/gradle.properties
newArchEnabled=true
hermesEnabled=true
```

```ruby
# ios/Podfile
ENV['RCT_NEW_ARCH_ENABLED'] = '1'

platform :ios, '13.4'
use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => true,
  :fabric_enabled => true,
  :flipper_configuration => FlipperConfiguration.disabled
)
```

---

## 3. Compatibility Layer

React Native มี compatibility layer ที่ช่วยให้ legacy modules ทำงานได้บน New Architecture

### 3.1 Legacy Module Compatibility

```kotlin
// Android: Ensure legacy modules work
// build.gradle
android {
  defaultConfig {
    // ถ้า library ยังไม่รองรับ New Architecture
    buildConfigField "boolean", "IS_NEW_ARCHITECTURE_ENABLED", "true"
  }
}
```

```typescript
// JavaScript compatibility check
import { TurboModuleRegistry, NativeModules } from 'react-native';

function getModule(name: string) {
  // ลอง Turbo Module ก่อน
  const turboModule = TurboModuleRegistry.get(name);
  if (turboModule) return turboModule;
  
  // Fallback ไป Legacy Module
  return NativeModules[name];
}

export const CalendarModule = getModule('CalendarModule');
```

### 3.2 Interop Layer สำหรับ Third-party Libraries

```typescript
// InteropLayer.ts
import { Platform } from 'react-native';

const isNewArchEnabled = () => {
  try {
    const { TurboModuleRegistry } = require('react-native');
    return TurboModuleRegistry.get('SomeModule') !== null;
  } catch {
    return false;
  }
};

export function createModuleAdapter<T extends object>(
  moduleName: string,
  fallback: T
): T {
  try {
    const { TurboModuleRegistry } = require('react-native');
    const turboModule = TurboModuleRegistry.get<any>(moduleName);
    if (turboModule) return turboModule as T;
  } catch {}
  
  try {
    const { NativeModules } = require('react-native');
    if (NativeModules[moduleName]) return NativeModules[moduleName] as T;
  } catch {}
  
  return fallback;
}
```

---

## 4. Breaking Changes

### 4.1 Removed APIs

```typescript
// ❌ Deprecated/Removed
import { BackAndroid } from 'react-native'; // ใช้ BackHandler แทน
import { NetInfo } from 'react-native'; // ใช้ @react-native-community/netinfo แทน
import { DatePickerIOS } from 'react-native'; // ใช้ DateTimePicker แทน

// ✅ New alternatives
import { BackHandler } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import DateTimePicker from '@react-native-community/datetimepicker';
```

### 4.2 Prop Changes

```tsx
// ❌ Old
<TextInput
  autoCompleteType="email" // deprecated
  keyboardAppearance="dark"
/>

// ✅ New
<TextInput
  autoComplete="email" // renamed
  keyboardAppearance="dark"
/>
```

### 4.3 Style Changes

```typescript
// ❌ Old shadow props
const styles = StyleSheet.create({
  shadow: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.3,
    shadowRadius: 4,
    elevation: 5 // Android only
  }
});

// ✅ New (with boxShadow support)
const styles = StyleSheet.create({
  shadow: {
    // iOS
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.3,
    shadowRadius: 4,
    // Android
    elevation: 5,
    // Both platforms (when supported)
    boxShadow: '0px 2px 4px rgba(0, 0, 0, 0.3)'
  }
});
```

---

## 5. Workshop: Migrate Existing App

### Step 1: Analysis Script

```bash
#!/bin/bash
# check-new-arch-compatibility.sh

echo "Checking New Architecture compatibility..."

# Check React Native version
RN_VERSION=$(node -e "console.log(require('./node_modules/react-native/package.json').version)")
echo "React Native version: $RN_VERSION"

# Check for deprecated imports
echo -e "\nChecking for deprecated imports..."
grep -r "BackAndroid\|NetInfo\|DatePickerIOS\|SliderIOS\|PickerIOS" src/ --include="*.ts" --include="*.tsx" 2>/dev/null

# Check for bridge-specific code
echo -e "\nChecking for bridge-specific patterns..."
grep -r "NativeModules\|NativeEventEmitter" src/ --include="*.ts" --include="*.tsx" 2>/dev/null

echo "Analysis complete!"
```

### Step 2: สร้าง Migration Wrapper

```typescript
// migration/ModuleWrapper.ts
import { NativeModules, TurboModuleRegistry } from 'react-native';

type ModuleSource = 'turbo' | 'legacy' | 'none';

interface ModuleInfo {
  module: any;
  source: ModuleSource;
}

export function getModuleWithInfo(name: string): ModuleInfo {
  // Try Turbo Module first
  try {
    const turboModule = TurboModuleRegistry.get(name);
    if (turboModule) {
      return { module: turboModule, source: 'turbo' };
    }
  } catch {}
  
  // Fallback to Legacy
  const legacyModule = NativeModules[name];
  if (legacyModule) {
    return { module: legacyModule, source: 'legacy' };
  }
  
  return { module: null, source: 'none' };
}

// Migration logger
export function logMigrationStatus() {
  const modules = [
    'CalendarModule',
    'BiometricModule',
    'NetworkModule',
    'StorageModule'
  ];
  
  console.log('=== Module Migration Status ===');
  modules.forEach(name => {
    const { source } = getModuleWithInfo(name);
    const status = source === 'turbo' ? '✅ Turbo' :
                   source === 'legacy' ? '⚠️  Legacy' : '❌ Missing';
    console.log(`${name}: ${status}`);
  });
}
```

### Step 3: Update Native Modules

```typescript
// Before migration (Legacy)
// storage/LegacyStorage.ts
import { NativeModules } from 'react-native';

const { StorageModule } = NativeModules;

export const Storage = {
  get: (key: string): Promise<string | null> => {
    return new Promise((resolve, reject) => {
      StorageModule.getItem(key, (error: any, value: string) => {
        if (error) reject(error);
        else resolve(value);
      });
    });
  },
  set: (key: string, value: string): Promise<void> => {
    return new Promise((resolve, reject) => {
      StorageModule.setItem(key, value, (error: any) => {
        if (error) reject(error);
        else resolve();
      });
    });
  }
};
```

```typescript
// After migration (Turbo Module)
// storage/NativeStorageSpec.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getItem(key: string): Promise<string | null>;
  setItem(key: string, value: string): Promise<void>;
  removeItem(key: string): Promise<void>;
  getAllKeys(): Promise<string[]>;
  multiGet(keys: string[]): Promise<[string, string | null][]>;
  multiSet(keyValuePairs: [string, string][]): Promise<void>;
  clear(): Promise<void>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('StorageModule');
```

### Step 4: Testing after Migration

```typescript
// __tests__/migration.test.ts
import { TurboModuleRegistry, NativeModules } from 'react-native';

jest.mock('react-native', () => ({
  TurboModuleRegistry: {
    get: jest.fn(),
    getEnforcing: jest.fn()
  },
  NativeModules: {
    StorageModule: {
      getItem: jest.fn(),
      setItem: jest.fn()
    }
  }
}));

describe('Module Migration', () => {
  it('should use Turbo Module when available', () => {
    const mockTurboModule = {
      getItem: jest.fn().mockResolvedValue('value'),
      setItem: jest.fn().mockResolvedValue(undefined)
    };
    
    (TurboModuleRegistry.get as jest.Mock).mockReturnValue(mockTurboModule);
    
    const { getModuleWithInfo } = require('./migration/ModuleWrapper');
    const { module, source } = getModuleWithInfo('StorageModule');
    
    expect(source).toBe('turbo');
    expect(module).toBe(mockTurboModule);
  });
  
  it('should fallback to Legacy Module', () => {
    (TurboModuleRegistry.get as jest.Mock).mockReturnValue(null);
    
    const { getModuleWithInfo } = require('./migration/ModuleWrapper');
    const { module, source } = getModuleWithInfo('StorageModule');
    
    expect(source).toBe('legacy');
  });
});
```

---

## 6. Performance Improvements

```tsx
// PerformanceComparison.tsx
import React, { useState, useCallback } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

interface Metrics {
  renderCount: number;
  avgRenderTime: number;
  lastRenderTime: number;
}

const PerformanceMonitor: React.FC = () => {
  const [metrics, setMetrics] = useState<Metrics>({
    renderCount: 0,
    avgRenderTime: 0,
    lastRenderTime: 0
  });
  
  const renderStart = React.useRef<number>(0);
  
  React.useEffect(() => {
    renderStart.current = performance.now();
  });
  
  React.useEffect(() => {
    const renderTime = performance.now() - renderStart.current;
    setMetrics(prev => ({
      renderCount: prev.renderCount + 1,
      lastRenderTime: Math.round(renderTime * 100) / 100,
      avgRenderTime: Math.round(
        ((prev.avgRenderTime * prev.renderCount + renderTime) / 
        (prev.renderCount + 1)) * 100
      ) / 100
    }));
  });
  
  return (
    <View style={styles.container}>
      <Text style={styles.metric}>Renders: {metrics.renderCount}</Text>
      <Text style={styles.metric}>Last: {metrics.lastRenderTime}ms</Text>
      <Text style={styles.metric}>Avg: {metrics.avgRenderTime}ms</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    padding: 8,
    backgroundColor: '#000',
    opacity: 0.8
  },
  metric: {
    color: '#0f0',
    fontFamily: 'monospace',
    fontSize: 12
  }
});

export default PerformanceMonitor;
```

---

## 7. Checklist for Migration

```markdown
## New Architecture Migration Checklist

### Before Migration
- [ ] Update React Native to latest stable
- [ ] Update all dependencies
- [ ] Run tests baseline
- [ ] Document all native modules used

### iOS Changes
- [ ] Update Podfile with new arch flags
- [ ] Run `pod install --repo-update`
- [ ] Update native modules to use JSI/Codegen
- [ ] Test on multiple iOS versions

### Android Changes
- [ ] Update gradle.properties
- [ ] Update build.gradle
- [ ] Update native modules to Kotlin/Java for TurboModules
- [ ] Test on multiple Android versions

### Code Changes
- [ ] Update deprecated API usage
- [ ] Migrate to Turbo Modules where needed
- [ ] Update tests for new architecture
- [ ] Performance testing

### QA
- [ ] Run full test suite
- [ ] Performance profiling
- [ ] Memory leak detection
- [ ] Device compatibility testing
```

---

## 8. Troubleshooting

### 8.1 Common Issues

```typescript
// ปัญหา: Module ไม่พบ
// สาเหตุ: ยังไม่ได้ register Turbo Module
// แก้ไข: ตรวจสอบว่า module registered ใน MainApplication

// ปัญหา: TypeScript errors จาก CodeGen
// สาเหตุ: Spec file ไม่ถูกต้อง
// แก้ไข: ตรวจสอบ TurboModule.Spec interface ให้ครบถ้วน

// ปัญหา: Fabric render ผิดพลาด
// สาเหตุ: Component ไม่รองรับ Fabric
// แก้ไข: ใช้ Fabric-compatible libraries หรือ fallback

// Debug New Architecture issues
import { TurboModuleRegistry } from 'react-native';

function debugModuleRegistration() {
  const moduleNames = [
    'CalendarModule',
    'BiometricModule',
    'NetworkModule'
  ];
  
  moduleNames.forEach(name => {
    const turbo = TurboModuleRegistry.get(name);
    console.log(`${name}: ${turbo ? 'Turbo ✅' : 'Not found ❌'}`);
  });
}
```

### 8.2 การตรวจสอบ New Architecture Status

```typescript
// utils/architectureCheck.ts
import { TurboModuleRegistry, Platform } from 'react-native';

export function getArchitectureInfo(): {
  isNewArchEnabled: boolean;
  platform: string;
  reactNativeVersion: string;
  hermes: boolean;
} {
  const isHermes = !!(global as any).HermesInternal;
  
  // ทดสอบ Turbo Module availability
  let isNewArch = false;
  try {
    // Turbo Modules สามารถ get ได้แบบ synchronous ใน New Arch
    const testModule = TurboModuleRegistry.get('SomeTestModule');
    isNewArch = true; // ถ้าไม่ throw แสดงว่า New Arch enabled
  } catch {
    isNewArch = false;
  }
  
  const rnVersion = require('react-native/package.json').version;
  
  return {
    isNewArchEnabled: isNewArch,
    platform: Platform.OS,
    reactNativeVersion: rnVersion,
    hermes: isHermes
  };
}

// Display Component
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

export const ArchitectureInfoPanel: React.FC = () => {
  const info = getArchitectureInfo();
  
  return (
    <View style={styles.panel}>
      <Text style={styles.title}>Architecture Info</Text>
      <Row label="New Architecture" value={info.isNewArchEnabled ? '✅ Enabled' : '❌ Disabled'} />
      <Row label="Hermes Engine" value={info.hermes ? '✅ Active' : '⚠️ JSC'} />
      <Row label="Platform" value={info.platform} />
      <Row label="RN Version" value={info.reactNativeVersion} />
    </View>
  );
};

const Row: React.FC<{ label: string; value: string }> = ({ label, value }) => (
  <View style={styles.row}>
    <Text style={styles.label}>{label}:</Text>
    <Text style={styles.value}>{value}</Text>
  </View>
);

const styles = StyleSheet.create({
  panel: {
    backgroundColor: '#1a1a2e',
    borderRadius: 12,
    padding: 16,
    margin: 16
  },
  title: {
    color: '#fff',
    fontSize: 16,
    fontWeight: 'bold',
    marginBottom: 10
  },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    paddingVertical: 4
  },
  label: { color: '#aaa', fontSize: 14 },
  value: { color: '#fff', fontSize: 14, fontWeight: '500' }
});
```

---

## 9. Tips สำหรับ New Architecture

### DO's
1. ใช้ TypeScript สำหรับทุก Native Module spec
2. Test บน physical device เสมอ (ไม่ใช่แค่ simulator)
3. Enable Hermes พร้อมกับ New Architecture
4. ตรวจสอบ third-party library compatibility ก่อน
5. ทำ performance benchmarks ก่อนและหลัง migration

### DON'Ts
1. อย่า migrate production app ทันทีโดยไม่ทดสอบ
2. อย่าใช้ eval() หรือ dynamic code execution
3. อย่าลืม unsubscribe จาก event listeners
4. อย่า mix legacy และ new architecture module patterns
5. อย่าข้ามขั้นตอนใน migration guide

---

## สรุป

New Architecture ของ React Native เป็นการเปลี่ยนแปลงครั้งใหญ่ที่:

1. **ปรับปรุง Performance** - JSI ไม่มี JSON serialization overhead
2. **Concurrent Mode** - Fabric รองรับ React 18 Concurrent
3. **Type Safety** - CodeGen สร้าง type-safe interfaces
4. **Lazy Loading** - Turbo Modules โหลดเมื่อจำเป็น
5. **Better Interop** - ทำงานร่วมกับ native code ได้ดีขึ้น

การ migrate ต้องทำอย่างระมัดระวัง ตรวจสอบ dependencies และทดสอบอย่างละเอียดก่อน release
แนะนำให้เริ่มจาก project ใหม่ที่ใช้ New Architecture ตั้งแต่ต้น แล้วค่อยย้าย logic จาก project เดิม
