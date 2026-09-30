# Part 051: Background Tasks ใน React Native

## ความรู้เบื้องต้นเกี่ยวกับ Background Tasks

Background Tasks คือการทำงานที่แอปพลิเคชันสามารถดำเนินการได้แม้ว่าผู้ใช้จะไม่ได้ใช้งานแอปอยู่ในขณะนั้น หรือแม้แต่เมื่อแอปอยู่ใน background state การทำงานแบบนี้มีความสำคัญมากสำหรับแอปที่ต้องการ sync ข้อมูล, รับ notification, ติดตาม location หรือทำการประมวลผลข้อมูลในเบื้องหลัง

### ประเภทของ Background Tasks

```
1. Background Fetch - ดึงข้อมูลเป็นระยะๆ
2. Background Location - ติดตาม GPS เมื่อแอปอยู่ background
3. Background Sync - Sync ข้อมูลกับ server
4. Headless JS (Android) - รัน JS code โดยไม่มี UI
5. Background Notifications - ประมวลผล push notifications
```

### ข้อจำกัดของ Background Tasks

ทั้ง iOS และ Android มีข้อจำกัดเกี่ยวกับ background tasks เพื่อประหยัด battery:

**iOS:**
- ให้เวลา background execution จำกัด (ประมาณ 30 วินาที)
- Background Fetch ถูกควบคุมโดย OS
- ต้องขอ permission พิเศษสำหรับ background location

**Android:**
- Doze Mode จำกัดการทำงาน background
- Background execution limits ใน Android 8+
- ต้องใช้ WorkManager หรือ JobScheduler

---

## การติดตั้ง react-native-background-fetch

```bash
npm install react-native-background-fetch
# หรือ
yarn add react-native-background-fetch

# สำหรับ iOS
cd ios && pod install
```

### การตั้งค่า iOS

เพิ่มใน `Info.plist`:

```xml
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
    <string>remote-notification</string>
</array>
```

### การตั้งค่า Android

เพิ่มใน `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>

<application ...>
    <service
        android:name="com.transistorsoft.rnbackgroundfetch.BackgroundFetchHeadlessTask"
        android:exported="false" />
</application>
```

---

## การใช้งาน react-native-background-fetch

### ตัวอย่างพื้นฐาน

