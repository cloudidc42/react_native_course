# Part 071: Native Modules - iOS

## Native Modules คืออะไร?

Native Modules คือ bridge ระหว่าง JavaScript และ native code (Objective-C/Swift สำหรับ iOS) ที่ช่วยให้เราเข้าถึง API ของระบบปฏิบัติการที่ React Native ไม่ได้รองรับโดยตรง เช่น การเข้าถึง hardware พิเศษ, library ของ third-party, หรือ performance-critical code

## เมื่อไหร่ควรใช้ Native Modules?

- เมื่อต้องการเข้าถึง iOS API ที่ไม่มีใน React Native
- เมื่อต้องการใช้ library native ที่มีอยู่แล้ว
- เมื่อต้องการ performance สูงสำหรับงานประมวลผลหนัก
- เมื่อต้องการรับ events จาก native code

---

## 1. การสร้าง Native Module ด้วย Objective-C

### 1.1 สร้างไฟล์ Header (.h)

```objectivec
// RCTCalendarModule.h
#import <React/RCTBridgeModule.h>

@interface RCTCalendarModule : NSObject <RCTBridgeModule>

@end
```

### 1.2 สร้างไฟล์ Implementation (.m)

```objectivec
// RCTCalendarModule.m
#import "RCTCalendarModule.h"
#import <React/RCTLog.h>
#import <EventKit/EventKit.h>

@implementation RCTCalendarModule

// บอก React Native ว่า module นี้ชื่ออะไร
RCT_EXPORT_MODULE(CalendarModule);

// Export method ที่ JavaScript สามารถเรียกได้
RCT_EXPORT_METHOD(createCalendarEvent:(NSString *)title
                  location:(NSString *)location
                  startDate:(NSDate *)startDate
                  resolve:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)
{
  EKEventStore *eventStore = [[EKEventStore alloc] init];
  
  [eventStore requestAccessToEntityType:EKEntityTypeEvent
                      completion:^(BOOL granted, NSError *error) {
    if (!granted || error) {
      reject(@"PERMISSION_DENIED", @"Calendar access denied", error);
      return;
    }
    
    EKEvent *event = [EKEvent eventWithEventStore:eventStore];
    event.title = title;
    event.location = location;
    event.startDate = startDate;
    event.endDate = [startDate dateByAddingTimeInterval:3600]; // 1 hour
    event.calendar = [eventStore defaultCalendarForNewEvents];
    
    NSError *saveError = nil;
    BOOL success = [eventStore saveEvent:event 
                                    span:EKSpanThisEvent 
                                    commit:YES 
                                    error:&saveError];
    
    if (success) {
      resolve(event.eventIdentifier);
    } else {
      reject(@"SAVE_FAILED", @"Failed to save event", saveError);
    }
  }];
}

// Synchronous method (ใช้ได้แต่ควรระวัง)
RCT_EXPORT_BLOCKING_SYNCHRONOUS_METHOD(getDeviceModel)
{
  return [[UIDevice currentDevice] model];
}

@end
```

---

## 2. การสร้าง Native Module ด้วย Swift

### 2.1 สร้าง Swift Class

