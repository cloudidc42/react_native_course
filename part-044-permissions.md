# Part 044: Permissions Management ใน React Native

## สารบัญ
1. [แนะนำ Permissions](#introduction)
2. [react-native-permissions](#react-native-permissions)
3. [Camera Permission](#camera-permission)
4. [Location Permission](#location-permission)
5. [Notification Permission](#notification-permission)
6. [Storage Permission](#storage-permission)
7. [Handling Denied Permissions](#handling-denied)
8. [Workshop: Permissions Manager](#workshop)

---

## 1. แนะนำ Permissions {#introduction}

การจัดการ Permissions เป็นสิ่งสำคัญในการพัฒนาแอป Mobile เพื่อให้ผู้ใช้รู้สึกปลอดภัยและควบคุมความเป็นส่วนตัวของตนเองได้

### ประเภทของ Permissions

```
Permissions
├── Dangerous Permissions (ต้องขออนุญาต)
│   ├── Camera
│   ├── Microphone
│   ├── Location (Fine/Coarse)
│   ├── Contacts
│   ├── Storage (Read/Write)
│   ├── Phone
│   └── SMS
└── Normal Permissions (ไม่ต้องขออนุญาต)
    ├── Internet
    ├── Network State
    └── Vibrate
```

### สถานะของ Permission

| สถานะ | คำอธิบาย |
|-------|---------|
| UNAVAILABLE | อุปกรณ์ไม่รองรับ |
| DENIED | ผู้ใช้ปฏิเสธ (สามารถถามใหม่ได้) |
| BLOCKED | ผู้ใช้บล็อก (ต้องไปที่ Settings) |
| GRANTED | ได้รับอนุญาต |
| LIMITED | ได้รับบางส่วน (iOS) |

### ติดตั้ง

```bash
npm install react-native-permissions
cd ios && pod install
```

---

## 2. react-native-permissions {#react-native-permissions}

### การตั้งค่า Permissions.android.ts

```typescript
// Podfile (iOS)
// ต้องเพิ่ม permissions ที่ต้องการใช้
target 'YourApp' do
  permissions_path = '../node_modules/react-native-permissions/ios'
  
  pod 'Permission-Camera', :path => "#{permissions_path}/Camera"
  pod 'Permission-LocationWhenInUse', :path => "#{permissions_path}/LocationWhenInUse"
  pod 'Permission-LocationAlways', :path => "#{permissions_path}/LocationAlways"
  pod 'Permission-Microphone', :path => "#{permissions_path}/Microphone"
  pod 'Permission-Notifications', :path => "#{permissions_path}/Notifications"
  pod 'Permission-PhotoLibrary', :path => "#{permissions_path}/PhotoLibrary"
  pod 'Permission-MediaLibrary', :path => "#{permissions_path}/MediaLibrary"
end
```

### Permission Manager Class

```typescript
// src/services/PermissionManager.ts
import {
  Permission,
  PERMISSIONS,
  RESULTS,
  check,
  request,
  checkMultiple,
  requestMultiple,
  openSettings,
  PermissionStatus,
} from 'react-native-permissions';
import { Platform, Alert, Linking } from 'react-native';

type PermissionKey =
  | 'camera'
  | 'microphone'
  | 'locationFine'
  | 'locationCoarse'
  | 'locationAlways'
  | 'notifications'
  | 'photoLibrary'
  | 'mediaLibrary'
  | 'contacts'
  | 'storage';

interface PermissionInfo {
  android: Permission;
  ios: Permission;
  title: string;
  message: string;
  icon: string;
}

class PermissionManager {
  private static permissionMap: Record<PermissionKey, PermissionInfo> = {
    camera: {
      android: PERMISSIONS.ANDROID.CAMERA,
      ios: PERMISSIONS.IOS.CAMERA,
      title: 'กล้อง',
      message: 'แอปต้องการเข้าถึงกล้องเพื่อถ่ายรูปและสแกน QR Code',
      icon: '📷',
    },
    microphone: {
      android: PERMISSIONS.ANDROID.RECORD_AUDIO,
      ios: PERMISSIONS.IOS.MICROPHONE,
      title: 'ไมโครโฟน',
      message: 'แอปต้องการเข้าถึงไมโครโฟนเพื่อบันทึกเสียง',
      icon: '🎤',
    },
    locationFine: {
      android: PERMISSIONS.ANDROID.ACCESS_FINE_LOCATION,
      ios: PERMISSIONS.IOS.LOCATION_WHEN_IN_USE,
      title: 'ตำแหน่งที่แม่นยำ',
      message: 'แอปต้องการตำแหน่งที่แม่นยำเพื่อแสดงสถานที่ใกล้เคียง',
      icon: '📍',
    },
    locationCoarse: {
      android: PERMISSIONS.ANDROID.ACCESS_COARSE_LOCATION,
      ios: PERMISSIONS.IOS.LOCATION_WHEN_IN_USE,
      title: 'ตำแหน่งโดยประมาณ',
      message: 'แอปต้องการตำแหน่งโดยประมาณ',
      icon: '🗺️',
    },
    locationAlways: {
      android: PERMISSIONS.ANDROID.ACCESS_BACKGROUND_LOCATION,
      ios: PERMISSIONS.IOS.LOCATION_ALWAYS,
      title: 'ตำแหน่งตลอดเวลา',
      message: 'แอปต้องการตำแหน่งตลอดเวลาเพื่อการติดตาม',
      icon: '🌍',
    },
    notifications: {
      android: PERMISSIONS.ANDROID.POST_NOTIFICATIONS,
      ios: PERMISSIONS.IOS.NOTIFICATIONS,
      title: 'การแจ้งเตือน',
      message: 'แอปต้องการส่งการแจ้งเตือนเพื่อแจ้งข่าวสาร',
      icon: '🔔',
    },
    photoLibrary: {
      android: PERMISSIONS.ANDROID.READ_MEDIA_IMAGES,
      ios: PERMISSIONS.IOS.PHOTO_LIBRARY,
      title: 'คลังรูปภาพ',
      message: 'แอปต้องการเข้าถึงคลังรูปภาพ',
      icon: '🖼️',
    },
    mediaLibrary: {
      android: PERMISSIONS.ANDROID.READ_MEDIA_VIDEO,
      ios: PERMISSIONS.IOS.MEDIA_LIBRARY,
      title: 'คลังสื่อ',
      message: 'แอปต้องการเข้าถึงคลังสื่อ',
      icon: '🎬',
    },
    contacts: {
      android: PERMISSIONS.ANDROID.READ_CONTACTS,
      ios: PERMISSIONS.IOS.CONTACTS,
      title: 'รายชื่อผู้ติดต่อ',
      message: 'แอปต้องการเข้าถึงรายชื่อผู้ติดต่อ',
      icon: '👥',
    },
    storage: {
      android: PERMISSIONS.ANDROID.WRITE_EXTERNAL_STORAGE,
      ios: PERMISSIONS.IOS.PHOTO_LIBRARY,
      title: 'พื้นที่จัดเก็บ',
      message: 'แอปต้องการเข้าถึงพื้นที่จัดเก็บ',
      icon: '💾',
    },
  };

  // ตรวจสอบ Permission
  static async checkPermission(key: PermissionKey): Promise<PermissionStatus> {
    const info = this.permissionMap[key];
    const permission = Platform.OS === 'ios' ? info.ios : info.android;
    return await check(permission);
  }

  // ขอ Permission
  static async requestPermission(key: PermissionKey): Promise<PermissionStatus> {
    const info = this.permissionMap[key];
    const permission = Platform.OS === 'ios' ? info.ios : info.android;

    // ตรวจสอบสถานะก่อน
    const currentStatus = await check(permission);

    if (currentStatus === RESULTS.GRANTED) return RESULTS.GRANTED;
    if (currentStatus === RESULTS.UNAVAILABLE) return RESULTS.UNAVAILABLE;

    if (currentStatus === RESULTS.BLOCKED) {
      // ต้องไปที่ Settings
      this.showBlockedAlert(info);
      return RESULTS.BLOCKED;
    }

    // ขอ Permission
    return await request(permission, {
      title: `${info.icon} ${info.title}`,
      message: info.message,
      buttonPositive: 'อนุญาต',
      buttonNegative: 'ปฏิเสธ',
    });
  }

  // ตรวจสอบหลาย Permissions พร้อมกัน
  static async checkMultiplePermissions(
    keys: PermissionKey[]
  ): Promise<Record<PermissionKey, PermissionStatus>> {
    const permissions = keys.reduce((acc, key) => {
      const info = this.permissionMap[key];
      acc[key] = Platform.OS === 'ios' ? info.ios : info.android;
      return acc;
    }, {} as Record<string, Permission>);

    const statuses = await checkMultiple(Object.values(permissions));
    
    return keys.reduce((acc, key, index) => {
      acc[key] = statuses[Object.values(permissions)[index]];
      return acc;
    }, {} as Record<PermissionKey, PermissionStatus>);
  }

  // ขอหลาย Permissions พร้อมกัน
  static async requestMultiplePermissions(
    keys: PermissionKey[]
  ): Promise<Record<PermissionKey, PermissionStatus>> {
    const permissions = keys.reduce((acc, key) => {
      const info = this.permissionMap[key];
      acc[Platform.OS === 'ios' ? info.ios : info.android] = key;
      return acc;
    }, {} as Record<Permission, PermissionKey>);

    const statuses = await requestMultiple(
      Object.keys(permissions) as Permission[]
    );

    return Object.entries(statuses).reduce((acc, [permission, status]) => {
      const key = permissions[permission as Permission];
      if (key) acc[key] = status;
      return acc;
    }, {} as Record<PermissionKey, PermissionStatus>);
  }

  // ตรวจสอบว่าได้รับ Permission หรือยัง
  static async hasPermission(key: PermissionKey): Promise<boolean> {
    const status = await this.checkPermission(key);
    return status === RESULTS.GRANTED || status === RESULTS.LIMITED;
  }

  // แสดง Alert เมื่อ Permission ถูก Block
  private static showBlockedAlert(info: PermissionInfo): void {
    Alert.alert(
      `${info.icon} ${info.title}`,
      `การเข้าถึง${info.title}ถูกปิดอยู่ กรุณาเปิดในการตั้งค่าของอุปกรณ์`,
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'เปิดการตั้งค่า',
          onPress: () => openSettings(),
        },
      ]
    );
  }

  // ดึงข้อมูล Permission Info
  static getPermissionInfo(key: PermissionKey): PermissionInfo {
    return this.permissionMap[key];
  }

  static getStatusText(status: PermissionStatus): string {
    switch (status) {
      case RESULTS.UNAVAILABLE: return 'ไม่รองรับ';
      case RESULTS.DENIED: return 'ปฏิเสธ';
      case RESULTS.LIMITED: return 'จำกัด';
      case RESULTS.GRANTED: return 'อนุญาต';
      case RESULTS.BLOCKED: return 'บล็อก';
      default: return 'ไม่ทราบ';
    }
  }

  static getStatusColor(status: PermissionStatus): string {
    switch (status) {
      case RESULTS.GRANTED: return '#4CAF50';
      case RESULTS.LIMITED: return '#FF9800';
      case RESULTS.DENIED: return '#F44336';
      case RESULTS.BLOCKED: return '#9E9E9E';
      default: return '#9E9E9E';
    }
  }
}

export default PermissionManager;
export type { PermissionKey };
```

---

## 3. Camera Permission {#camera-permission}

```typescript
// src/utils/cameraPermission.ts
import { RESULTS } from 'react-native-permissions';
import PermissionManager from '../services/PermissionManager';

export async function requestCameraPermission(): Promise<{
  camera: boolean;
  microphone: boolean;
}> {
  const statuses = await PermissionManager.requestMultiplePermissions([
    'camera',
    'microphone',
  ]);

  return {
    camera: statuses.camera === RESULTS.GRANTED,
    microphone: statuses.microphone === RESULTS.GRANTED,
  };
}

// HOC สำหรับ Component ที่ต้องการ Camera Permission
export function withCameraPermission<T extends object>(
  Component: React.ComponentType<T>
) {
  return function PermissionWrapper(props: T) {
    const [permissionStatus, setPermissionStatus] = useState<'checking' | 'granted' | 'denied'>('checking');

    useEffect(() => {
      checkAndRequestPermission();
    }, []);

    const checkAndRequestPermission = async () => {
      const hasCamera = await PermissionManager.hasPermission('camera');
      if (hasCamera) {
        setPermissionStatus('granted');
      } else {
        const status = await PermissionManager.requestPermission('camera');
        setPermissionStatus(status === RESULTS.GRANTED ? 'granted' : 'denied');
      }
    };

    if (permissionStatus === 'checking') {
      return <ActivityIndicator />;
    }

    if (permissionStatus === 'denied') {
      return (
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <Text>ต้องการสิทธิ์เข้าถึงกล้อง</Text>
          <TouchableOpacity onPress={checkAndRequestPermission}>
            <Text>ขออนุญาตอีกครั้ง</Text>
          </TouchableOpacity>
        </View>
      );
    }

    return <Component {...props} />;
  };
}
```

---

## 4. Location Permission {#location-permission}

```typescript
// src/utils/locationPermission.ts
import { Platform } from 'react-native';
import { RESULTS } from 'react-native-permissions';
import PermissionManager from '../services/PermissionManager';

export async function requestLocationPermission(
  type: 'whenInUse' | 'always' = 'whenInUse'
): Promise<boolean> {
  if (type === 'whenInUse') {
    const status = await PermissionManager.requestPermission('locationFine');
    return status === RESULTS.GRANTED;
  } else {
    // ต้องขอ whenInUse ก่อน แล้วจึงขอ always
    const whenInUseStatus = await PermissionManager.requestPermission('locationFine');
    if (whenInUseStatus !== RESULTS.GRANTED) return false;

    // สำหรับ Android 10+
    if (Platform.OS === 'android' && Platform.Version >= 29) {
      const alwaysStatus = await PermissionManager.requestPermission('locationAlways');
      return alwaysStatus === RESULTS.GRANTED;
    }

    return true;
  }
}

// ตรวจสอบ Background Location Permission (Android)
export async function checkBackgroundLocationPermission(): Promise<boolean> {
  if (Platform.OS === 'android' && Platform.Version >= 29) {
    return await PermissionManager.hasPermission('locationAlways');
  }
  return true;
}
```

---

## 5. Notification Permission {#notification-permission}

```typescript
// src/utils/notificationPermission.ts
import { Platform } from 'react-native';
import { RESULTS, checkNotifications, requestNotifications } from 'react-native-permissions';

interface NotificationSettings {
  alert: boolean;
  badge: boolean;
  sound: boolean;
  criticalAlert: boolean;
  lockScreen: boolean;
  notificationCenter: boolean;
}

export async function checkNotificationPermission(): Promise<{
  status: string;
  settings: NotificationSettings | null;
}> {
  const { status, settings } = await checkNotifications();
  return {
    status,
    settings: settings ? {
      alert: settings.alert ?? false,
      badge: settings.badge ?? false,
      sound: settings.sound ?? false,
      criticalAlert: settings.criticalAlert ?? false,
      lockScreen: settings.lockScreen ?? false,
      notificationCenter: settings.notificationCenter ?? false,
    } : null,
  };
}

export async function requestNotificationPermission(): Promise<boolean> {
  const { status } = await requestNotifications([
    'alert',
    'badge',
    'sound',
    'criticalAlert',
  ]);

  return status === RESULTS.GRANTED;
}
```

---

## 6. Storage Permission {#storage-permission}

```typescript
// src/utils/storagePermission.ts
import { Platform } from 'react-native';
import { RESULTS, PERMISSIONS, check, request } from 'react-native-permissions';

export async function requestStoragePermission(): Promise<boolean> {
  if (Platform.OS === 'ios') {
    // iOS ใช้ Photo Library permission
    const status = await request(PERMISSIONS.IOS.PHOTO_LIBRARY_ADD_ONLY);
    return status === RESULTS.GRANTED;
  }

  if (Platform.Version >= 33) {
    // Android 13+ ใช้ Media permissions แทน Storage
    const imageStatus = await request(PERMISSIONS.ANDROID.READ_MEDIA_IMAGES);
    const videoStatus = await request(PERMISSIONS.ANDROID.READ_MEDIA_VIDEO);
    return imageStatus === RESULTS.GRANTED || videoStatus === RESULTS.GRANTED;
  } else if (Platform.Version >= 29) {
    // Android 10-12
    const readStatus = await request(PERMISSIONS.ANDROID.READ_EXTERNAL_STORAGE);
    return readStatus === RESULTS.GRANTED;
  } else {
    // Android < 10
    const writeStatus = await request(PERMISSIONS.ANDROID.WRITE_EXTERNAL_STORAGE);
    return writeStatus === RESULTS.GRANTED;
  }
}
```

---

## 7. Handling Denied Permissions {#handling-denied}

### Permission Rationale Component

```typescript
// src/components/PermissionRationale.tsx
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Modal,
} from 'react-native';
import { openSettings } from 'react-native-permissions';
import Icon from 'react-native-vector-icons/MaterialIcons';

interface PermissionRationaleProps {
  visible: boolean;
  icon: string;
  title: string;
  message: string;
  features: string[];
  isBlocked: boolean;
  onAllow: () => void;
  onDeny: () => void;
}

const PermissionRationale: React.FC<PermissionRationaleProps> = ({
  visible,
  icon,
  title,
  message,
  features,
  isBlocked,
  onAllow,
  onDeny,
}) => (
  <Modal visible={visible} transparent animationType="slide">
    <View style={styles.overlay}>
      <View style={styles.container}>
        <Text style={styles.icon}>{icon}</Text>
        <Text style={styles.title}>{title}</Text>
        <Text style={styles.message}>{message}</Text>

        <View style={styles.features}>
          <Text style={styles.featuresTitle}>ฟีเจอร์ที่ต้องการ:</Text>
          {features.map((feature, index) => (
            <View key={index} style={styles.featureRow}>
              <Icon name="check-circle" size={16} color="#4CAF50" />
              <Text style={styles.featureText}>{feature}</Text>
            </View>
          ))}
        </View>

        {isBlocked && (
          <View style={styles.blockedNote}>
            <Icon name="info" size={16} color="#FF9800" />
            <Text style={styles.blockedText}>
              กรุณาเปิดสิทธิ์ในการตั้งค่าของอุปกรณ์
            </Text>
          </View>
        )}

        <View style={styles.buttons}>
          <TouchableOpacity style={styles.denyButton} onPress={onDeny}>
            <Text style={styles.denyText}>ไม่อนุญาต</Text>
          </TouchableOpacity>
          <TouchableOpacity
            style={styles.allowButton}
            onPress={isBlocked ? openSettings : onAllow}
          >
            <Text style={styles.allowText}>
              {isBlocked ? 'เปิดการตั้งค่า' : 'อนุญาต'}
            </Text>
          </TouchableOpacity>
        </View>
      </View>
    </View>
  </Modal>
);

const styles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'flex-end',
  },
  container: {
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    padding: 24,
    gap: 12,
  },
  icon: { fontSize: 48, textAlign: 'center' },
  title: { fontSize: 20, fontWeight: 'bold', textAlign: 'center' },
  message: { color: '#666', textAlign: 'center', lineHeight: 22 },
  features: { backgroundColor: '#F5F5F5', padding: 16, borderRadius: 12, gap: 8 },
  featuresTitle: { fontWeight: 'bold', marginBottom: 4 },
  featureRow: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  featureText: { color: '#333', fontSize: 14 },
  blockedNote: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
    backgroundColor: '#FFF3E0',
    padding: 12,
    borderRadius: 8,
  },
  blockedText: { color: '#E65100', flex: 1, fontSize: 13 },
  buttons: { flexDirection: 'row', gap: 12, marginTop: 8 },
  denyButton: {
    flex: 1,
    padding: 14,
    borderRadius: 8,
    borderWidth: 1,
    borderColor: '#E0E0E0',
    alignItems: 'center',
  },
  denyText: { color: '#666', fontWeight: '500' },
  allowButton: {
    flex: 2,
    padding: 14,
    borderRadius: 8,
    backgroundColor: '#2196F3',
    alignItems: 'center',
  },
  allowText: { color: 'white', fontWeight: 'bold' },
});

export default PermissionRationale;
```

---

## 8. Workshop: Permissions Manager {#workshop}

```typescript
// src/screens/PermissionsScreen.tsx
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  FlatList,
  TouchableOpacity,
  StyleSheet,
  RefreshControl,
  Switch,
} from 'react-native';
import { RESULTS, PermissionStatus, openSettings } from 'react-native-permissions';
import Icon from 'react-native-vector-icons/MaterialIcons';
import PermissionManager, { PermissionKey } from '../services/PermissionManager';

interface PermissionItem {
  key: PermissionKey;
  icon: string;
  title: string;
  description: string;
  status: PermissionStatus | null;
  required: boolean;
}

const PERMISSION_LIST: Omit<PermissionItem, 'status'>[] = [
  {
    key: 'camera',
    icon: '📷',
    title: 'กล้อง',
    description: 'ถ่ายรูปและสแกน QR Code',
    required: true,
  },
  {
    key: 'microphone',
    icon: '🎤',
    title: 'ไมโครโฟน',
    description: 'บันทึกเสียงและโทรวิดีโอ',
    required: false,
  },
  {
    key: 'locationFine',
    icon: '📍',
    title: 'ตำแหน่ง',
    description: 'แสดงสถานที่ใกล้เคียงและนำทาง',
    required: true,
  },
  {
    key: 'notifications',
    icon: '🔔',
    title: 'การแจ้งเตือน',
    description: 'รับการแจ้งเตือนและข่าวสาร',
    required: false,
  },
  {
    key: 'photoLibrary',
    icon: '🖼️',
    title: 'คลังรูปภาพ',
    description: 'เลือกและบันทึกรูปภาพ',
    required: false,
  },
  {
    key: 'contacts',
    icon: '👥',
    title: 'รายชื่อผู้ติดต่อ',
    description: 'เพิ่มเพื่อนจากรายชื่อ',
    required: false,
  },
];

const PermissionsScreen: React.FC = () => {
  const [permissions, setPermissions] = useState<PermissionItem[]>([]);
  const [loading, setLoading] = useState(true);
  const [refreshing, setRefreshing] = useState(false);

  const loadPermissions = useCallback(async () => {
    const keys = PERMISSION_LIST.map(p => p.key);
    const statuses = await PermissionManager.checkMultiplePermissions(keys);

    const items = PERMISSION_LIST.map(p => ({
      ...p,
      status: statuses[p.key] || null,
    }));

    setPermissions(items);
    setLoading(false);
    setRefreshing(false);
  }, []);

  useEffect(() => {
    loadPermissions();
  }, []);

  const handleTogglePermission = async (item: PermissionItem) => {
    if (item.status === RESULTS.GRANTED) {
      // ไม่สามารถเพิกถอน permission ใน app ได้ต้องไปที่ settings
      openSettings();
      return;
    }

    const newStatus = await PermissionManager.requestPermission(item.key);
    setPermissions(prev =>
      prev.map(p => p.key === item.key ? { ...p, status: newStatus } : p)
    );
  };

  const getGrantedCount = () =>
    permissions.filter(p => p.status === RESULTS.GRANTED || p.status === RESULTS.LIMITED).length;

  const renderPermissionItem = ({ item }: { item: PermissionItem }) => {
    const isGranted = item.status === RESULTS.GRANTED || item.status === RESULTS.LIMITED;
    const isBlocked = item.status === RESULTS.BLOCKED;
    const statusText = item.status ? PermissionManager.getStatusText(item.status) : 'กำลังโหลด...';
    const statusColor = item.status ? PermissionManager.getStatusColor(item.status) : '#9E9E9E';

    return (
      <View style={[
        styles.permissionItem,
        item.required && !isGranted && styles.requiredItem,
      ]}>
        <Text style={styles.permissionIcon}>{item.icon}</Text>
        <View style={styles.permissionInfo}>
          <View style={styles.permissionHeader}>
            <Text style={styles.permissionTitle}>{item.title}</Text>
            {item.required && (
              <View style={styles.requiredBadge}>
                <Text style={styles.requiredBadgeText}>จำเป็น</Text>
              </View>
            )}
          </View>
          <Text style={styles.permissionDesc}>{item.description}</Text>
          <View style={styles.statusRow}>
            <View style={[styles.statusDot, { backgroundColor: statusColor }]} />
            <Text style={[styles.statusText, { color: statusColor }]}>{statusText}</Text>
            {isBlocked && (
              <TouchableOpacity onPress={openSettings}>
                <Text style={styles.settingsLink}>เปิดการตั้งค่า</Text>
              </TouchableOpacity>
            )}
          </View>
        </View>
        <Switch
          value={isGranted}
          onValueChange={() => handleTogglePermission(item)}
          trackColor={{ false: '#E0E0E0', true: '#4CAF5080' }}
          thumbColor={isGranted ? '#4CAF50' : '#9E9E9E'}
          disabled={item.status === RESULTS.UNAVAILABLE}
        />
      </View>
    );
  };

  const grantedCount = getGrantedCount();
  const progressPercent = permissions.length > 0
    ? (grantedCount / permissions.length) * 100
    : 0;

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>จัดการสิทธิ์การเข้าถึง</Text>
        <Text style={styles.headerSubtitle}>
          {grantedCount}/{permissions.length} สิทธิ์ที่อนุญาต
        </Text>
        <View style={styles.progressBar}>
          <View style={[styles.progressFill, { width: `${progressPercent}%` }]} />
        </View>
      </View>

      {/* Tips */}
      <View style={styles.tipsContainer}>
        <Icon name="info-outline" size={16} color="#2196F3" />
        <Text style={styles.tipsText}>
          สิทธิ์ที่ "จำเป็น" ต้องได้รับการอนุญาตเพื่อใช้งานแอปได้อย่างสมบูรณ์
        </Text>
      </View>

      {/* Permission List */}
      <FlatList
        data={permissions}
        renderItem={renderPermissionItem}
        keyExtractor={item => item.key}
        refreshControl={
          <RefreshControl
            refreshing={refreshing}
            onRefresh={() => {
              setRefreshing(true);
              loadPermissions();
            }}
          />
        }
        contentContainerStyle={styles.listContent}
      />

      {/* Request All Button */}
      {permissions.some(p =>
        p.status !== RESULTS.GRANTED &&
        p.status !== RESULTS.UNAVAILABLE &&
        p.status !== RESULTS.LIMITED
      ) && (
        <View style={styles.footer}>
          <TouchableOpacity
            style={styles.requestAllButton}
            onPress={async () => {
              const keys = permissions
                .filter(p => p.status !== RESULTS.GRANTED && p.status !== RESULTS.UNAVAILABLE)
                .map(p => p.key);
              const statuses = await PermissionManager.requestMultiplePermissions(keys);
              setPermissions(prev =>
                prev.map(p => ({
                  ...p,
                  status: statuses[p.key] || p.status,
                }))
              );
            }}
          >
            <Icon name="security" size={20} color="white" />
            <Text style={styles.requestAllText}>ขอสิทธิ์ทั้งหมด</Text>
          </TouchableOpacity>
        </View>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  header: { backgroundColor: 'white', padding: 20, gap: 4 },
  headerTitle: { fontSize: 20, fontWeight: 'bold' },
  headerSubtitle: { color: '#666', fontSize: 14 },
  progressBar: { height: 6, backgroundColor: '#E0E0E0', borderRadius: 3, marginTop: 8 },
  progressFill: { height: '100%', backgroundColor: '#4CAF50', borderRadius: 3 },
  tipsContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
    backgroundColor: '#E3F2FD',
    margin: 16,
    padding: 12,
    borderRadius: 8,
  },
  tipsText: { flex: 1, color: '#1565C0', fontSize: 13 },
  listContent: { padding: 16, gap: 8 },
  permissionItem: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 12,
    gap: 12,
  },
  requiredItem: { borderWidth: 1, borderColor: '#FFCCBC' },
  permissionIcon: { fontSize: 32 },
  permissionInfo: { flex: 1, gap: 2 },
  permissionHeader: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  permissionTitle: { fontSize: 15, fontWeight: 'bold' },
  requiredBadge: {
    backgroundColor: '#FF5722',
    paddingHorizontal: 6,
    paddingVertical: 2,
    borderRadius: 4,
  },
  requiredBadgeText: { color: 'white', fontSize: 10, fontWeight: 'bold' },
  permissionDesc: { color: '#666', fontSize: 13 },
  statusRow: { flexDirection: 'row', alignItems: 'center', gap: 4, marginTop: 4 },
  statusDot: { width: 8, height: 8, borderRadius: 4 },
  statusText: { fontSize: 12, fontWeight: '500' },
  settingsLink: { color: '#2196F3', fontSize: 12, marginLeft: 8 },
  footer: { padding: 16, backgroundColor: 'white', borderTopWidth: 1, borderTopColor: '#E0E0E0' },
  requestAllButton: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: '#2196F3',
    padding: 14,
    borderRadius: 12,
    gap: 8,
  },
  requestAllText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
});

export default PermissionsScreen;
```

---

## Tips และ Best Practices

### 1. Permission Request Timing

```typescript
// ขอ permission ในเวลาที่เหมาะสม ไม่ขอทันทีเมื่อเปิดแอป
// Bad: ขอ permission ทันทีเมื่อเปิดแอป
useEffect(() => {
  requestCameraPermission(); // ❌
}, []);

// Good: ขอเมื่อผู้ใช้จะใช้งาน feature นั้น
const handleCameraPress = async () => {
  const granted = await requestCameraPermission(); // ✅
  if (granted) openCamera();
};
```

### 2. Progressive Permission Request

```typescript
// ขอ permission แบบ Progressive
async function requestPermissionsProgressively() {
  // 1. ขอ permission ที่จำเป็นก่อน
  const cameraGranted = await PermissionManager.requestPermission('camera');
  if (!cameraGranted) {
    // แจ้งว่าฟีเจอร์ใดจะไม่สามารถใช้ได้
    showFeaturesUnavailable(['ถ่ายรูป', 'สแกน QR']);
    return;
  }
  
  // 2. ขอ optional permissions
  await PermissionManager.requestPermission('microphone');
  await PermissionManager.requestPermission('locationFine');
}
```

### สรุป

- ขอ permission เฉพาะเมื่อจำเป็นและอธิบายเหตุผล
- จัดการทุกสถานะของ permission: granted, denied, blocked
- ให้ผู้ใช้ยังสามารถใช้แอปได้แม้ไม่อนุญาต permission บางอย่าง
- ใช้ openSettings() เมื่อ permission ถูก block
- ตรวจสอบ permission ทุกครั้งที่ app กลับมา foreground