```typescript
import React, { useEffect, useState } from 'react';
import { View, Text, StyleSheet, FlatList } from 'react-native';
import BackgroundFetch from 'react-native-background-fetch';

interface SyncRecord {
  id: string;
  timestamp: string;
  status: string;
}

const BackgroundFetchDemo: React.FC = () => {
  const [syncHistory, setSyncHistory] = useState<SyncRecord[]>([]);
  const [lastSync, setLastSync] = useState<string>('ยังไม่เคย sync');

  useEffect(() => {
    // กำหนดค่า Background Fetch
    initBackgroundFetch();

    return () => {
      // Cleanup เมื่อ component unmount
      BackgroundFetch.stop();
    };
  }, []);

  const initBackgroundFetch = async () => {
    try {
      const status = await BackgroundFetch.configure(
        {
          // ระยะเวลาขั้นต่ำระหว่าง fetch (นาที)
          minimumFetchInterval: 15,
          
          // iOS เท่านั้น
          stopOnTerminate: false,  // ทำงานต่อแม้แอปถูกปิด
          enableHeadless: true,    // รองรับ Headless JS
          
          // Android เท่านั้น
          startOnBoot: true,       // เริ่มทำงานเมื่อเปิดเครื่อง
          requiredNetworkType: BackgroundFetch.NETWORK_TYPE_ANY,
          requiresCharging: false,
          requiresDeviceIdle: false,
        },
        async (taskId) => {
          // Callback ที่จะถูกเรียกเมื่อ background fetch เกิดขึ้น
          console.log('[BackgroundFetch] Task started:', taskId);
          
          // ทำการ sync ข้อมูล
          await performDataSync();
          
          // บอกให้ระบบรู้ว่า task เสร็จสิ้นแล้ว (สำคัญมาก!)
          BackgroundFetch.finish(taskId);
        },
        async (taskId) => {
          // Timeout callback - เมื่อ task ใช้เวลานานเกินไป
          console.log('[BackgroundFetch] Task timeout:', taskId);
          BackgroundFetch.finish(taskId);
        }
      );
      
      console.log('[BackgroundFetch] Status:', status);
    } catch (error) {
      console.error('[BackgroundFetch] Error:', error);
    }
  };

  const performDataSync = async () => {
    try {
      // จำลองการ sync ข้อมูลกับ server
      const response = await fetch('https://api.example.com/sync', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          deviceId: 'device-123',
          timestamp: new Date().toISOString(),
        }),
      });

      if (response.ok) {
        const timestamp = new Date().toLocaleString('th-TH');
        setLastSync(timestamp);
        
        setSyncHistory(prev => [
          {
            id: Date.now().toString(),
            timestamp,
            status: 'สำเร็จ',
          },
          ...prev.slice(0, 9), // เก็บแค่ 10 รายการล่าสุด
        ]);
      }
    } catch (error) {
      console.error('Sync failed:', error);
      setSyncHistory(prev => [
        {
          id: Date.now().toString(),
          timestamp: new Date().toLocaleString('th-TH'),
          status: 'ล้มเหลว',
        },
        ...prev.slice(0, 9),
      ]);
    }
  };

  const testFetch = async () => {
    // ทดสอบ background fetch ด้วยตนเอง
    await performDataSync();
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Background Fetch Demo</Text>
      <Text style={styles.lastSync}>Sync ล่าสุด: {lastSync}</Text>
      
      <Text style={styles.subtitle}>ประวัติการ Sync:</Text>
      <FlatList
        data={syncHistory}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View style={styles.syncItem}>
            <Text style={styles.syncTime}>{item.timestamp}</Text>
            <Text style={[
              styles.syncStatus,
              { color: item.status === 'สำเร็จ' ? 'green' : 'red' }
            ]}>
              {item.status}
            </Text>
          </View>
        )}
        ListEmptyComponent={
          <Text style={styles.empty}>ยังไม่มีประวัติการ Sync</Text>
        }
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    backgroundColor: '#f5f5f5',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 10,
    color: '#333',
  },
  lastSync: {
    fontSize: 16,
    color: '#666',
    marginBottom: 20,
  },
  subtitle: {
    fontSize: 18,
    fontWeight: '600',
    marginBottom: 10,
    color: '#444',
  },
  syncItem: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    padding: 12,
    backgroundColor: 'white',
    borderRadius: 8,
    marginBottom: 8,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
    elevation: 2,
  },
  syncTime: {
    fontSize: 14,
    color: '#555',
  },
  syncStatus: {
    fontSize: 14,
    fontWeight: '600',
  },
  empty: {
    textAlign: 'center',
    color: '#999',
    fontSize: 16,
    marginTop: 20,
  },
});

export default BackgroundFetchDemo;
```

---

## Background Location

การติดตาม location ใน background เป็นฟีเจอร์ที่ต้องการ permission พิเศษ

### การติดตั้ง

```bash
npm install @react-native-community/geolocation
# หรือใช้ react-native-background-geolocation (แนะนำสำหรับ background)
npm install react-native-background-geolocation
```

### การตั้งค่า Permission

**iOS - Info.plist:**
```xml
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>แอปต้องการเข้าถึง location เพื่อติดตามเส้นทาง</string>
<key>NSLocationAlwaysUsageDescription</key>
<string>แอปต้องการเข้าถึง location ตลอดเวลา</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปต้องการเข้าถึง location</string>
<key>UIBackgroundModes</key>
<array>
    <string>location</string>
</array>
```

**Android - AndroidManifest.xml:**
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
```

### ตัวอย่าง Background Location Service

```typescript
import React, { useEffect, useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Alert,
  PermissionsAndroid,
  Platform,
} from 'react-native';
import Geolocation from '@react-native-community/geolocation';

interface LocationPoint {
  latitude: number;
  longitude: number;
  timestamp: number;
  accuracy: number;
}