```swift
// CalendarModule.swift
import Foundation
import EventKit

@objc(CalendarModule)
class CalendarModule: NSObject {
  
  private let eventStore = EKEventStore()
  
  @objc
  static func requiresMainQueueSetup() -> Bool {
    return false
  }
  
  @objc(createCalendarEvent:location:startDate:resolve:reject:)
  func createCalendarEvent(
    title: String,
    location: String,
    startDate: Date,
    resolve: @escaping RCTPromiseResolveBlock,
    reject: @escaping RCTPromiseRejectBlock
  ) {
    eventStore.requestAccess(to: .event) { [weak self] granted, error in
      guard let self = self else { return }
      
      if let error = error {
        reject("PERMISSION_ERROR", "Cannot access calendar", error)
        return
      }
      
      guard granted else {
        reject("PERMISSION_DENIED", "Calendar permission denied", nil)
        return
      }
      
      let event = EKEvent(eventStore: self.eventStore)
      event.title = title
      event.location = location
      event.startDate = startDate
      event.endDate = startDate.addingTimeInterval(3600)
      event.calendar = self.eventStore.defaultCalendarForNewEvents
      
      do {
        try self.eventStore.save(event, span: .thisEvent, commit: true)
        resolve(event.eventIdentifier)
      } catch {
        reject("SAVE_FAILED", "Failed to save event", error)
      }
    }
  }
  
  @objc(getAllEvents:reject:)
  func getAllEvents(
    resolve: @escaping RCTPromiseResolveBlock,
    reject: @escaping RCTPromiseRejectBlock
  ) {
    eventStore.requestAccess(to: .event) { [weak self] granted, error in
      guard let self = self else { return }
      
      if !granted {
        reject("PERMISSION_DENIED", "Calendar permission denied", error)
        return
      }
      
      let calendars = self.eventStore.calendars(for: .event)
      let now = Date()
      let oneMonthLater = Calendar.current.date(byAdding: .month, value: 1, to: now)!
      
      let predicate = self.eventStore.predicateForEvents(
        withStart: now,
        end: oneMonthLater,
        calendars: calendars
      )
      
      let events = self.eventStore.events(matching: predicate)
      let eventDicts = events.map { event -> [String: Any] in
        return [
          "id": event.eventIdentifier ?? "",
          "title": event.title ?? "",
          "startDate": event.startDate.timeIntervalSince1970,
          "endDate": event.endDate?.timeIntervalSince1970 ?? 0,
          "location": event.location ?? ""
        ]
      }
      
      resolve(eventDicts)
    }
  }
}
```

### 2.2 สร้าง Bridging Header สำหรับ Swift

```objectivec
// CalendarModule-Bridging-Header.h
// เพิ่มเมื่อใช้ Swift
#import <React/RCTBridgeModule.h>
#import <React/RCTEventEmitter.h>
```

### 2.3 สร้าง Objective-C Bridge สำหรับ Swift Module

```objectivec
// CalendarModuleBridge.m
#import <React/RCTBridgeModule.h>

@interface RCT_EXTERN_MODULE(CalendarModule, NSObject)

RCT_EXTERN_METHOD(createCalendarEvent:(NSString *)title
                  location:(NSString *)location
                  startDate:(NSDate *)startDate
                  resolve:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)

RCT_EXTERN_METHOD(getAllEvents:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)

@end
```

---

## 3. การส่ง Events จาก Native ไปยัง JavaScript

### 3.1 สร้าง Event Emitter ใน Objective-C

```objectivec
// RCTEventEmitterModule.h
#import <React/RCTBridgeModule.h>
#import <React/RCTEventEmitter.h>

@interface RCTEventEmitterModule : RCTEventEmitter <RCTBridgeModule>

@end
```

```objectivec
// RCTEventEmitterModule.m
#import "RCTEventEmitterModule.h"

@implementation RCTEventEmitterModule
{
  bool hasListeners;
}

RCT_EXPORT_MODULE(EventEmitterModule);

// กำหนด events ที่จะ emit
- (NSArray<NSString *> *)supportedEvents
{
  return @[@"onBatteryLevelChanged", 
           @"onNetworkStatusChanged",
           @"onLocationUpdate"];
}

// เรียกเมื่อมี listener
- (void)startObserving
{
  hasListeners = YES;
  // เริ่ม subscribe events ของ native
  [[UIDevice currentDevice] setBatteryMonitoringEnabled:YES];
  [[NSNotificationCenter defaultCenter] addObserver:self
    selector:@selector(batteryLevelDidChange:)
    name:UIDeviceBatteryLevelDidChangeNotification
    object:nil];
}

// เรียกเมื่อไม่มี listener แล้ว
- (void)stopObserving
{
  hasListeners = NO;
  [[UIDevice currentDevice] setBatteryMonitoringEnabled:NO];
  [[NSNotificationCenter defaultCenter] removeObserver:self];
}

- (void)batteryLevelDidChange:(NSNotification *)notification
{
  if (hasListeners) {
    float batteryLevel = [[UIDevice currentDevice] batteryLevel];
    [self sendEventWithName:@"onBatteryLevelChanged" 
                      body:@{@"level": @(batteryLevel * 100)}];
  }
}

RCT_EXPORT_METHOD(startMonitoring)
{
  // Method ที่ JavaScript เรียกเพื่อเริ่ม monitoring
  RCTLogInfo(@"Starting native monitoring");
}

@end
```

