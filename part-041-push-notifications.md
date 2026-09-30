# Part 041: Push Notifications ใน React Native

## สารบัญ
1. [แนะนำ Push Notifications](#introduction)
2. [FCM (Firebase Cloud Messaging)](#fcm)
3. [APNs (Apple Push Notification Service)](#apns)
4. [Local Notifications](#local-notifications)
5. [Notification Handling](#notification-handling)
6. [Notification Categories](#notification-categories)
7. [Workshop: Notification System](#workshop)

---

## 1. แนะนำ Push Notifications {#introduction}

Push Notifications เป็นหนึ่งในฟีเจอร์ที่สำคัญที่สุดของ Mobile App ที่ช่วยให้เราสามารถส่งข้อความหาผู้ใช้แม้ในขณะที่แอปไม่ได้เปิดอยู่

### ประเภทของ Notifications

```
Push Notifications
├── Remote Notifications (ส่งจาก Server)
│   ├── FCM (Android)
│   └── APNs (iOS)
└── Local Notifications (สร้างใน App)
    ├── Scheduled Notifications
    └── Immediate Notifications
```

### สถานะของ App เมื่อรับ Notification

| สถานะ | คำอธิบาย |
|-------|---------|
| Foreground | App กำลังเปิดอยู่และผู้ใช้เห็น |
| Background | App ทำงานอยู่เบื้องหลัง |
| Killed/Quit | App ถูกปิดสนิท |

### ติดตั้ง Dependencies

```bash
# สำหรับ Firebase
npm install @react-native-firebase/app
npm install @react-native-firebase/messaging

# สำหรับ Local Notifications
npm install @notifee/react-native

# หรือใช้ Expo
npx expo install expo-notifications
npx expo install expo-device
```

---

## 2. FCM (Firebase Cloud Messaging) {#fcm}

### การตั้งค่า Firebase Project

#### Android - google-services.json
```json
// วางไฟล์ google-services.json ใน android/app/
{
  "project_info": {
    "project_number": "123456789",
    "project_id": "my-app",
    "storage_bucket": "my-app.appspot.com"
  },
  "client": [
    {
      "client_info": {
        "mobilesdk_app_id": "1:123456789:android:abcdef",
        "android_client_info": {
          "package_name": "com.myapp"
        }
      }
    }
  ]
}
```

#### android/build.gradle
```gradle
buildscript {
    dependencies {
        classpath 'com.google.gms:google-services:4.3.15'
    }
}
```

#### android/app/build.gradle
```gradle
apply plugin: 'com.google.gms.google-services'

dependencies {
    implementation platform('com.google.firebase:firebase-bom:32.0.0')
    implementation 'com.google.firebase:firebase-messaging'
}
```

### การขอ Permission และรับ FCM Token

```typescript
// src/services/NotificationService.ts
import messaging from '@react-native-firebase/messaging';
import { Platform, Alert } from 'react-native';

class NotificationService {
  private static instance: NotificationService;
  private fcmToken: string | null = null;

  static getInstance(): NotificationService {
    if (!NotificationService.instance) {
      NotificationService.instance = new NotificationService();
    }
    return NotificationService.instance;
  }

  // ขอ Permission
  async requestPermission(): Promise<boolean> {
    try {
      if (Platform.OS === 'ios') {
        const authStatus = await messaging().requestPermission();
        const enabled =
          authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
          authStatus === messaging.AuthorizationStatus.PROVISIONAL;
        
        if (!enabled) {
          Alert.alert(
            'ต้องการการอนุญาต',
            'กรุณาเปิดการแจ้งเตือนในการตั้งค่าเพื่อรับการแจ้งเตือน',
            [
              { text: 'ยกเลิก', style: 'cancel' },
              { text: 'ตั้งค่า', onPress: () => Linking.openSettings() }
            ]
          );
        }
        return enabled;
      }
      return true; // Android ไม่ต้องขอ permission ก่อน API 33
    } catch (error) {
      console.error('Permission error:', error);
      return false;
    }
  }

  // รับ FCM Token
  async getFCMToken(): Promise<string | null> {
    try {
      // ตรวจสอบว่ามี token เก็บไว้แล้วหรือยัง
      const token = await messaging().getToken();
      this.fcmToken = token;
      console.log('FCM Token:', token);
      return token;
    } catch (error) {
      console.error('Get token error:', error);
      return null;
    }
  }

  // ลงทะเบียน Token กับ Server
  async registerTokenWithServer(userId: string, token: string): Promise<void> {
    try {
      await fetch('https://api.example.com/users/token', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${await AsyncStorage.getItem('auth_token')}`,
        },
        body: JSON.stringify({
          userId,
          fcmToken: token,
          platform: Platform.OS,
          deviceInfo: {
            brand: DeviceInfo.getBrand(),
            model: DeviceInfo.getModel(),
          },
        }),
      });
    } catch (error) {
      console.error('Register token error:', error);
    }
  }

  // ติดตาม Token Refresh
  onTokenRefresh(callback: (token: string) => void): () => void {
    return messaging().onTokenRefresh(async (newToken) => {
      this.fcmToken = newToken;
      callback(newToken);
    });
  }
}

export default NotificationService.getInstance();
```

### การส่ง Notification จาก Server (Node.js)

```javascript
// server/notificationSender.js
const admin = require('firebase-admin');
const serviceAccount = require('./serviceAccountKey.json');

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
});