const BackgroundLocationTracker: React.FC = () => {
  const [isTracking, setIsTracking] = useState(false);
  const [currentLocation, setCurrentLocation] = useState<LocationPoint | null>(null);
  const [locationHistory, setLocationHistory] = useState<LocationPoint[]>([]);
  const [watchId, setWatchId] = useState<number | null>(null);

  const requestPermissions = async (): Promise<boolean> => {
    if (Platform.OS === 'ios') {
      // iOS จัดการ permission ผ่าน dialog อัตโนมัติ
      return true;
    }

    try {
      // ขอ fine location permission
      const fineLocation = await PermissionsAndroid.request(
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
        {
          title: 'ขอ Location Permission',
          message: 'แอปต้องการเข้าถึง location ของคุณเพื่อติดตามเส้นทาง',
          buttonNeutral: 'ถามทีหลัง',
          buttonNegative: 'ยกเลิก',
          buttonPositive: 'อนุญาต',
        }
      );

      if (fineLocation !== PermissionsAndroid.RESULTS.GRANTED) {
        Alert.alert('ข้อผิดพลาด', 'ต้องการ Location Permission');
        return false;
      }

      // Android 10+ ต้องขอ background location แยก
      if (Platform.Version >= 29) {
        const backgroundLocation = await PermissionsAndroid.request(
          PermissionsAndroid.PERMISSIONS.ACCESS_BACKGROUND_LOCATION,
          {
            title: 'ขอ Background Location Permission',
            message: 'เพื่อติดตามเส้นทางเมื่อแอปอยู่ background กรุณาเลือก "Allow all the time"',
            buttonNeutral: 'ถามทีหลัง',
            buttonNegative: 'ยกเลิก',
            buttonPositive: 'ตั้งค่า',
          }
        );

        if (backgroundLocation !== PermissionsAndroid.RESULTS.GRANTED) {
          Alert.alert('คำเตือน', 'ไม่ได้รับ background location permission การติดตามอาจไม่สมบูรณ์');
        }
      }

      return true;
    } catch (error) {
      console.error('Permission error:', error);
      return false;
    }
  };

  const startTracking = async () => {
    const hasPermission = await requestPermissions();
    if (!hasPermission) return;

    const id = Geolocation.watchPosition(
      (position) => {
        const point: LocationPoint = {
          latitude: position.coords.latitude,
          longitude: position.coords.longitude,
          timestamp: position.timestamp,
          accuracy: position.coords.accuracy,
        };

        setCurrentLocation(point);
        setLocationHistory(prev => [...prev, point]);
      },
      (error) => {
        console.error('Location error:', error);
        Alert.alert('ข้อผิดพลาด', `ไม่สามารถรับ location: ${error.message}`);
      },
      {
        enableHighAccuracy: true,
        distanceFilter: 10,    // อัปเดตทุก 10 เมตร
        interval: 5000,         // อัปเดตทุก 5 วินาที
        fastestInterval: 2000,  // เร็วสุด 2 วินาที
      }
    );

    setWatchId(id);
    setIsTracking(true);
  };

  const stopTracking = () => {
    if (watchId !== null) {
      Geolocation.clearWatch(watchId);
      setWatchId(null);
    }
    setIsTracking(false);
  };

  const calculateDistance = (points: LocationPoint[]): number => {
    if (points.length < 2) return 0;
    
    let totalDistance = 0;
    for (let i = 1; i < points.length; i++) {
      const p1 = points[i - 1];
      const p2 = points[i];
      
      // Haversine formula
      const R = 6371000; // Earth radius in meters
      const φ1 = (p1.latitude * Math.PI) / 180;
      const φ2 = (p2.latitude * Math.PI) / 180;
      const Δφ = ((p2.latitude - p1.latitude) * Math.PI) / 180;
      const Δλ = ((p2.longitude - p1.longitude) * Math.PI) / 180;

      const a =
        Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
        Math.cos(φ1) * Math.cos(φ2) * Math.sin(Δλ / 2) * Math.sin(Δλ / 2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

      totalDistance += R * c;
    }
    
    return totalDistance;
  };

  useEffect(() => {
    return () => {
      if (watchId !== null) {
        Geolocation.clearWatch(watchId);
      }
    };
  }, [watchId]);

  const totalDistance = calculateDistance(locationHistory);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Background Location Tracker</Text>
      
      <View style={styles.statusCard}>
        <Text style={styles.statusLabel}>สถานะ:</Text>
        <Text style={[
          styles.statusValue,
          { color: isTracking ? '#4CAF50' : '#F44336' }
        ]}>
          {isTracking ? 'กำลังติดตาม' : 'หยุดแล้ว'}
        </Text>
      </View>

      {currentLocation && (
        <View style={styles.locationCard}>
          <Text style={styles.cardTitle}>ตำแหน่งปัจจุบัน</Text>
          <Text style={styles.locationText}>
            Lat: {currentLocation.latitude.toFixed(6)}
          </Text>
          <Text style={styles.locationText}>
            Lng: {currentLocation.longitude.toFixed(6)}
          </Text>
          <Text style={styles.locationText}>
            ความแม่นยำ: ±{currentLocation.accuracy.toFixed(0)} ม.
          </Text>
        </View>
      )}

      <View style={styles.statsCard}>
        <Text style={styles.cardTitle}>สถิติ</Text>
        <Text style={styles.statsText}>
          จุดที่บันทึก: {locationHistory.length} จุด
        </Text>
        <Text style={styles.statsText}>
          ระยะทางรวม: {(totalDistance / 1000).toFixed(2)} กม.
        </Text>
      </View>

      <TouchableOpacity
        style={[styles.button, isTracking ? styles.stopButton : styles.startButton]}
        onPress={isTracking ? stopTracking : startTracking}
      >
        <Text style={styles.buttonText}>
          {isTracking ? 'หยุดติดตาม' : 'เริ่มติดตาม'}
        </Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    backgroundColor: '#f0f4f8',
  },
  title: {
    fontSize: 22,
    fontWeight: 'bold',
    textAlign: 'center',
    marginBottom: 20,
    color: '#1a1a2e',
  },
  statusCard: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 15,
    borderRadius: 12,
    marginBottom: 15,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  statusLabel: {
    fontSize: 16,
    color: '#666',
    marginRight: 10,
  },
  statusValue: {
    fontSize: 16,
    fontWeight: 'bold',
  },
  locationCard: {
    backgroundColor: 'white',
    padding: 15,
    borderRadius: 12,
    marginBottom: 15,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  cardTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  locationText: {
    fontSize: 14,
    color: '#555',
    marginBottom: 4,
  },
  statsCard: {
    backgroundColor: 'white',
    padding: 15,
    borderRadius: 12,
    marginBottom: 20,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  statsText: {
    fontSize: 14,
    color: '#555',
    marginBottom: 4,
  },
  button: {
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  startButton: {
    backgroundColor: '#4CAF50',
  },
  stopButton: {
    backgroundColor: '#F44336',
  },
  buttonText: {
    color: 'white',
    fontSize: 18,
    fontWeight: 'bold',
  },
});

export default BackgroundLocationTracker;
```

---

## Background Sync

Background Sync คือการ synchronize ข้อมูลระหว่าง local และ server ในเบื้องหลัง

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import NetInfo from '@react-native-community/netinfo';

interface PendingOperation {
  id: string;
  type: 'CREATE' | 'UPDATE' | 'DELETE';
  endpoint: string;
  data: any;
  createdAt: number;
  retryCount: number;
}

class BackgroundSyncService {
  private static instance: BackgroundSyncService;
  private syncQueue: PendingOperation[] = [];
  private isSyncing = false;
  private readonly QUEUE_STORAGE_KEY = '@sync_queue';
  private readonly MAX_RETRIES = 3;

  static getInstance(): BackgroundSyncService {
    if (!BackgroundSyncService.instance) {
      BackgroundSyncService.instance = new BackgroundSyncService();
    }
    return BackgroundSyncService.instance;
  }

  async initialize(): Promise<void> {
    // โหลด queue จาก storage
    await this.loadQueue();
    
    // ฟังการเปลี่ยนแปลงของ network
    NetInfo.addEventListener(state => {
      if (state.isConnected && state.isInternetReachable) {
        this.processSyncQueue();
      }
    });
  }

  async addToQueue(operation: Omit<PendingOperation, 'id' | 'createdAt' | 'retryCount'>): Promise<void> {
    const newOperation: PendingOperation = {
      ...operation,
      id: `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      createdAt: Date.now(),
      retryCount: 0,
    };

    this.syncQueue.push(newOperation);
    await this.saveQueue();

    // พยายาม sync ทันทีถ้ามี connection
    const netState = await NetInfo.fetch();
    if (netState.isConnected && netState.isInternetReachable) {
      this.processSyncQueue();
    }
  }

  async processSyncQueue(): Promise<void> {
    if (this.isSyncing || this.syncQueue.length === 0) return;

    this.isSyncing = true;
    console.log(`[BackgroundSync] Processing ${this.syncQueue.length} operations`);

    const failedOperations: PendingOperation[] = [];

    for (const operation of [...this.syncQueue]) {
      try {
        await this.executeOperation(operation);
        // ลบ operation ที่สำเร็จออกจาก queue
        this.syncQueue = this.syncQueue.filter(op => op.id !== operation.id);
      } catch (error) {
        console.error(`[BackgroundSync] Operation failed:`, error);
        
        if (operation.retryCount < this.MAX_RETRIES) {
          // เพิ่ม retry count
          operation.retryCount++;
          failedOperations.push(operation);
        } else {
          // ลบ operation ที่ retry เกิน limit
          console.error(`[BackgroundSync] Max retries reached for operation ${operation.id}`);
          this.syncQueue = this.syncQueue.filter(op => op.id !== operation.id);
        }
      }
    }

    await this.saveQueue();
    this.isSyncing = false;
  }

  private async executeOperation(operation: PendingOperation): Promise<void> {
    const methodMap = {
      CREATE: 'POST',
      UPDATE: 'PUT',
      DELETE: 'DELETE',
    };

    const response = await fetch(operation.endpoint, {
      method: methodMap[operation.type],
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${await this.getAuthToken()}`,
      },
      body: operation.type !== 'DELETE' ? JSON.stringify(operation.data) : undefined,
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
  }

  private async getAuthToken(): Promise<string> {
    const token = await AsyncStorage.getItem('@auth_token');
    return token || '';
  }

  private async loadQueue(): Promise<void> {
    try {
      const stored = await AsyncStorage.getItem(this.QUEUE_STORAGE_KEY);
      if (stored) {
        this.syncQueue = JSON.parse(stored);
      }
    } catch (error) {
      console.error('[BackgroundSync] Failed to load queue:', error);
    }
  }

  private async saveQueue(): Promise<void> {
    try {
      await AsyncStorage.setItem(
        this.QUEUE_STORAGE_KEY,
        JSON.stringify(this.syncQueue)
      );
    } catch (error) {
      console.error('[BackgroundSync] Failed to save queue:', error);
    }
  }

  getQueueLength(): number {
    return this.syncQueue.length;
  }

  getPendingOperations(): PendingOperation[] {
    return [...this.syncQueue];
  }
}

export default BackgroundSyncService;
```

---

## Headless JS (Android)

Headless JS ช่วยให้ Android รัน JavaScript code โดยไม่ต้องมี UI เหมาะสำหรับการทำงานใน background

### สร้าง Headless Task

```typescript
// src/headlessTasks/backgroundSyncTask.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

const backgroundSyncTask = async (taskData: any) => {
  console.log('[HeadlessJS] Task started with data:', taskData);
  
  try {
    // ทำการ sync ข้อมูล
    const pendingItems = await AsyncStorage.getItem('@pending_sync');
    
    if (pendingItems) {
      const items = JSON.parse(pendingItems);
      
      for (const item of items) {
        try {
          await fetch(`https://api.example.com/${item.endpoint}`, {
            method: item.method,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(item.data),
          });
        } catch (error) {
          console.error('[HeadlessJS] Failed to sync item:', error);
        }
      }
      
      // ล้าง pending items หลัง sync สำเร็จ
      await AsyncStorage.removeItem('@pending_sync');
    }
    
    console.log('[HeadlessJS] Task completed successfully');
  } catch (error) {
    console.error('[HeadlessJS] Task failed:', error);
  }
};