### 3.2 สร้าง Event Emitter ใน Swift

```swift
// NetworkMonitorModule.swift
import Foundation
import Network

@objc(NetworkMonitorModule)
class NetworkMonitorModule: RCTEventEmitter {
  
  private var monitor: NWPathMonitor?
  private var hasListeners = false
  
  override static func requiresMainQueueSetup() -> Bool {
    return false
  }
  
  override func supportedEvents() -> [String]! {
    return ["onNetworkChange", "onConnectionTypeChange"]
  }
  
  override func startObserving() {
    hasListeners = true
    startNetworkMonitoring()
  }
  
  override func stopObserving() {
    hasListeners = false
    monitor?.cancel()
    monitor = nil
  }
  
  private func startNetworkMonitoring() {
    monitor = NWPathMonitor()
    monitor?.pathUpdateHandler = { [weak self] path in
      guard let self = self, self.hasListeners else { return }
      
      let isConnected = path.status == .satisfied
      let connectionType = self.getConnectionType(path)
      
      self.sendEvent(
        withName: "onNetworkChange",
        body: [
          "isConnected": isConnected,
          "type": connectionType
        ]
      )
    }
    monitor?.start(queue: DispatchQueue.global())
  }
  
  private func getConnectionType(_ path: NWPath) -> String {
    if path.usesInterfaceType(.wifi) { return "wifi" }
    if path.usesInterfaceType(.cellular) { return "cellular" }
    if path.usesInterfaceType(.wiredEthernet) { return "ethernet" }
    return "unknown"
  }
  
  @objc(startMonitoring:reject:)
  func startMonitoring(
    resolve: RCTPromiseResolveBlock,
    reject: RCTPromiseRejectBlock
  ) {
    startNetworkMonitoring()
    resolve(["status": "monitoring started"])
  }
}
```

```objectivec
// NetworkMonitorModuleBridge.m
#import <React/RCTBridgeModule.h>
#import <React/RCTEventEmitter.h>

@interface RCT_EXTERN_MODULE(NetworkMonitorModule, RCTEventEmitter)

RCT_EXTERN_METHOD(startMonitoring:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)

@end
```

---

## 4. การใช้งาน Native Module ใน JavaScript

### 4.1 Basic Usage

```typescript
// CalendarService.ts
import { NativeModules, Platform } from 'react-native';

const { CalendarModule } = NativeModules;

interface CalendarEvent {
  title: string;
  location: string;
  startDate: Date;
}

interface NativeCalendarModule {
  createCalendarEvent(
    title: string,
    location: string,
    startDate: number,
    resolve: (id: string) => void,
    reject: (code: string, message: string, error: any) => void
  ): void;
  getAllEvents(): Promise<CalendarEvent[]>;
}

export class CalendarService {
  private static module: NativeCalendarModule = CalendarModule;
  
  static async createEvent(event: CalendarEvent): Promise<string> {
    if (Platform.OS !== 'ios') {
      throw new Error('CalendarModule is iOS only');
    }
    
    return new Promise((resolve, reject) => {
      CalendarModule.createCalendarEvent(
        event.title,
        event.location,
        event.startDate.getTime() / 1000,
        resolve,
        reject
      );
    });
  }
  
  static async getEvents(): Promise<CalendarEvent[]> {
    if (Platform.OS !== 'ios') {
      return [];
    }
    return CalendarModule.getAllEvents();
  }
}
```

### 4.2 การใช้งาน Event Emitter