// ส่งหา Token เดียว
async function sendToDevice(token, notification, data = {}) {
  const message = {
    token,
    notification: {
      title: notification.title,
      body: notification.body,
      imageUrl: notification.imageUrl,
    },
    data: {
      ...data,
      click_action: 'FLUTTER_NOTIFICATION_CLICK',
    },
    android: {
      priority: 'high',
      notification: {
        channelId: 'default',
        sound: 'default',
        priority: 'high',
        defaultVibrateTimings: true,
      },
    },
    apns: {
      payload: {
        aps: {
          sound: 'default',
          badge: 1,
          'content-available': 1,
        },
      },
    },
  };

  const response = await admin.messaging().send(message);
  console.log('Notification sent:', response);
  return response;
}

// ส่งหา Topic
async function sendToTopic(topic, notification, data = {}) {
  const message = {
    topic,
    notification,
    data,
  };

  return await admin.messaging().send(message);
}

// ส่งหาหลาย Tokens
async function sendToMultipleDevices(tokens, notification, data = {}) {
  const message = {
    notification,
    data,
    tokens, // สูงสุด 500 tokens
  };

  const response = await admin.messaging().sendEachForMulticast(message);
  
  // จัดการ tokens ที่ไม่ valid
  response.responses.forEach((resp, idx) => {
    if (!resp.success) {
      console.error(`Failed for token ${tokens[idx]}:`, resp.error);
      // ลบ token ที่ไม่ valid ออกจาก database
      if (resp.error.code === 'messaging/invalid-registration-token' ||
          resp.error.code === 'messaging/registration-token-not-registered') {
        removeInvalidToken(tokens[idx]);
      }
    }
  });

  return response;
}

module.exports = { sendToDevice, sendToTopic, sendToMultipleDevices };
```

---

## 3. APNs (Apple Push Notification Service) {#apns}

### การตั้งค่า APNs

```
1. ไปที่ Apple Developer Account
2. Certificates, Identifiers & Profiles
3. Keys → Create a new key
4. เลือก Apple Push Notifications service (APNs)
5. Download .p8 file
```

### iOS Configuration

```xml
<!-- ios/YourApp/Info.plist -->
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
    <string>remote-notification</string>
</array>
```

```swift
// ios/YourApp/AppDelegate.swift
import UIKit
import Firebase
import UserNotifications

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate, UNUserNotificationCenterDelegate {

  func application(_ application: UIApplication,
                   didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    FirebaseApp.configure()
    
    UNUserNotificationCenter.current().delegate = self
    
    // ลงทะเบียน Remote Notifications
    application.registerForRemoteNotifications()
    
    return true
  }

  // รับ APNs Token
  func application(_ application: UIApplication,
                   didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    Messaging.messaging().apnsToken = deviceToken
  }
  
  // Handle Notification ขณะ App เปิดอยู่ (Foreground)
  func userNotificationCenter(_ center: UNUserNotificationCenter,
                               willPresent notification: UNNotification,
                               withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
    completionHandler([.banner, .badge, .sound])
  }
}
```

### การใช้งาน APNs ผ่าน Firebase (iOS)

```typescript
// ตั้งค่า iOS Notifications
import messaging from '@react-native-firebase/messaging';

