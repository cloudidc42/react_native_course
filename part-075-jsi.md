# Part 075: JSI (JavaScript Interface)

## JSI คืออะไร?

JSI (JavaScript Interface) คือ C++ API ที่ช่วยให้ JavaScript engine สามารถ interact กับ C++ objects โดยตรงโดยไม่ต้องผ่าน bridge เดิมที่ต้องทำการ serialize/deserialize JSON ทุกครั้ง

## ปัญหาของ Legacy Bridge

```
ก่อน JSI:
JS Thread → serialize to JSON → Bridge Queue → deserialize → Native Thread
                                 (ช้า, blocking)

หลัง JSI:
JS Thread → C++ function call (shared memory) → Native Thread
                                (เร็ว, direct)
```

---

## 1. JSI Architecture

### 1.1 Core Concepts

```cpp
// jsi/jsi.h - Core JSI Types (simplified)
namespace facebook {
namespace jsi {

class Runtime {
public:
  // Execute JavaScript
  virtual Value evaluateJavaScript(
    const std::shared_ptr<const Buffer>& buffer,
    const std::string& sourceURL
  ) = 0;
  
  // Create objects
  virtual Object global() = 0;
};

class Value {
public:
  bool isNull() const;
  bool isUndefined() const;
  bool isString() const;
  bool isNumber() const;
  bool isObject() const;
  bool isBool() const;
  
  double asNumber() const;
  std::string asString(Runtime& runtime) const;
  Object asObject(Runtime& runtime) const;
  bool asBool() const;
};

class Object {
public:
  void setProperty(Runtime& runtime, const char* name, const Value& value);
  Value getProperty(Runtime& runtime, const char* name);
  bool isFunction(Runtime& runtime) const;
  bool isArray(Runtime& runtime) const;
};

class HostObject {
public:
  // Called when JS reads a property
  virtual Value get(Runtime& runtime, const PropNameID& name) = 0;
  
  // Called when JS writes a property
  virtual void set(Runtime& runtime, const PropNameID& name, const Value& value) = 0;
};

} // namespace jsi
} // namespace facebook
```

### 1.2 JSI vs Bridge Comparison

```typescript
// Performance benchmark concept
// Legacy Bridge
async function legacyBridgeCall() {
  // JS → serialize to JSON → async queue → native → serialize JSON → JS
  const result = await LegacyNativeModule.computeHeavy(data);
  return result;
}

// JSI
function jsiCall() {
  // JS → direct C++ call → return value (no serialization)
  const result = global.__nativeComputeHeavy(data);
  return result;
}
```

---

## 2. การสร้าง JSI Module

### 2.1 C++ Header

```cpp
// JSIExample.h
#pragma once

#include <ReactCommon/CallInvoker.h>
#include <jsi/jsi.h>
#include <memory>
#include <string>

using namespace facebook::jsi;

class JSIExample : public HostObject {
public:
  JSIExample(
    std::shared_ptr<CallInvoker> jsInvoker
  );
  
  // Called when JS reads property
  Value get(Runtime& runtime, const PropNameID& name) override;
  
  // Called when JS sets property
  void set(
    Runtime& runtime,
    const PropNameID& name,
    const Value& value
  ) override;
  
  // List all properties
  std::vector<PropNameID> getPropertyNames(Runtime& runtime) override;

private:
  std::shared_ptr<CallInvoker> jsInvoker_;
  
  // Native methods exposed to JS
  Value multiply(Runtime& runtime, const Value* args, size_t count);
  Value computeHash(Runtime& runtime, const Value* args, size_t count);
  Value processArray(Runtime& runtime, const Value* args, size_t count);
};
```

### 2.2 C++ Implementation