```typescript
// NetworkMonitor.ts
import { NativeModules, NativeEventEmitter, Platform } from 'react-native';
import { useEffect, useState } from 'react';

const { NetworkMonitorModule } = NativeModules;

interface NetworkStatus {
  isConnected: boolean;
  type: 'wifi' | 'cellular' | 'ethernet' | 'unknown';
}

const networkEmitter = Platform.OS === 'ios' 
  ? new NativeEventEmitter(NetworkMonitorModule)
  : null;

export function useNetworkMonitor() {
  const [networkStatus, setNetworkStatus] = useState<NetworkStatus>({
    isConnected: true,
    type: 'unknown'
  });
  
  useEffect(() => {
    if (Platform.OS !== 'ios' || !networkEmitter) return;
    
    // Start monitoring
    NetworkMonitorModule.startMonitoring()
      .then((result: any) => console.log('Monitoring started:', result))
      .catch((error: any) => console.error('Monitor error:', error));
    
    // Listen for events
    const subscription = networkEmitter.addListener(
      'onNetworkChange',
      (event: NetworkStatus) => {
        setNetworkStatus(event);
      }
    );
    
    return () => {
      subscription.remove();
    };
  }, []);
  
  return networkStatus;
}
```

### 4.3 สร้าง Wrapper ที่ดีกว่า

```typescript
// useNetworkStatus.tsx
import React, { createContext, useContext, useEffect, useState } from 'react';
import { NativeModules, NativeEventEmitter, Platform } from 'react-native';

interface NetworkContextType {
  isConnected: boolean;
  connectionType: string;
  startMonitoring: () => void;
  stopMonitoring: () => void;
}

const NetworkContext = createContext<NetworkContextType>({
  isConnected: true,
  connectionType: 'unknown',
  startMonitoring: () => {},
  stopMonitoring: () => {}
});

export const NetworkProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [isConnected, setIsConnected] = useState(true);
  const [connectionType, setConnectionType] = useState('unknown');
  
  const startMonitoring = () => {
    if (Platform.OS !== 'ios') return;
    
    const { NetworkMonitorModule } = NativeModules;
    const emitter = new NativeEventEmitter(NetworkMonitorModule);
    
    NetworkMonitorModule.startMonitoring()
      .catch(console.error);
    
    const sub = emitter.addListener('onNetworkChange', (event: any) => {
      setIsConnected(event.isConnected);
      setConnectionType(event.type);
    });
    
    return () => sub.remove();
  };
  
  useEffect(() => {
    const cleanup = startMonitoring();
    return cleanup;
  }, []);
  
  return (
    <NetworkContext.Provider value={{
      isConnected,
      connectionType,
      startMonitoring,
      stopMonitoring: () => {}
    }}>
      {children}
    </NetworkContext.Provider>
  );
};

export const useNetwork = () => useContext(NetworkContext);
```

---

## 5. Advanced: Custom Type Conversions

### 5.1 การส่ง Complex Data Types

```objectivec
// DataConverterModule.m
#import "DataConverterModule.h"

@implementation DataConverterModule

RCT_EXPORT_MODULE();

// ส่ง Dictionary กลับ
RCT_EXPORT_METHOD(getUserInfo:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)
{
  NSDictionary *userInfo = @{
    @"id": @"user123",
    @"name": @"สมชาย ใจดี",
    @"age": @(25),
    @"isPremium": @YES,
    @"tags": @[@"developer", @"ios"],
    @"metadata": @{
      @"createdAt": @([[NSDate date] timeIntervalSince1970]),
      @"deviceModel": [[UIDevice currentDevice] model]
    }
  };
  
  resolve(userInfo);
}

// รับ Complex Object จาก JavaScript
RCT_EXPORT_METHOD(processData:(NSDictionary *)data
                  resolve:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)
{
  NSString *name = data[@"name"];
  NSNumber *value = data[@"value"];
  NSArray *items = data[@"items"];
  
  if (!name || !value) {
    reject(@"INVALID_DATA", @"name and value are required", nil);
    return;
  }
  
  // Process data...
  NSDictionary *result = @{
    @"processed": @YES,
    @"name": name,
    @"total": @([value doubleValue] * [items count])
  };
  
  resolve(result);
}

// ส่ง Callback แทน Promise
RCT_EXPORT_METHOD(fetchDataWithCallback:(RCTResponseSenderBlock)callback)
{
  // Simulate async operation
  dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC),
    dispatch_get_main_queue(), ^{
    callback(@[[NSNull null], @{@"data": @"fetched successfully"}]);
    // callback(@[@"Error message", [NSNull null]]); // For error
  });
}

@end
```