async function setupiOSNotifications() {
  // Request permission
  const authStatus = await messaging().requestPermission({
    alert: true,
    badge: true,
    sound: true,
    provisional: false, // true = ได้รับ permission แบบชั่วคราว
    announcement: false,
    carPlay: false,
    criticalAlert: false,
  });
  
  if (authStatus === messaging.AuthorizationStatus.AUTHORIZED) {
    console.log('User granted full permission');
  } else if (authStatus === messaging.AuthorizationStatus.PROVISIONAL) {
    console.log('User granted provisional permission');
  } else {
    console.log('User declined permission');
  }
}
```

---

## 4. Local Notifications {#local-notifications}

### ติดตั้ง Notifee

```bash
npm install @notifee/react-native
cd ios && pod install
```

### สร้าง Notification Channel (Android)

```typescript
// src/services/LocalNotificationService.ts
import notifee, {
  AndroidImportance,
  AndroidVisibility,
  EventType,
  TriggerType,
  RepeatFrequency,
} from '@notifee/react-native';

class LocalNotificationService {
  // สร้าง Channel สำหรับ Android
  async createChannels(): Promise<void> {
    // Channel ทั่วไป
    await notifee.createChannel({
      id: 'default',
      name: 'General Notifications',
      importance: AndroidImportance.HIGH,
      sound: 'default',
      vibration: true,
      vibrationPattern: [300, 500],
    });

    // Channel สำหรับ Messages
    await notifee.createChannel({
      id: 'messages',
      name: 'Messages',
      importance: AndroidImportance.HIGH,
      badge: true,
      sound: 'message_sound',
    });

    // Channel สำหรับ Promotions
    await notifee.createChannel({
      id: 'promotions',
      name: 'Promotions',
      importance: AndroidImportance.DEFAULT,
      badge: false,
    });
  }

  // แสดง Notification ทันที
  async displayNotification(options: {
    title: string;
    body: string;
    imageUrl?: string;
    data?: Record<string, string>;
    channelId?: string;
  }): Promise<string> {
    const notificationId = await notifee.displayNotification({
      title: options.title,
      body: options.body,
      data: options.data,
      android: {
        channelId: options.channelId || 'default',
        largeIcon: options.imageUrl,
        style: options.body.length > 50 ? {
          type: 'BIGTEXT' as any,
          text: options.body,
        } : undefined,
        pressAction: {
          id: 'default',
          launchActivity: 'default',
        },
        actions: [
          {
            title: 'ดูรายละเอียด',
            pressAction: { id: 'view' },
          },
          {
            title: 'ปิด',
            pressAction: { id: 'dismiss' },
          },
        ],
      },
      ios: {
        categoryId: 'general',
        foregroundPresentationOptions: {
          badge: true,
          banner: true,
          sound: true,
        },
      },
    });

    return notificationId;
  }

  // ตั้งเวลา Notification
  async scheduleNotification(options: {
    title: string;
    body: string;
    date: Date;
    repeat?: 'daily' | 'weekly' | 'monthly';
  }): Promise<void> {
    const trigger = {
      type: TriggerType.TIMESTAMP,
      timestamp: options.date.getTime(),
      repeatFrequency: options.repeat 
        ? options.repeat === 'daily' 
          ? RepeatFrequency.DAILY
          : options.repeat === 'weekly'
            ? RepeatFrequency.WEEKLY
            : undefined
        : undefined,
    };

    await notifee.createTriggerNotification(
      {
        title: options.title,
        body: options.body,
        android: { channelId: 'default' },
      },
      trigger
    );
  }

  // ยกเลิก Notification
  async cancelNotification(notificationId: string): Promise<void> {
    await notifee.cancelNotification(notificationId);
  }

  // ยกเลิกทั้งหมด
  async cancelAllNotifications(): Promise<void> {
    await notifee.cancelAllNotifications();
  }