export default backgroundSyncTask;
```

### ลงทะเบียน Headless Task

```typescript
// index.js
import { AppRegistry } from 'react-native';
import App from './App';
import backgroundSyncTask from './src/headlessTasks/backgroundSyncTask';
import BackgroundFetch from 'react-native-background-fetch';

// ลงทะเบียน main app
AppRegistry.registerComponent('MyApp', () => App);

// ลงทะเบียน Headless JS task สำหรับ background fetch
BackgroundFetch.registerHeadlessTask(backgroundSyncTask);

// หรือลงทะเบียนผ่าน AppRegistry โดยตรง
AppRegistry.registerHeadlessTask('BackgroundSync', () => backgroundSyncTask);
```

### การเรียกใช้จาก Android Native

```java
// android/app/src/main/java/com/yourapp/BackgroundSyncService.java
package com.yourapp;

import android.content.Intent;
import com.facebook.react.HeadlessJsTaskService;
import com.facebook.react.bridge.Arguments;
import com.facebook.react.jstasks.HeadlessJsTaskConfig;
import javax.annotation.Nullable;

public class BackgroundSyncService extends HeadlessJsTaskService {
    @Nullable
    @Override
    protected HeadlessJsTaskConfig getTaskConfig(Intent intent) {
        Bundle extras = intent.getExtras();
        if (extras != null) {
            return new HeadlessJsTaskConfig(
                "BackgroundSync",  // ชื่อ task ที่ลงทะเบียนใน JS
                Arguments.fromBundle(extras),
                5000,   // timeout ms
                false   // optional: อนุญาตให้รันเมื่อแอปอยู่ foreground
            );
        }
        return null;
    }
}
```

---

## Workshop: Background Data Sync App

สร้างแอปที่ sync ข้อมูล note ในเบื้องหลัง

```typescript
import React, { useEffect, useState, useCallback } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  FlatList,
  StyleSheet,
  Alert,
  ActivityIndicator,
} from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';
import NetInfo from '@react-native-community/netinfo';
import BackgroundFetch from 'react-native-background-fetch';