---

## 6. Threading และ Performance

```objectivec
// ThreadingModule.m
#import "ThreadingModule.h"

@implementation ThreadingModule

RCT_EXPORT_MODULE();

// ระบุว่าให้ run บน main queue
- (dispatch_queue_t)methodQueue
{
  return dispatch_get_main_queue();
}

// หรือสร้าง custom queue
- (dispatch_queue_t)methodQueue
{
  return dispatch_queue_create("com.myapp.CalendarModule", DISPATCH_QUEUE_SERIAL);
}

RCT_EXPORT_METHOD(heavyComputation:(NSArray *)data
                  resolve:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)
{
  // Background thread computation
  dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSMutableArray *results = [NSMutableArray array];
    
    for (NSNumber *item in data) {
      // Simulate heavy computation
      double result = sqrt([item doubleValue]) * M_PI;
      [results addObject:@(result)];
    }
    
    // Return on main thread
    dispatch_async(dispatch_get_main_queue(), ^{
      resolve(results);
    });
  });
}

@end
```

---

## Workshop: สร้าง Biometric Authentication Module

### Step 1: Native Implementation

```swift
// BiometricAuthModule.swift
import Foundation
import LocalAuthentication

@objc(BiometricAuthModule)
class BiometricAuthModule: NSObject {
  
  @objc static func requiresMainQueueSetup() -> Bool {
    return false
  }
  
  @objc(checkBiometricAvailability:reject:)
  func checkBiometricAvailability(
    resolve: @escaping RCTPromiseResolveBlock,
    reject: @escaping RCTPromiseRejectBlock
  ) {
    let context = LAContext()
    var error: NSError?
    
    let available = context.canEvaluatePolicy(
      .deviceOwnerAuthenticationWithBiometrics,
      error: &error
    )
    
    if available {
      let biometryType: String
      if #available(iOS 11.0, *) {
        switch context.biometryType {
        case .faceID:
          biometryType = "FaceID"
        case .touchID:
          biometryType = "TouchID"
        default:
          biometryType = "None"
        }
      } else {
        biometryType = "TouchID"
      }
      
      resolve([
        "available": true,
        "biometryType": biometryType
      ])
    } else {
      let errorCode: String
      switch error?.code {
      case LAError.biometryNotAvailable.rawValue:
        errorCode = "NOT_AVAILABLE"
      case LAError.biometryNotEnrolled.rawValue:
        errorCode = "NOT_ENROLLED"
      default:
        errorCode = "UNKNOWN"
      }
      
      resolve([
        "available": false,
        "errorCode": errorCode,
        "errorMessage": error?.localizedDescription ?? "Unknown error"
      ])
    }
  }
  
  @objc(authenticate:reason:resolve:reject:)
  func authenticate(
    options: NSDictionary,
    reason: String,
    resolve: @escaping RCTPromiseResolveBlock,
    reject: @escaping RCTPromiseRejectBlock
  ) {
    let context = LAContext()
    
    // Configure options
    if let fallbackTitle = options["fallbackTitle"] as? String {
      context.localizedFallbackTitle = fallbackTitle
    }
    
    if let cancelTitle = options["cancelTitle"] as? String {
      if #available(iOS 10.0, *) {
        context.localizedCancelTitle = cancelTitle
      }
    }
    
    context.evaluatePolicy(
      .deviceOwnerAuthenticationWithBiometrics,
      localizedReason: reason
    ) { success, error in
      if success {
        resolve(["success": true])
      } else {
        let errorCode: String
        switch (error as? LAError)?.code {
        case .userCancel:
          errorCode = "USER_CANCEL"
        case .userFallback:
          errorCode = "USER_FALLBACK"
        case .biometryLockout:
          errorCode = "BIOMETRY_LOCKOUT"
        default:
          errorCode = "AUTH_FAILED"
        }
        
        resolve([
          "success": false,
          "errorCode": errorCode,
          "errorMessage": error?.localizedDescription ?? ""
        ])
      }
    }
  }
}
```