  // รับ Notifications ที่ pending
  async getScheduledNotifications() {
    return await notifee.getTriggerNotificationIds();
  }
}

export default new LocalNotificationService();
```

### Notification แบบ Progress Bar

```typescript
async function showProgressNotification() {
  const notificationId = 'upload-progress';
  
  // แสดง notification เริ่มต้น
  await notifee.displayNotification({
    id: notificationId,
    title: 'กำลังอัพโหลดไฟล์',
    body: 'เริ่มต้นอัพโหลด...',
    android: {
      channelId: 'default',
      ongoing: true, // ไม่ให้ user dismiss ได้
      progress: {
        max: 100,
        current: 0,
        indeterminate: false,
      },
    },
  });

  // อัพเดต progress
  for (let progress = 0; progress <= 100; progress += 10) {
    await notifee.displayNotification({
      id: notificationId,
      title: 'กำลังอัพโหลดไฟล์',
      body: `อัพโหลด ${progress}%`,
      android: {
        channelId: 'default',
        ongoing: progress < 100,
        progress: {
          max: 100,
          current: progress,
          indeterminate: false,
        },
      },
    });

    await new Promise(resolve => setTimeout(resolve, 500));
  }

  // แสดง notification สำเร็จ
  await notifee.displayNotification({
    id: notificationId,
    title: 'อัพโหลดสำเร็จ',
    body: 'ไฟล์ถูกอัพโหลดเรียบร้อยแล้ว',
    android: {
      channelId: 'default',
      ongoing: false,
    },
  });
}
```

---

## 5. Notification Handling {#notification-handling}

### การจัดการ Notifications ใน App

```typescript
// src/hooks/useNotifications.ts
import { useEffect, useCallback } from 'react';
import messaging from '@react-native-firebase/messaging';
import notifee, { EventType } from '@notifee/react-native';
import { useNavigation } from '@react-navigation/native';
import NotificationService from '../services/NotificationService';

export function useNotifications() {
  const navigation = useNavigation();

  const handleNotification = useCallback((notification: any) => {
    const { data } = notification;
    
    if (!data) return;

    // Navigate ตาม notification type
    switch (data.type) {
      case 'message':
        navigation.navigate('Chat', { chatId: data.chatId });
        break;
      case 'order':
        navigation.navigate('OrderDetail', { orderId: data.orderId });
        break;
      case 'promo':
        navigation.navigate('Promotion', { promoId: data.promoId });
        break;
      default:
        navigation.navigate('Notifications');
    }
  }, [navigation]);

  useEffect(() => {
    // 1. App อยู่ใน Foreground
    const unsubscribeForeground = messaging().onMessage(async (remoteMessage) => {
      console.log('Foreground notification:', remoteMessage);
      
      // แสดง Local Notification แทน
      await notifee.displayNotification({
        title: remoteMessage.notification?.title,
        body: remoteMessage.notification?.body,
        data: remoteMessage.data as any,
        android: { channelId: 'default' },
      });
    });

    // 2. App อยู่ใน Background แล้วกด notification
    messaging().onNotificationOpenedApp((remoteMessage) => {
      console.log('Background notification opened:', remoteMessage);
      handleNotification(remoteMessage);
    });

    // 3. App ถูกปิดแล้วเปิดผ่าน notification
    messaging().getInitialNotification().then((remoteMessage) => {
      if (remoteMessage) {
        console.log('Killed state notification:', remoteMessage);
        handleNotification(remoteMessage);
      }
    });

    // 4. Notifee foreground events
    const unsubscribeNotifee = notifee.onForegroundEvent(({ type, detail }) => {
      switch (type) {
        case EventType.DISMISSED:
          console.log('Notification dismissed:', detail.notification?.id);
          break;
        case EventType.PRESS:
          console.log('Notification pressed:', detail.notification);
          handleNotification(detail.notification);
          break;
        case EventType.ACTION_PRESS:
          console.log('Action pressed:', detail.pressAction?.id);
          handleAction(detail.pressAction?.id, detail.notification);
          break;
      }
    });

    return () => {
      unsubscribeForeground();
      unsubscribeNotifee();
    };
  }, [handleNotification]);

  const handleAction = (actionId: string | undefined, notification: any) => {
    switch (actionId) {
      case 'reply':
        // เปิด reply input
        break;
      case 'view':
        handleNotification(notification);
        break;
      case 'dismiss':
        notifee.cancelNotification(notification.id);
        break;
    }
  };
}