```cpp
// JSIExample.cpp
#include "JSIExample.h"
#include <stdexcept>
#include <functional>
#include <numeric>

JSIExample::JSIExample(
  std::shared_ptr<CallInvoker> jsInvoker
) : jsInvoker_(jsInvoker) {}

Value JSIExample::get(Runtime& runtime, const PropNameID& name) {
  std::string nameStr = name.utf8(runtime);
  
  if (nameStr == "multiply") {
    return Function::createFromHostFunction(
      runtime,
      name,
      2, // number of args
      [this](Runtime& rt, const Value& thisVal, const Value* args, size_t count) {
        return this->multiply(rt, args, count);
      }
    );
  }
  
  if (nameStr == "computeHash") {
    return Function::createFromHostFunction(
      runtime,
      name,
      1,
      [this](Runtime& rt, const Value& thisVal, const Value* args, size_t count) {
        return this->computeHash(rt, args, count);
      }
    );
  }
  
  if (nameStr == "processArray") {
    return Function::createFromHostFunction(
      runtime,
      name,
      1,
      [this](Runtime& rt, const Value& thisVal, const Value* args, size_t count) {
        return this->processArray(rt, args, count);
      }
    );
  }
  
  return Value::undefined();
}

void JSIExample::set(
  Runtime& runtime,
  const PropNameID& name,
  const Value& value
) {
  // Handle property setting if needed
}

std::vector<PropNameID> JSIExample::getPropertyNames(Runtime& runtime) {
  return {
    PropNameID::forAscii(runtime, "multiply"),
    PropNameID::forAscii(runtime, "computeHash"),
    PropNameID::forAscii(runtime, "processArray")
  };
}

Value JSIExample::multiply(Runtime& runtime, const Value* args, size_t count) {
  if (count < 2) {
    throw JSError(runtime, "multiply requires 2 arguments");
  }
  
  double a = args[0].asNumber();
  double b = args[1].asNumber();
  
  return Value(a * b);
}

Value JSIExample::computeHash(Runtime& runtime, const Value* args, size_t count) {
  if (count < 1 || !args[0].isString()) {
    throw JSError(runtime, "computeHash requires a string argument");
  }
  
  std::string input = args[0].asString(runtime).utf8(runtime);
  
  // Simple hash computation
  std::hash<std::string> hasher;
  size_t hashValue = hasher(input);
  
  return Value((double)hashValue);
}

Value JSIExample::processArray(Runtime& runtime, const Value* args, size_t count) {
  if (count < 1 || !args[0].isObject()) {
    throw JSError(runtime, "processArray requires an array argument");
  }
  
  auto arr = args[0].asObject(runtime).asArray(runtime);
  size_t length = arr.size(runtime);
  
  double sum = 0;
  for (size_t i = 0; i < length; i++) {
    Value item = arr.getValueAtIndex(runtime, i);
    if (item.isNumber()) {
      sum += item.asNumber();
    }
  }
  
  // Return object with results
  Object result(runtime);
  result.setProperty(runtime, "sum", Value(sum));
  result.setProperty(runtime, "count", Value((double)length));
  result.setProperty(runtime, "average", 
    length > 0 ? Value(sum / length) : Value(0.0));
  
  return result;
}
```

### 2.3 iOS Installation

```objectivec
// JSIExampleInstaller.h
#pragma once

#include <ReactCommon/CallInvoker.h>
#include <jsi/jsi.h>
#include <memory>

void installJSIExample(
  facebook::jsi::Runtime& runtime,
  std::shared_ptr<facebook::react::CallInvoker> jsCallInvoker
);
```

```objectivec
// JSIExampleInstaller.mm
#import "JSIExampleInstaller.h"
#import "JSIExample.h"

void installJSIExample(
  facebook::jsi::Runtime& runtime,
  std::shared_ptr<facebook::react::CallInvoker> jsCallInvoker
) {
  auto jsiExample = std::make_shared<JSIExample>(jsCallInvoker);
  
  runtime.global().setProperty(
    runtime,
    "__jsiExample",
    facebook::jsi::Object::createFromHostObject(runtime, jsiExample)
  );
}
```

```objectivec
// RCTJSIExampleModule.mm
#import <React/RCTBridge+Private.h>
#import <React/RCTUtils.h>
#import "RCTJSIExampleModule.h"
#import "JSIExampleInstaller.h"

@implementation RCTJSIExampleModule

RCT_EXPORT_MODULE(JSIExampleModule)

- (void)setBridge:(RCTBridge *)bridge {
  RCTCxxBridge *cxxBridge = (RCTCxxBridge *)bridge;
  if (!cxxBridge.runtime) {
    return;
  }
  
  installJSIExample(
    *(facebook::jsi::Runtime *)cxxBridge.runtime,
    bridge.jsCallInvoker
  );
}

@end
```

---

## 3. Android Installation

```kotlin
// JSIExamplePackage.kt
package com.myapp

import com.facebook.react.bridge.JSIModulePackage
import com.facebook.react.bridge.JSIModuleSpec
import com.facebook.react.bridge.JSIModuleType
import com.facebook.react.bridge.JavaScriptContextHolder
import com.facebook.react.bridge.ReactApplicationContext

class JSIExamplePackage : JSIModulePackage {
  override fun getJSIModules(
    reactApplicationContext: ReactApplicationContext,
    jsContext: JavaScriptContextHolder
  ): List<JSIModuleSpec<*>> {
    return listOf(
      JSIModuleSpec.of(
        JSIModuleType.Native,
        JSIModuleProvider { _, _ ->
          JSIExampleModule(reactApplicationContext)
        }
      )
    )
  }
}
```