```objectivec
// BiometricAuthModuleBridge.m
#import <React/RCTBridgeModule.h>

@interface RCT_EXTERN_MODULE(BiometricAuthModule, NSObject)

RCT_EXTERN_METHOD(checkBiometricAvailability:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)

RCT_EXTERN_METHOD(authenticate:(NSDictionary *)options
                  reason:(NSString *)reason
                  resolve:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)

@end
```

### Step 2: JavaScript Interface

```typescript
// BiometricAuth.ts
import { NativeModules, Platform } from 'react-native';

const { BiometricAuthModule } = NativeModules;

export interface BiometricAvailability {
  available: boolean;
  biometryType?: 'FaceID' | 'TouchID' | 'None';
  errorCode?: string;
  errorMessage?: string;
}

export interface AuthResult {
  success: boolean;
  errorCode?: string;
  errorMessage?: string;
}

export interface AuthOptions {
  fallbackTitle?: string;
  cancelTitle?: string;
}

class BiometricAuth {
  async checkAvailability(): Promise<BiometricAvailability> {
    if (Platform.OS !== 'ios') {
      return { available: false, errorCode: 'PLATFORM_NOT_SUPPORTED' };
    }
    return BiometricAuthModule.checkBiometricAvailability();
  }
  
  async authenticate(
    reason: string,
    options: AuthOptions = {}
  ): Promise<AuthResult> {
    if (Platform.OS !== 'ios') {
      return { success: false, errorCode: 'PLATFORM_NOT_SUPPORTED' };
    }
    
    const availability = await this.checkAvailability();
    if (!availability.available) {
      return {
        success: false,
        errorCode: availability.errorCode,
        errorMessage: availability.errorMessage
      };
    }
    
    return BiometricAuthModule.authenticate(options, reason);
  }
}

export default new BiometricAuth();
```

### Step 3: React Component

```tsx
// BiometricLoginScreen.tsx
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Alert,
  ActivityIndicator
} from 'react-native';
import BiometricAuth from './BiometricAuth';

const BiometricLoginScreen: React.FC = () => {
  const [biometryType, setBiometryType] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  
  useEffect(() => {
    checkBiometricAvailability();
  }, []);
  
  const checkBiometricAvailability = async () => {
    const result = await BiometricAuth.checkAvailability();
    if (result.available && result.biometryType) {
      setBiometryType(result.biometryType);
    }
  };
  
  const handleBiometricAuth = async () => {
    setIsLoading(true);
    
    try {
      const result = await BiometricAuth.authenticate(
        'กรุณายืนยันตัวตนเพื่อเข้าสู่ระบบ',
        {
          fallbackTitle: 'ใช้รหัสผ่าน',
          cancelTitle: 'ยกเลิก'
        }
      );
      
      if (result.success) {
        setIsAuthenticated(true);
        Alert.alert('สำเร็จ', 'ยืนยันตัวตนสำเร็จ!');
      } else {
        switch (result.errorCode) {
          case 'USER_CANCEL':
            // User cancelled, do nothing
            break;
          case 'USER_FALLBACK':
            // User wants to use password
            showPasswordLogin();
            break;
          case 'BIOMETRY_LOCKOUT':
            Alert.alert(
              'ล็อคแล้ว',
              'Biometric ถูกล็อคเนื่องจากพยายามหลายครั้งเกินไป'
            );
            break;
          default:
            Alert.alert('ผิดพลาด', 'ไม่สามารถยืนยันตัวตนได้');
        }
      }
    } catch (error) {
      Alert.alert('Error', 'เกิดข้อผิดพลาด');
    } finally {
      setIsLoading(false);
    }
  };
  
  const showPasswordLogin = () => {
    Alert.alert('รหัสผ่าน', 'กรุณาใส่รหัสผ่าน');
  };
  
  if (isAuthenticated) {
    return (
      <View style={styles.container}>
        <Text style={styles.successText}>ยืนยันตัวตนสำเร็จ!</Text>
      </View>
    );
  }
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>เข้าสู่ระบบ</Text>
      
      {biometryType && (
        <TouchableOpacity
          style={styles.biometricButton}
          onPress={handleBiometricAuth}
          disabled={isLoading}
        >
          {isLoading ? (
            <ActivityIndicator color="#fff" />
          ) : (
            <Text style={styles.buttonText}>
              เข้าสู่ระบบด้วย {biometryType}
            </Text>
          )}
        </TouchableOpacity>
      )}
      
      <TouchableOpacity
        style={styles.passwordButton}
        onPress={showPasswordLogin}
      >
        <Text style={styles.passwordButtonText}>ใช้รหัสผ่าน</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f5f5f5',
    padding: 20
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    marginBottom: 40,
    color: '#333'
  },
  biometricButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 40,
    paddingVertical: 15,
    borderRadius: 10,
    marginBottom: 15,
    minWidth: 200,
    alignItems: 'center'
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600'
  },
  passwordButton: {
    padding: 10
  },
  passwordButtonText: {
    color: '#007AFF',
    fontSize: 14
  },
  successText: {
    fontSize: 24,
    color: '#34C759',
    fontWeight: 'bold'
  }
});

export default BiometricLoginScreen;
```