// Background Handler (ต้องอยู่นอก component)
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  console.log('Background message:', remoteMessage);
  // ทำงานใน background เช่น sync data
});

// Notifee Background Handler
notifee.onBackgroundEvent(async ({ type, detail }) => {
  if (type === EventType.ACTION_PRESS && detail.pressAction?.id === 'mark-read') {
    await markNotificationAsRead(detail.notification?.data?.notificationId);
    await notifee.cancelNotification(detail.notification!.id!);
  }
});
```

### การตั้งค่า Badge Count

```typescript
// จัดการ Badge Count
class BadgeManager {
  static async setBadgeCount(count: number): Promise<void> {
    if (Platform.OS === 'ios') {
      await notifee.setBadgeCount(count);
    } else {
      // Android ใช้ ShortcutBadger หรือ notifee
      await notifee.incrementBadgeCount(count);
    }
  }

  static async clearBadge(): Promise<void> {
    await notifee.setBadgeCount(0);
  }

  static async getBadgeCount(): Promise<number> {
    return await notifee.getBadgeCount();
  }
}
```

---

## 6. Notification Categories {#notification-categories}

### iOS Notification Categories

```typescript
// ตั้งค่า Categories สำหรับ iOS
async function setupiOSCategories() {
  await notifee.setNotificationCategories([
    {
      id: 'message',
      actions: [
        {
          id: 'reply',
          title: 'ตอบกลับ',
          input: {
            placeholder: 'พิมพ์ข้อความ...',
            buttonTitle: 'ส่ง',
          },
        },
        {
          id: 'mark-read',
          title: 'อ่านแล้ว',
          destructive: false,
          foreground: false,
        },
      ],
    },
    {
      id: 'order',
      actions: [
        {
          id: 'track',
          title: 'ติดตามคำสั่งซื้อ',
          foreground: true,
        },
        {
          id: 'cancel',
          title: 'ยกเลิกคำสั่งซื้อ',
          destructive: true,
          foreground: false,
        },
      ],
    },
  ]);
}
```

### Android Notification Channels Group

```typescript
// สร้าง Channel Groups
async function createChannelGroups() {
  await notifee.createChannelGroup({
    id: 'social',
    name: 'โซเชียล',
  });

  await notifee.createChannelGroup({
    id: 'commerce',
    name: 'การซื้อขาย',
  });

  // สร้าง Channels ใน Groups
  await notifee.createChannel({
    id: 'social-messages',
    name: 'ข้อความ',
    groupId: 'social',
    importance: AndroidImportance.HIGH,
  });

  await notifee.createChannel({
    id: 'social-likes',
    name: 'ไลค์และคอมเมนต์',
    groupId: 'social',
    importance: AndroidImportance.DEFAULT,
  });

  await notifee.createChannel({
    id: 'order-updates',
    name: 'อัพเดตคำสั่งซื้อ',
    groupId: 'commerce',
    importance: AndroidImportance.HIGH,
  });
}
```

---

## 7. Workshop: Notification System {#workshop}

### โปรเจกต์: ระบบ Notification ครบวงจร

```typescript
// src/screens/NotificationScreen.tsx
import React, { useEffect, useState } from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  StyleSheet,
  Switch,
  Alert,
} from 'react-native';
import Icon from 'react-native-vector-icons/MaterialIcons';
import notifee from '@notifee/react-native';
import messaging from '@react-native-firebase/messaging';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface NotificationItem {
  id: string;
  title: string;
  body: string;
  timestamp: Date;
  read: boolean;
  type: 'message' | 'order' | 'promo' | 'system';
  data?: Record<string, string>;
}

interface NotificationSettings {
  messages: boolean;
  orders: boolean;
  promotions: boolean;
  system: boolean;
}