```java
// JSIExampleModule.java
package com.myapp;

import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.jni.HybridData;
import com.facebook.react.turbomodule.core.CallInvokerHolderImpl;

public class JSIExampleModule {
  static {
    System.loadLibrary("jsiexample"); // Load native library
  }
  
  private final HybridData hybridData;
  
  public JSIExampleModule(ReactApplicationContext reactContext) {
    hybridData = initHybrid();
  }
  
  private native HybridData initHybrid();
  
  public native void install(long jsContextNativePointer);
}
```

---

## 4. การใช้งาน JSI ใน JavaScript

```typescript
// useJSIExample.ts
declare global {
  var __jsiExample: {
    multiply: (a: number, b: number) => number;
    computeHash: (input: string) => number;
    processArray: (arr: number[]) => {
      sum: number;
      count: number;
      average: number;
    };
  };
}

export function useJSIFunctions() {
  const isAvailable = typeof global.__jsiExample !== 'undefined';
  
  const multiply = (a: number, b: number): number => {
    if (!isAvailable) throw new Error('JSI not available');
    return global.__jsiExample.multiply(a, b);
  };
  
  const computeHash = (input: string): number => {
    if (!isAvailable) throw new Error('JSI not available');
    return global.__jsiExample.computeHash(input);
  };
  
  const processArray = (arr: number[]) => {
    if (!isAvailable) throw new Error('JSI not available');
    return global.__jsiExample.processArray(arr);
  };
  
  return {
    isAvailable,
    multiply,
    computeHash,
    processArray
  };
}
```

### 4.1 Performance Demo

```tsx
// JSIPerformanceDemo.tsx
import React, { useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView
} from 'react-native';
import { useJSIFunctions } from './useJSIExample';

interface BenchmarkResult {
  name: string;
  iterations: number;
  duration: number;
  opsPerSecond: number;
}

const JSIPerformanceDemo: React.FC = () => {
  const { isAvailable, multiply, computeHash, processArray } = useJSIFunctions();
  const [results, setResults] = useState<BenchmarkResult[]>([]);
  
  const runBenchmark = useCallback((
    name: string,
    fn: () => void,
    iterations: number
  ): BenchmarkResult => {
    const start = performance.now();
    for (let i = 0; i < iterations; i++) {
      fn();
    }
    const duration = performance.now() - start;
    return {
      name,
      iterations,
      duration: Math.round(duration * 100) / 100,
      opsPerSecond: Math.round(iterations / (duration / 1000))
    };
  }, []);
  
  const runAllBenchmarks = useCallback(() => {
    if (!isAvailable) return;
    
    const newResults: BenchmarkResult[] = [];
    
    // Multiply benchmark
    newResults.push(runBenchmark(
      'JSI multiply',
      () => multiply(Math.random(), Math.random()),
      100000
    ));
    
    // Hash benchmark
    newResults.push(runBenchmark(
      'JSI computeHash',
      () => computeHash('Hello World React Native JSI'),
      10000
    ));
    
    // Array processing benchmark
    const testArray = Array.from({ length: 100 }, () => Math.random() * 1000);
    newResults.push(runBenchmark(
      'JSI processArray (100 items)',
      () => processArray(testArray),
      1000
    ));
    
    // JavaScript equivalent for comparison
    newResults.push(runBenchmark(
      'JS multiply (comparison)',
      () => Math.random() * Math.random(),
      100000
    ));
    
    setResults(newResults);
  }, [isAvailable, multiply, computeHash, processArray, runBenchmark]);
  
  return (
    <ScrollView style={styles.container}>
      <Text style={styles.title}>JSI Performance Demo</Text>
      
      {!isAvailable && (
        <Text style={styles.warning}>JSI not available on this device</Text>
      )}
      
      <TouchableOpacity
        style={[styles.button, !isAvailable && styles.buttonDisabled]}
        onPress={runAllBenchmarks}
        disabled={!isAvailable}
      >
        <Text style={styles.buttonText}>Run Benchmarks</Text>
      </TouchableOpacity>
      
      {results.map((result, index) => (
        <View key={index} style={styles.resultCard}>
          <Text style={styles.resultName}>{result.name}</Text>
          <Text style={styles.resultDetail}>
            {result.iterations.toLocaleString()} iterations in {result.duration}ms
          </Text>
          <Text style={styles.resultOps}>
            {result.opsPerSecond.toLocaleString()} ops/sec
          </Text>
        </View>
      ))}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 22, fontWeight: 'bold', marginBottom: 16 },
  warning: { color: '#FF9500', marginBottom: 12 },
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
    padding: 12,
    borderRadius: 8,
    marginBottom: 8,
    borderWidth: 1,
    borderColor: '#e0e0e0'
  },
  resultName: { fontSize: 16, fontWeight: '600', marginBottom: 4 },
  resultDetail: { fontSize: 14, color: '#666', marginBottom: 2 },
  resultOps: { fontSize: 14, color: '#34C759', fontWeight: '600' }
});

export default JSIPerformanceDemo;
```