---

## Tips และ Best Practices

### 1. Thread Safety
```objectivec
// ใช้ dispatch_queue เพื่อความปลอดภัย
@implementation MyModule
{
  dispatch_queue_t _queue;
}

- (instancetype)init
{
  if (self = [super init]) {
    _queue = dispatch_queue_create("com.myapp.module", DISPATCH_QUEUE_SERIAL);
  }
  return self;
}

RCT_EXPORT_METHOD(safeMethod:(RCTPromiseResolveBlock)resolve
                  reject:(RCTPromiseRejectBlock)reject)
{
  dispatch_async(_queue, ^{
    // Thread-safe operations here
    resolve(@"done");
  });
}
@end
```

### 2. Error Handling ที่ดี
```objectivec
// กำหนด error codes ที่ชัดเจน
typedef NS_ENUM(NSInteger, CalendarError) {
  CalendarErrorPermissionDenied = 1001,
  CalendarErrorSaveFailed = 1002,
  CalendarErrorNotFound = 1003
};

static NSString * const CalendarErrorDomain = @"com.myapp.CalendarError";

// ใช้ใน method
NSError *error = [NSError errorWithDomain:CalendarErrorDomain
                                    code:CalendarErrorPermissionDenied
                                userInfo:@{
  NSLocalizedDescriptionKey: @"Calendar access was denied",
  NSLocalizedFailureReasonErrorKey: @"User denied calendar permission"
}];
reject(@"PERMISSION_DENIED", @"Calendar access denied", error);
```

### 3. Memory Management
```swift
// ใช้ weak self เพื่อป้องกัน retain cycle
func doAsync(resolve: @escaping RCTPromiseResolveBlock) {
  someAsyncOperation { [weak self] result in
    guard let self = self else { return }
    resolve(result)
  }
}
```

### 4. Testing Native Modules
```typescript
// __mocks__/NativeModules.ts
import { NativeModules } from 'react-native';

NativeModules.CalendarModule = {
  createCalendarEvent: jest.fn().mockResolvedValue('event-123'),
  getAllEvents: jest.fn().mockResolvedValue([]),
};

NativeModules.BiometricAuthModule = {
  checkBiometricAvailability: jest.fn().mockResolvedValue({
    available: true,
    biometryType: 'FaceID'
  }),
  authenticate: jest.fn().mockResolvedValue({ success: true })
};
```

---

## สรุป

Native Modules ใน iOS ช่วยให้เราสามารถ:

1. เข้าถึง iOS API ที่ไม่มีใน React Native
2. ใช้งาน library native ที่มีอยู่แล้ว
3. ทำงานที่ต้องการ performance สูง
4. รับ events จาก native code

ควรคำนึงถึง:
- Thread safety เสมอ
- Error handling ที่ครอบคลุม
- Memory management ที่ถูกต้อง
- การ test ที่เหมาะสม