const NotificationScreen: React.FC = () => {
  const [notifications, setNotifications] = useState<NotificationItem[]>([]);
  const [settings, setSettings] = useState<NotificationSettings>({
    messages: true,
    orders: true,
    promotions: true,
    system: true,
  });
  const [permissionGranted, setPermissionGranted] = useState(false);

  useEffect(() => {
    loadNotifications();
    loadSettings();
    checkPermission();
    subscribeToTopics();
  }, []);

  const checkPermission = async () => {
    const authStatus = await messaging().hasPermission();
    const enabled =
      authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
      authStatus === messaging.AuthorizationStatus.PROVISIONAL;
    setPermissionGranted(enabled);
  };

  const requestPermission = async () => {
    const granted = await NotificationService.requestPermission();
    setPermissionGranted(granted);
    if (granted) {
      const token = await NotificationService.getFCMToken();
      if (token) {
        await NotificationService.registerTokenWithServer('user_id', token);
      }
    }
  };

  const subscribeToTopics = async () => {
    if (settings.promotions) {
      await messaging().subscribeToTopic('promotions');
    }
    if (settings.system) {
      await messaging().subscribeToTopic('system');
    }
  };

  const loadNotifications = async () => {
    try {
      const stored = await AsyncStorage.getItem('notifications');
      if (stored) {
        const parsed = JSON.parse(stored);
        setNotifications(parsed.map((n: any) => ({
          ...n,
          timestamp: new Date(n.timestamp),
        })));
      }
    } catch (error) {
      console.error('Load notifications error:', error);
    }
  };

  const loadSettings = async () => {
    try {
      const stored = await AsyncStorage.getItem('notification_settings');
      if (stored) {
        setSettings(JSON.parse(stored));
      }
    } catch (error) {
      console.error('Load settings error:', error);
    }
  };

  const saveSettings = async (newSettings: NotificationSettings) => {
    setSettings(newSettings);
    await AsyncStorage.setItem('notification_settings', JSON.stringify(newSettings));
    
    // Update topic subscriptions
    if (newSettings.promotions) {
      await messaging().subscribeToTopic('promotions');
    } else {
      await messaging().unsubscribeFromTopic('promotions');
    }
  };

  const markAsRead = async (id: string) => {
    const updated = notifications.map(n =>
      n.id === id ? { ...n, read: true } : n
    );
    setNotifications(updated);
    await AsyncStorage.setItem('notifications', JSON.stringify(updated));
    await notifee.setBadgeCount(updated.filter(n => !n.read).length);
  };

  const markAllAsRead = async () => {
    const updated = notifications.map(n => ({ ...n, read: true }));
    setNotifications(updated);
    await AsyncStorage.setItem('notifications', JSON.stringify(updated));
    await notifee.setBadgeCount(0);
  };

  const deleteNotification = async (id: string) => {
    const updated = notifications.filter(n => n.id !== id);
    setNotifications(updated);
    await AsyncStorage.setItem('notifications', JSON.stringify(updated));
  };

  const testNotification = async () => {
    await notifee.displayNotification({
      title: '🔔 ทดสอบ Notification',
      body: 'นี่คือการทดสอบระบบแจ้งเตือน',
      android: {
        channelId: 'default',
        smallIcon: 'ic_notification',
        color: '#FF6B6B',
        pressAction: { id: 'default' },
      },
      ios: {
        foregroundPresentationOptions: {
          badge: true,
          banner: true,
          sound: true,
        },
      },
    });
  };

  const getTypeIcon = (type: string) => {
    switch (type) {
      case 'message': return 'chat';
      case 'order': return 'shopping-bag';
      case 'promo': return 'local-offer';
      default: return 'notifications';
    }
  };

  const getTypeColor = (type: string) => {
    switch (type) {
      case 'message': return '#4CAF50';
      case 'order': return '#2196F3';
      case 'promo': return '#FF9800';
      default: return '#9E9E9E';
    }
  };

  const renderNotification = ({ item }: { item: NotificationItem }) => (
    <TouchableOpacity
      style={[styles.notificationItem, !item.read && styles.unread]}
      onPress={() => markAsRead(item.id)}
      onLongPress={() => {
        Alert.alert('ลบ', 'ต้องการลบการแจ้งเตือนนี้?', [
          { text: 'ยกเลิก', style: 'cancel' },
          { text: 'ลบ', style: 'destructive', onPress: () => deleteNotification(item.id) },
        ]);
      }}
    >
      <View style={[styles.iconContainer, { backgroundColor: getTypeColor(item.type) + '20' }]}>
        <Icon name={getTypeIcon(item.type)} size={24} color={getTypeColor(item.type)} />
      </View>
      <View style={styles.content}>
        <Text style={styles.title}>{item.title}</Text>
        <Text style={styles.body} numberOfLines={2}>{item.body}</Text>
        <Text style={styles.time}>
          {formatRelativeTime(item.timestamp)}
        </Text>
      </View>
      {!item.read && <View style={styles.unreadDot} />}
    </TouchableOpacity>
  );

  const formatRelativeTime = (date: Date): string => {
    const now = new Date();
    const diff = now.getTime() - date.getTime();
    const minutes = Math.floor(diff / 60000);
    const hours = Math.floor(diff / 3600000);
    const days = Math.floor(diff / 86400000);

    if (minutes < 1) return 'เมื่อกี้';
    if (minutes < 60) return `${minutes} นาทีที่แล้ว`;
    if (hours < 24) return `${hours} ชั่วโมงที่แล้ว`;
    return `${days} วันที่แล้ว`;
  };

  const unreadCount = notifications.filter(n => !n.read).length;

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>
          การแจ้งเตือน
          {unreadCount > 0 && (
            <Text style={styles.badge}> ({unreadCount})</Text>
          )}
        </Text>
        {unreadCount > 0 && (
          <TouchableOpacity onPress={markAllAsRead}>
            <Text style={styles.markAllRead}>อ่านทั้งหมด</Text>
          </TouchableOpacity>
        )}
      </View>

      {/* Permission Banner */}
      {!permissionGranted && (
        <TouchableOpacity style={styles.permissionBanner} onPress={requestPermission}>
          <Icon name="notifications-off" size={20} color="#FF5722" />
          <Text style={styles.permissionText}>
            เปิดการแจ้งเตือนเพื่อรับข้อมูลล่าสุด
          </Text>
          <Icon name="chevron-right" size={20} color="#FF5722" />
        </TouchableOpacity>
      )}

      {/* Settings */}
      <View style={styles.settingsSection}>
        <Text style={styles.sectionTitle}>การตั้งค่าการแจ้งเตือน</Text>
        {Object.entries(settings).map(([key, value]) => (
          <View key={key} style={styles.settingRow}>
            <Text style={styles.settingLabel}>
              {key === 'messages' ? 'ข้อความ' :
               key === 'orders' ? 'คำสั่งซื้อ' :
               key === 'promotions' ? 'โปรโมชั่น' : 'ระบบ'}
            </Text>
            <Switch
              value={value}
              onValueChange={(newValue) => saveSettings({ ...settings, [key]: newValue })}
              trackColor={{ false: '#E0E0E0', true: '#4CAF5080' }}
              thumbColor={value ? '#4CAF50' : '#9E9E9E'}
            />
          </View>
        ))}
      </View>

      {/* Test Button */}
      <TouchableOpacity style={styles.testButton} onPress={testNotification}>
        <Icon name="notifications-active" size={20} color="white" />
        <Text style={styles.testButtonText}>ทดสอบ Notification</Text>
      </TouchableOpacity>

      {/* Notification List */}
      <FlatList
        data={notifications}
        renderItem={renderNotification}
        keyExtractor={item => item.id}
        ListEmptyComponent={
          <View style={styles.empty}>
            <Icon name="notifications-none" size={64} color="#E0E0E0" />
            <Text style={styles.emptyText}>ไม่มีการแจ้งเตือน</Text>
          </View>
        }
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#F5F5F5',
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: 'white',
    borderBottomWidth: 1,
    borderBottomColor: '#E0E0E0',
  },
  headerTitle: {
    fontSize: 18,
    fontWeight: 'bold',
  },
  badge: {
    color: '#FF5722',
  },
  markAllRead: {
    color: '#2196F3',
    fontSize: 14,
  },
  permissionBanner: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 12,
    backgroundColor: '#FFF3E0',
    borderBottomWidth: 1,
    borderBottomColor: '#FFE0B2',
    gap: 8,
  },
  permissionText: {
    flex: 1,
    color: '#FF5722',
    fontSize: 14,
  },
  settingsSection: {
    backgroundColor: 'white',
    margin: 16,
    borderRadius: 12,
    padding: 16,
  },
  sectionTitle: {
    fontSize: 16,
    fontWeight: 'bold',
    marginBottom: 12,
  },
  settingRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 8,
    borderTopWidth: 1,
    borderTopColor: '#F5F5F5',
  },
  settingLabel: {
    fontSize: 14,
    color: '#333',
  },
  testButton: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: '#2196F3',
    margin: 16,
    marginTop: 0,
    padding: 12,
    borderRadius: 8,
    gap: 8,
  },
  testButtonText: {
    color: 'white',
    fontWeight: 'bold',
  },
  notificationItem: {
    flexDirection: 'row',
    backgroundColor: 'white',
    padding: 16,
    marginHorizontal: 16,
    marginBottom: 8,
    borderRadius: 12,
    gap: 12,
  },
  unread: {
    backgroundColor: '#E3F2FD',
  },
  iconContainer: {
    width: 48,
    height: 48,
    borderRadius: 24,
    justifyContent: 'center',
    alignItems: 'center',
  },
  content: {
    flex: 1,
  },
  title: {
    fontSize: 15,
    fontWeight: 'bold',
    marginBottom: 4,
  },
  body: {
    fontSize: 13,
    color: '#666',
    marginBottom: 4,
  },
  time: {
    fontSize: 12,
    color: '#9E9E9E',
  },
  unreadDot: {
    width: 10,
    height: 10,
    borderRadius: 5,
    backgroundColor: '#2196F3',
    alignSelf: 'center',
  },
  empty: {
    alignItems: 'center',
    padding: 48,
    gap: 12,
  },
  emptyText: {
    color: '#9E9E9E',
    fontSize: 16,
  },
});