interface Note {
  id: string;
  content: string;
  createdAt: string;
  isSynced: boolean;
  localOnly?: boolean;
}

const BackgroundSyncNotesApp: React.FC = () => {
  const [notes, setNotes] = useState<Note[]>([]);
  const [newNote, setNewNote] = useState('');
  const [isOnline, setIsOnline] = useState(true);
  const [isSyncing, setIsSyncing] = useState(false);
  const [pendingCount, setPendingCount] = useState(0);

  useEffect(() => {
    loadNotes();
    setupNetworkListener();
    setupBackgroundFetch();

    return () => {
      BackgroundFetch.stop();
    };
  }, []);

  const setupNetworkListener = () => {
    NetInfo.addEventListener(state => {
      const connected = !!(state.isConnected && state.isInternetReachable);
      setIsOnline(connected);
      
      if (connected) {
        syncPendingNotes();
      }
    });
  };

  const setupBackgroundFetch = async () => {
    await BackgroundFetch.configure(
      {
        minimumFetchInterval: 15,
        stopOnTerminate: false,
        startOnBoot: true,
        enableHeadless: true,
      },
      async (taskId) => {
        console.log('[BackgroundFetch] Running background sync');
        await syncPendingNotes();
        BackgroundFetch.finish(taskId);
      },
      (taskId) => {
        console.log('[BackgroundFetch] Timeout:', taskId);
        BackgroundFetch.finish(taskId);
      }
    );
  };

  const loadNotes = async () => {
    try {
      const stored = await AsyncStorage.getItem('@notes');
      if (stored) {
        const loadedNotes: Note[] = JSON.parse(stored);
        setNotes(loadedNotes);
        setPendingCount(loadedNotes.filter(n => !n.isSynced).length);
      }
    } catch (error) {
      console.error('Failed to load notes:', error);
    }
  };

  const saveNotes = async (updatedNotes: Note[]) => {
    try {
      await AsyncStorage.setItem('@notes', JSON.stringify(updatedNotes));
    } catch (error) {
      console.error('Failed to save notes:', error);
    }
  };

  const addNote = async () => {
    if (!newNote.trim()) return;

    const note: Note = {
      id: Date.now().toString(),
      content: newNote.trim(),
      createdAt: new Date().toISOString(),
      isSynced: false,
      localOnly: !isOnline,
    };

    const updatedNotes = [note, ...notes];
    setNotes(updatedNotes);
    setPendingCount(updatedNotes.filter(n => !n.isSynced).length);
    await saveNotes(updatedNotes);
    setNewNote('');

    if (isOnline) {
      await syncNote(note);
    }
  };

  const syncNote = async (note: Note): Promise<boolean> => {
    try {
      // จำลองการ sync กับ server
      await new Promise(resolve => setTimeout(resolve, 1000));
      
      // จำลอง API call
      // const response = await fetch('https://api.example.com/notes', {
      //   method: 'POST',
      //   headers: { 'Content-Type': 'application/json' },
      //   body: JSON.stringify(note),
      // });
      
      return true; // สมมติว่าสำเร็จ
    } catch (error) {
      return false;
    }
  };

  const syncPendingNotes = useCallback(async () => {
    const pending = notes.filter(n => !n.isSynced);
    if (pending.length === 0 || isSyncing) return;

    setIsSyncing(true);
    
    try {
      const updatedNotes = [...notes];
      
      for (const note of pending) {
        const success = await syncNote(note);
        if (success) {
          const index = updatedNotes.findIndex(n => n.id === note.id);
          if (index !== -1) {
            updatedNotes[index] = { ...updatedNotes[index], isSynced: true };
          }
        }
      }

      setNotes(updatedNotes);
      setPendingCount(updatedNotes.filter(n => !n.isSynced).length);
      await saveNotes(updatedNotes);
    } catch (error) {
      console.error('Sync failed:', error);
    } finally {
      setIsSyncing(false);
    }
  }, [notes, isSyncing]);

  const deleteNote = async (id: string) => {
    const updatedNotes = notes.filter(n => n.id !== id);
    setNotes(updatedNotes);
    setPendingCount(updatedNotes.filter(n => !n.isSynced).length);
    await saveNotes(updatedNotes);
  };

  return (
    <View style={workshopStyles.container}>
      {/* Header */}
      <View style={workshopStyles.header}>
        <Text style={workshopStyles.title}>Notes (Background Sync)</Text>
        <View style={workshopStyles.statusRow}>
          <View style={[
            workshopStyles.indicator,
            { backgroundColor: isOnline ? '#4CAF50' : '#F44336' }
          ]} />
          <Text style={workshopStyles.statusText}>
            {isOnline ? 'ออนไลน์' : 'ออฟไลน์'}
          </Text>
          {pendingCount > 0 && (
            <Text style={workshopStyles.pendingText}>
              ({pendingCount} รอ sync)
            </Text>
          )}
          {isSyncing && <ActivityIndicator size="small" color="#2196F3" />}
        </View>
      </View>

      {/* Input */}
      <View style={workshopStyles.inputContainer}>
        <TextInput
          style={workshopStyles.input}
          value={newNote}
          onChangeText={setNewNote}
          placeholder="เขียน note ใหม่..."
          multiline
          maxLength={500}
        />
        <TouchableOpacity
          style={workshopStyles.addButton}
          onPress={addNote}
          disabled={!newNote.trim()}
        >
          <Text style={workshopStyles.addButtonText}>เพิ่ม</Text>
        </TouchableOpacity>
      </View>

      {/* Sync Button */}
      {pendingCount > 0 && isOnline && (
        <TouchableOpacity
          style={workshopStyles.syncButton}
          onPress={syncPendingNotes}
          disabled={isSyncing}
        >
          <Text style={workshopStyles.syncButtonText}>
            Sync ตอนนี้ ({pendingCount} รายการ)
          </Text>
        </TouchableOpacity>
      )}

      {/* Notes List */}
      <FlatList
        data={notes}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View style={[
            workshopStyles.noteItem,
            !item.isSynced && workshopStyles.pendingNote
          ]}>
            <View style={workshopStyles.noteContent}>
              <Text style={workshopStyles.noteText}>{item.content}</Text>
              <Text style={workshopStyles.noteDate}>
                {new Date(item.createdAt).toLocaleString('th-TH')}
              </Text>
              {!item.isSynced && (
                <Text style={workshopStyles.pendingLabel}>⏳ รอ sync</Text>
              )}
            </View>
            <TouchableOpacity
              style={workshopStyles.deleteButton}
              onPress={() => deleteNote(item.id)}
            >
              <Text style={workshopStyles.deleteText}>✕</Text>
            </TouchableOpacity>
          </View>
        )}
        ListEmptyComponent={
          <Text style={workshopStyles.empty}>ยังไม่มี notes</Text>
        }
      />
    </View>
  );
};

const workshopStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  header: {
    backgroundColor: '#2196F3',
    padding: 20,
    paddingTop: 40,
  },
  title: {
    fontSize: 22,
    fontWeight: 'bold',
    color: 'white',
    marginBottom: 8,
  },
  statusRow: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
  },
  indicator: {
    width: 10,
    height: 10,
    borderRadius: 5,
  },
  statusText: { color: 'white', fontSize: 14 },
  pendingText: { color: '#FFE082', fontSize: 14 },
  inputContainer: {
    flexDirection: 'row',
    padding: 15,
    backgroundColor: 'white',
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  input: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 10,
    maxHeight: 80,
    marginRight: 10,
  },
  addButton: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 16,
    paddingVertical: 10,
    borderRadius: 8,
    justifyContent: 'center',
  },
  addButtonText: { color: 'white', fontWeight: 'bold' },
  syncButton: {
    backgroundColor: '#FF9800',
    margin: 15,
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  syncButtonText: { color: 'white', fontWeight: 'bold' },
  noteItem: {
    flexDirection: 'row',
    backgroundColor: 'white',
    margin: 10,
    marginBottom: 0,
    borderRadius: 10,
    padding: 15,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
  },
  pendingNote: {
    borderLeftWidth: 3,
    borderLeftColor: '#FF9800',
  },
  noteContent: { flex: 1 },
  noteText: { fontSize: 15, color: '#333', marginBottom: 6 },
  noteDate: { fontSize: 12, color: '#999' },
  pendingLabel: { fontSize: 12, color: '#FF9800', marginTop: 4 },
  deleteButton: {
    padding: 5,
  },
  deleteText: { color: '#F44336', fontSize: 18, fontWeight: 'bold' },
  empty: {
    textAlign: 'center',
    color: '#999',
    fontSize: 16,
    marginTop: 40,
  },
});