---

## 5. Custom JSI Module สำหรับ Real-world Use

### 5.1 Image Processing Module

```cpp
// ImageProcessingJSI.h
#pragma once
#include <jsi/jsi.h>
#include <memory>

class ImageProcessingJSI : public facebook::jsi::HostObject {
public:
  facebook::jsi::Value get(
    facebook::jsi::Runtime& runtime,
    const facebook::jsi::PropNameID& name
  ) override;
  
private:
  // Grayscale conversion
  facebook::jsi::Value toGrayscale(
    facebook::jsi::Runtime& runtime,
    const facebook::jsi::Value* args,
    size_t count
  );
  
  // Resize image data
  facebook::jsi::Value resize(
    facebook::jsi::Runtime& runtime,
    const facebook::jsi::Value* args,
    size_t count
  );
  
  // Apply blur
  facebook::jsi::Value gaussianBlur(
    facebook::jsi::Runtime& runtime,
    const facebook::jsi::Value* args,
    size_t count
  );
};
```

### 5.2 TypeScript Interface

```typescript
// ImageProcessing.ts
declare global {
  var __imageProcessing: {
    toGrayscale: (pixels: Uint8Array) => Uint8Array;
    resize: (pixels: Uint8Array, width: number, height: number, newWidth: number, newHeight: number) => Uint8Array;
    gaussianBlur: (pixels: Uint8Array, width: number, height: number, radius: number) => Uint8Array;
  };
}

export class ImageProcessor {
  static isAvailable() {
    return typeof global.__imageProcessing !== 'undefined';
  }
  
  static toGrayscale(imageData: ImageData): ImageData {
    if (!this.isAvailable()) {
      // Fallback JavaScript implementation
      return this.toGrayscaleJS(imageData);
    }
    
    const result = global.__imageProcessing.toGrayscale(imageData.data);
    return new ImageData(result, imageData.width, imageData.height);
  }
  
  private static toGrayscaleJS(imageData: ImageData): ImageData {
    const data = new Uint8ClampedArray(imageData.data);
    for (let i = 0; i < data.length; i += 4) {
      const gray = 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
      data[i] = data[i + 1] = data[i + 2] = gray;
    }
    return new ImageData(data, imageData.width, imageData.height);
  }
}
```

---

## Tips และ Best Practices

### 1. Error Handling ใน C++
```cpp
Value safeMethod(Runtime& runtime, const Value* args, size_t count) {
  try {
    // Your code here
    return Value(42.0);
  } catch (const std::exception& e) {
    throw JSError(runtime, e.what());
  }
}
```

### 2. Memory Management
```cpp
// ใช้ shared_ptr สำหรับ ownership
auto jsiModule = std::make_shared<JSIExample>(jsCallInvoker);
runtime.global().setProperty(
  runtime,
  "__myModule",
  Object::createFromHostObject(runtime, jsiModule)
);
```

### 3. Thread Safety
```cpp
// JSI calls ต้องทำบน JS thread เสมอ
jsCallInvoker_->invokeAsync([&]() {
  // Safe to call JS here
});
```

### 4. Feature Detection
```typescript
// ตรวจสอบก่อนใช้
function isJSIAvailable(): boolean {
  return typeof global.__jsiExample !== 'undefined';
}

function withFallback<T>(
  jsiImpl: () => T,
  bridgeImpl: () => Promise<T>
): T | Promise<T> {
  if (isJSIAvailable()) {
    return jsiImpl();
  }
  return bridgeImpl();
}
```

---

## สรุป

JSI เป็น core technology ของ React Native New Architecture ที่ช่วย:
1. **ลด overhead** - ไม่มี JSON serialization
2. **Synchronous calls** - เรียก native code แบบ synchronous ได้
3. **Shared memory** - JS และ Native ใช้ memory ร่วมกัน
4. **Better performance** - เร็วกว่า Legacy Bridge มาก
5. **Foundation** สำหรับ Turbo Modules และ Fabric