export default NotificationScreen;
```

---

## Tips และ Best Practices

### 1. การจัดการ Token

```typescript
// จัดการ token อย่างถูกต้อง
class TokenManager {
  private static TOKEN_KEY = 'fcm_token';

  static async saveToken(token: string): Promise<void> {
    await AsyncStorage.setItem(this.TOKEN_KEY, token);
  }

  static async getToken(): Promise<string | null> {
    return await AsyncStorage.getItem(this.TOKEN_KEY);
  }

  static async refreshTokenIfNeeded(): Promise<void> {
    const currentToken = await messaging().getToken();
    const savedToken = await this.getToken();

    if (currentToken !== savedToken) {
      await this.saveToken(currentToken);
      await this.syncTokenWithServer(currentToken);
    }
  }

  private static async syncTokenWithServer(token: string): Promise<void> {
    // อัพเดต token ใน server
  }
}
```

### 2. Deep Linking จาก Notifications

```typescript
// ตั้งค่า Deep Linking
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: 'home',
      Chat: 'chat/:chatId',
      OrderDetail: 'order/:orderId',
    },
  },
};

// Handle notification ที่ทำให้ app เปิด
messaging().getInitialNotification().then((message) => {
  if (message?.data?.url) {
    // Navigate ด้วย URL
    Linking.openURL(message.data.url);
  }
});
```

### 3. Rich Notifications (iOS)

```typescript
// iOS Rich Notifications พร้อมรูปภาพ
await notifee.displayNotification({
  title: 'คุณมีข้อความใหม่',
  body: 'ส่งรูปภาพมาให้คุณ',
  ios: {
    attachments: [
      {
        url: 'https://example.com/image.jpg',
        thumbnailHidden: false,
      },
    ],
    categoryId: 'message',
  },
});
```

### สรุป

- ใช้ FCM สำหรับ Android และ APNs ผ่าน Firebase สำหรับ iOS
- ตรวจสอบ permission ก่อนส่ง notification เสมอ
- จัดการ notification ใน 3 สถานะ: foreground, background, killed
- ใช้ notification categories สำหรับ actionable notifications
- Track และ refresh FCM token เมื่อมีการเปลี่ยนแปลง
- ลบ token ที่ invalid ออกจาก server ทันที