export default BackgroundSyncNotesApp;
```

---

## Tips และ Best Practices

### 1. ประหยัด Battery
```typescript
// ใช้ minimumFetchInterval ที่เหมาะสม
BackgroundFetch.configure({
  minimumFetchInterval: 30, // ไม่ควรน้อยกว่า 15 นาที
  requiredNetworkType: BackgroundFetch.NETWORK_TYPE_ANY,
});
```

### 2. จัดการ Errors อย่างเหมาะสม
```typescript
try {
  await performBackgroundTask();
} catch (error) {
  // Log error แต่ไม่ crash
  console.error('Background task error:', error);
} finally {
  // สำคัญมาก! เรียก finish เสมอ
  BackgroundFetch.finish(taskId);
}
```

### 3. ทดสอบ Background Fetch
```typescript
// ทดสอบด้วย simulateFetch
await BackgroundFetch.scheduleTask({
  taskId: 'com.example.test',
  delay: 5000, // 5 วินาที
  periodic: false,
});
```

### 4. Debugging
```typescript
// เปิด debug mode
BackgroundFetch.configure({
  debug: __DEV__, // เปิดเฉพาะ development
});
```

---

## สรุป

Background Tasks ใน React Native เป็นสิ่งสำคัญสำหรับแอปที่ต้องการ:
- Sync ข้อมูลเป็นระยะๆ
- ติดตาม GPS ในเบื้องหลัง
- ประมวลผล notifications
- อัปเดตข้อมูลก่อนผู้ใช้เปิดแอป

ข้อควรระวัง:
- ปฏิบัติตาม guidelines ของ iOS และ Android เสมอ
- อย่าทำงานหนักเกินไปใน background
- เรียก `BackgroundFetch.finish()` เสมอ
- ทดสอบบน device จริง เพราะ simulator อาจให้ผลต่าง
