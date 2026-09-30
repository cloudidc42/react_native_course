# Part 086: Bluetooth และ IoT ใน React Native

## บทนำ

Bluetooth Low Energy (BLE) เป็นเทคโนโลยีที่สำคัญในการพัฒนา IoT applications ใน React Native เราสามารถใช้ library `react-native-ble-plx` เพื่อเชื่อมต่อกับอุปกรณ์ BLE ต่างๆ เช่น สมาร์ทวอทช์, เซนเซอร์, และอุปกรณ์ IoT อื่นๆ

## เนื้อหาที่จะเรียน

1. การติดตั้งและตั้งค่า react-native-ble-plx
2. การสแกนหาอุปกรณ์ BLE
3. การเชื่อมต่อและตัดการเชื่อมต่อ
4. การอ่านและเขียน characteristics
5. Workshop: IoT Device Control

---

## 1. การติดตั้ง react-native-ble-plx

### การติดตั้ง

```bash
npm install react-native-ble-plx
# หรือ
yarn add react-native-ble-plx
```

### การตั้งค่า Android

แก้ไขไฟล์ `android/app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
  
  <!-- Bluetooth permissions -->
  <uses-permission android:name="android.permission.BLUETOOTH" />
  <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
  <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
  <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
  
  <!-- BLE feature -->
  <uses-feature
    android:name="android.hardware.bluetooth_le"
    android:required="true" />
    
  <application ...>
    ...
  </application>
</manifest>
```

### การตั้งค่า iOS

แก้ไขไฟล์ `ios/[ProjectName]/Info.plist`:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>แอปนี้ต้องการเข้าถึง Bluetooth เพื่อเชื่อมต่อกับอุปกรณ์ IoT</string>
<key>NSBluetoothPeripheralUsageDescription</key>
<string>แอปนี้ต้องการเข้าถึง Bluetooth Peripheral</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>แอปนี้ต้องการ Location สำหรับ Bluetooth scanning</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปนี้ต้องการ Location สำหรับ Bluetooth scanning</string>
```

แก้ไขไฟล์ `ios/Podfile`:

```ruby
platform :ios, '13.0'

target 'YourApp' do
  config = use_native_modules!
  
  use_react_native!(
    :path => config[:reactNativePath],
  )
  
  pod 'react-native-ble-plx', :path => '../node_modules/react-native-ble-plx'
end
```

---

## 2. การสร้าง BLE Manager

### BleManager.ts

```typescript
import { BleManager, Device, State } from 'react-native-ble-plx';
import { Platform, PermissionsAndroid } from 'react-native';

class BluetoothManager {
  private manager: BleManager;
  private static instance: BluetoothManager;

  private constructor() {
    this.manager = new BleManager();
  }

  // Singleton pattern
  static getInstance(): BluetoothManager {
    if (!BluetoothManager.instance) {
      BluetoothManager.instance = new BluetoothManager();
    }
    return BluetoothManager.instance;
  }

  // ตรวจสอบสถานะ Bluetooth
  async checkBluetoothState(): Promise<State> {
    return new Promise((resolve) => {
      this.manager.onStateChange((state) => {
        resolve(state);
      }, true);
    });
  }

  // ขอ permissions สำหรับ Android
  async requestPermissions(): Promise<boolean> {
    if (Platform.OS === 'android') {
      const apiLevel = Platform.Version;
      
      if (apiLevel >= 31) {
        // Android 12+
        const results = await PermissionsAndroid.requestMultiple([
          PermissionsAndroid.PERMISSIONS.BLUETOOTH_SCAN,
          PermissionsAndroid.PERMISSIONS.BLUETOOTH_CONNECT,
          PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
        ]);
        
        return (
          results['android.permission.BLUETOOTH_SCAN'] === 'granted' &&
          results['android.permission.BLUETOOTH_CONNECT'] === 'granted' &&
          results['android.permission.ACCESS_FINE_LOCATION'] === 'granted'
        );
      } else {
        // Android 11 และต่ำกว่า
        const result = await PermissionsAndroid.request(
          PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
          {
            title: 'ต้องการสิทธิ์ Location',
            message: 'แอปต้องการสิทธิ์ Location เพื่อใช้งาน Bluetooth',
            buttonPositive: 'ตกลง',
            buttonNegative: 'ยกเลิก',
          }
        );
        return result === 'granted';
      }
    }
    return true; // iOS จัดการ permissions ผ่าน Info.plist
  }

  getManager(): BleManager {
    return this.manager;
  }

  destroy() {
    this.manager.destroy();
  }
}

export default BluetoothManager;
```

---

## 3. การสแกนหาอุปกรณ์ BLE

### hooks/useBLEScanner.ts

```typescript
import { useState, useCallback, useRef } from 'react';
import { Device } from 'react-native-ble-plx';
import BluetoothManager from '../utils/BluetoothManager';

interface ScannedDevice {
  id: string;
  name: string | null;
  rssi: number | null;
  localName: string | null;
  device: Device;
}

export const useBLEScanner = () => {
  const [isScanning, setIsScanning] = useState(false);
  const [devices, setDevices] = useState<Map<string, ScannedDevice>>(new Map());
  const [error, setError] = useState<string | null>(null);
  const manager = BluetoothManager.getInstance().getManager();
  const scanTimeout = useRef<NodeJS.Timeout | null>(null);

  const startScan = useCallback(async () => {
    try {
      setError(null);
      setDevices(new Map());
      setIsScanning(true);

      // ตรวจสอบ permissions
      const hasPermission = await BluetoothManager.getInstance().requestPermissions();
      if (!hasPermission) {
        setError('ไม่ได้รับสิทธิ์ในการใช้ Bluetooth');
        setIsScanning(false);
        return;
      }

      // เริ่มสแกน
      manager.startDeviceScan(
        null, // UUIDs filter (null = ทุกอุปกรณ์)
        { allowDuplicates: false },
        (error, device) => {
          if (error) {
            console.error('Scan error:', error);
            setError(error.message);
            setIsScanning(false);
            return;
          }

          if (device) {
            setDevices((prevDevices) => {
              const newDevices = new Map(prevDevices);
              newDevices.set(device.id, {
                id: device.id,
                name: device.name,
                rssi: device.rssi,
                localName: device.localName,
                device: device,
              });
              return newDevices;
            });
          }
        }
      );

      // หยุดสแกนหลัง 10 วินาที
      scanTimeout.current = setTimeout(() => {
        stopScan();
      }, 10000);

    } catch (err) {
      setError('เกิดข้อผิดพลาดในการสแกน');
      setIsScanning(false);
    }
  }, [manager]);

  const stopScan = useCallback(() => {
    manager.stopDeviceScan();
    setIsScanning(false);
    if (scanTimeout.current) {
      clearTimeout(scanTimeout.current);
    }
  }, [manager]);

  return {
    isScanning,
    devices: Array.from(devices.values()),
    error,
    startScan,
    stopScan,
  };
};
```

### screens/ScanScreen.tsx

```typescript
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  FlatList,
  StyleSheet,
  ActivityIndicator,
  Alert,
} from 'react-native';
import { useBLEScanner } from '../hooks/useBLEScanner';
import { Device } from 'react-native-ble-plx';

interface DeviceItemProps {
  name: string | null;
  id: string;
  rssi: number | null;
  onConnect: () => void;
}

const DeviceItem: React.FC<DeviceItemProps> = ({ name, id, rssi, onConnect }) => {
  const signalStrength = rssi ? Math.min(Math.max((rssi + 100) * 2, 0), 100) : 0;
  
  return (
    <TouchableOpacity style={styles.deviceItem} onPress={onConnect}>
      <View style={styles.deviceInfo}>
        <Text style={styles.deviceName}>{name || 'Unknown Device'}</Text>
        <Text style={styles.deviceId}>{id}</Text>
        <View style={styles.signalContainer}>
          <Text style={styles.rssiText}>RSSI: {rssi} dBm</Text>
          <View style={[styles.signalBar, { width: `${signalStrength}%` }]} />
        </View>
      </View>
      <Text style={styles.connectText}>เชื่อมต่อ</Text>
    </TouchableOpacity>
  );
};

const ScanScreen: React.FC<{ navigation: any }> = ({ navigation }) => {
  const { isScanning, devices, error, startScan, stopScan } = useBLEScanner();

  const handleConnect = (device: Device) => {
    Alert.alert(
      'เชื่อมต่อ',
      `ต้องการเชื่อมต่อกับ ${device.name || device.id}?`,
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'เชื่อมต่อ',
          onPress: () => {
            stopScan();
            navigation.navigate('DeviceControl', { device });
          },
        },
      ]
    );
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>สแกนหาอุปกรณ์ BLE</Text>
      
      {error && (
        <View style={styles.errorContainer}>
          <Text style={styles.errorText}>{error}</Text>
        </View>
      )}
      
      <TouchableOpacity
        style={[styles.scanButton, isScanning && styles.stopButton]}
        onPress={isScanning ? stopScan : startScan}
      >
        {isScanning ? (
          <View style={styles.scanningContainer}>
            <ActivityIndicator color="white" size="small" />
            <Text style={styles.buttonText}> กำลังสแกน...</Text>
          </View>
        ) : (
          <Text style={styles.buttonText}>เริ่มสแกน</Text>
        )}
      </TouchableOpacity>

      <Text style={styles.deviceCount}>
        พบอุปกรณ์: {devices.length} เครื่อง
      </Text>

      <FlatList
        data={devices}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <DeviceItem
            name={item.name}
            id={item.id}
            rssi={item.rssi}
            onConnect={() => handleConnect(item.device)}
          />
        )}
        ListEmptyComponent={
          <View style={styles.emptyContainer}>
            <Text style={styles.emptyText}>
              {isScanning ? 'กำลังค้นหาอุปกรณ์...' : 'ไม่พบอุปกรณ์ กดสแกนเพื่อค้นหา'}
            </Text>
          </View>
        }
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
    padding: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 16,
  },
  errorContainer: {
    backgroundColor: '#ffebee',
    padding: 12,
    borderRadius: 8,
    marginBottom: 12,
  },
  errorText: {
    color: '#c62828',
  },
  scanButton: {
    backgroundColor: '#2196F3',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginBottom: 16,
  },
  stopButton: {
    backgroundColor: '#f44336',
  },
  scanningContainer: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  buttonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  deviceCount: {
    fontSize: 14,
    color: '#666',
    marginBottom: 8,
  },
  deviceItem: {
    backgroundColor: 'white',
    padding: 16,
    borderRadius: 12,
    marginBottom: 8,
    flexDirection: 'row',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 2,
  },
  deviceInfo: {
    flex: 1,
  },
  deviceName: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
  },
  deviceId: {
    fontSize: 12,
    color: '#999',
    marginTop: 2,
  },
  signalContainer: {
    marginTop: 8,
  },
  rssiText: {
    fontSize: 12,
    color: '#666',
  },
  signalBar: {
    height: 4,
    backgroundColor: '#4CAF50',
    borderRadius: 2,
    marginTop: 4,
  },
  connectText: {
    color: '#2196F3',
    fontWeight: '600',
  },
  emptyContainer: {
    alignItems: 'center',
    padding: 40,
  },
  emptyText: {
    color: '#999',
    textAlign: 'center',
  },
});

export default ScanScreen;
```

---

## 4. การเชื่อมต่อและตัดการเชื่อมต่อ

### hooks/useBLEConnection.ts

```typescript
import { useState, useCallback, useEffect } from 'react';
import { Device, Characteristic, BleError } from 'react-native-ble-plx';
import BluetoothManager from '../utils/BluetoothManager';

export type ConnectionStatus = 'disconnected' | 'connecting' | 'connected' | 'error';

export const useBLEConnection = (device: Device | null) => {
  const [status, setStatus] = useState<ConnectionStatus>('disconnected');
  const [connectedDevice, setConnectedDevice] = useState<Device | null>(null);
  const [error, setError] = useState<string | null>(null);
  const manager = BluetoothManager.getInstance().getManager();

  const connect = useCallback(async () => {
    if (!device) return;
    
    setStatus('connecting');
    setError(null);
    
    try {
      // เชื่อมต่อกับอุปกรณ์
      const connected = await manager.connectToDevice(device.id, {
        autoConnect: false,
        timeout: 10000,
      });
      
      // Discover services และ characteristics
      await connected.discoverAllServicesAndCharacteristics();
      
      setConnectedDevice(connected);
      setStatus('connected');
      
      // ตรวจสอบการตัดการเชื่อมต่อ
      manager.onDeviceDisconnected(connected.id, (error, device) => {
        if (error) {
          console.log('Disconnected with error:', error);
          setError('การเชื่อมต่อถูกตัด: ' + error.message);
        }
        setStatus('disconnected');
        setConnectedDevice(null);
      });
      
    } catch (err: any) {
      setStatus('error');
      setError(err.message || 'ไม่สามารถเชื่อมต่อได้');
    }
  }, [device, manager]);

  const disconnect = useCallback(async () => {
    if (!connectedDevice) return;
    
    try {
      await manager.cancelDeviceConnection(connectedDevice.id);
      setConnectedDevice(null);
      setStatus('disconnected');
    } catch (err: any) {
      setError(err.message);
    }
  }, [connectedDevice, manager]);

  // Cleanup เมื่อ component unmount
  useEffect(() => {
    return () => {
      if (connectedDevice) {
        manager.cancelDeviceConnection(connectedDevice.id).catch(console.error);
      }
    };
  }, [connectedDevice, manager]);

  return {
    status,
    connectedDevice,
    error,
    connect,
    disconnect,
  };
};
```

---

## 5. การอ่านและเขียน Characteristics

### hooks/useBLECharacteristics.ts

```typescript
import { useState, useCallback } from 'react';
import { Device, Characteristic } from 'react-native-ble-plx';
import { Buffer } from 'buffer';

interface ServiceInfo {
  uuid: string;
  characteristics: CharacteristicInfo[];
}

interface CharacteristicInfo {
  uuid: string;
  isReadable: boolean;
  isWritable: boolean;
  isNotifiable: boolean;
  value: string | null;
}

export const useBLECharacteristics = (device: Device | null) => {
  const [services, setServices] = useState<ServiceInfo[]>([]);
  const [loading, setLoading] = useState(false);

  // ดึงข้อมูล services และ characteristics
  const fetchServices = useCallback(async () => {
    if (!device) return;
    
    setLoading(true);
    try {
      const deviceServices = await device.services();
      const serviceInfos: ServiceInfo[] = [];
      
      for (const service of deviceServices) {
        const characteristics = await service.characteristics();
        const charInfos: CharacteristicInfo[] = characteristics.map(char => ({
          uuid: char.uuid,
          isReadable: char.isReadable,
          isWritable: char.isWritableWithResponse || char.isWritableWithoutResponse,
          isNotifiable: char.isNotifiable,
          value: null,
        }));
        
        serviceInfos.push({
          uuid: service.uuid,
          characteristics: charInfos,
        });
      }
      
      setServices(serviceInfos);
    } catch (err) {
      console.error('Error fetching services:', err);
    } finally {
      setLoading(false);
    }
  }, [device]);

  // อ่านค่าจาก characteristic
  const readCharacteristic = useCallback(async (
    serviceUUID: string,
    characteristicUUID: string
  ): Promise<string | null> => {
    if (!device) return null;
    
    try {
      const characteristic = await device.readCharacteristicForService(
        serviceUUID,
        characteristicUUID
      );
      
      if (characteristic.value) {
        // Decode base64 value
        const decoded = Buffer.from(characteristic.value, 'base64').toString('utf-8');
        return decoded;
      }
      return null;
    } catch (err) {
      console.error('Error reading characteristic:', err);
      return null;
    }
  }, [device]);

  // เขียนค่าลง characteristic
  const writeCharacteristic = useCallback(async (
    serviceUUID: string,
    characteristicUUID: string,
    value: string
  ): Promise<boolean> => {
    if (!device) return false;
    
    try {
      // Encode value เป็น base64
      const encoded = Buffer.from(value, 'utf-8').toString('base64');
      
      await device.writeCharacteristicWithResponseForService(
        serviceUUID,
        characteristicUUID,
        encoded
      );
      return true;
    } catch (err) {
      console.error('Error writing characteristic:', err);
      return false;
    }
  }, [device]);

  // Subscribe ไปยัง notifications
  const subscribeToCharacteristic = useCallback((
    serviceUUID: string,
    characteristicUUID: string,
    callback: (value: string) => void
  ) => {
    if (!device) return () => {};
    
    const subscription = device.monitorCharacteristicForService(
      serviceUUID,
      characteristicUUID,
      (error, characteristic) => {
        if (error) {
          console.error('Monitor error:', error);
          return;
        }
        
        if (characteristic?.value) {
          const decoded = Buffer.from(characteristic.value, 'base64').toString('utf-8');
          callback(decoded);
        }
      }
    );
    
    return () => subscription.remove();
  }, [device]);

  return {
    services,
    loading,
    fetchServices,
    readCharacteristic,
    writeCharacteristic,
    subscribeToCharacteristic,
  };
};
```

---

## 6. Workshop: IoT Device Control

### สร้างหน้า IoT Control Panel

สมมติว่าเราควบคุม Arduino/ESP32 ที่มี:
- LED control (on/off)
- Temperature sensor
- Humidity sensor

#### constants/bleUUIDs.ts

```typescript
// UUIDs สำหรับ IoT Device ของเรา
export const IOT_SERVICE_UUID = '12345678-1234-1234-1234-123456789012';

export const CHARACTERISTICS = {
  LED_CONTROL: '12345678-1234-1234-1234-123456789013',
  TEMPERATURE: '12345678-1234-1234-1234-123456789014',
  HUMIDITY: '12345678-1234-1234-1234-123456789015',
  RELAY_1: '12345678-1234-1234-1234-123456789016',
  RELAY_2: '12345678-1234-1234-1234-123456789017',
};

export const COMMANDS = {
  LED_ON: '1',
  LED_OFF: '0',
  RELAY_ON: '1',
  RELAY_OFF: '0',
};
```

#### screens/DeviceControlScreen.tsx

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
  Switch,
  ScrollView,
  Alert,
  ActivityIndicator,
} from 'react-native';
import { Device } from 'react-native-ble-plx';
import { useBLEConnection } from '../hooks/useBLEConnection';
import { useBLECharacteristics } from '../hooks/useBLECharacteristics';
import { IOT_SERVICE_UUID, CHARACTERISTICS, COMMANDS } from '../constants/bleUUIDs';

interface SensorData {
  temperature: number | null;
  humidity: number | null;
}

interface DeviceControlScreenProps {
  route: {
    params: {
      device: Device;
    };
  };
}

const DeviceControlScreen: React.FC<DeviceControlScreenProps> = ({ route }) => {
  const { device } = route.params;
  const { status, connectedDevice, error, connect, disconnect } = useBLEConnection(device);
  const { readCharacteristic, writeCharacteristic, subscribeToCharacteristic } = 
    useBLECharacteristics(connectedDevice);
  
  const [ledState, setLedState] = useState(false);
  const [relay1State, setRelay1State] = useState(false);
  const [relay2State, setRelay2State] = useState(false);
  const [sensorData, setSensorData] = useState<SensorData>({
    temperature: null,
    humidity: null,
  });
  const [isLoadingLED, setIsLoadingLED] = useState(false);

  // เชื่อมต่ออัตโนมัติ
  useEffect(() => {
    connect();
  }, []);

  // Subscribe ไปยัง sensor notifications
  useEffect(() => {
    if (status !== 'connected' || !connectedDevice) return;

    // Subscribe temperature
    const unsubTemp = subscribeToCharacteristic(
      IOT_SERVICE_UUID,
      CHARACTERISTICS.TEMPERATURE,
      (value) => {
        const temp = parseFloat(value);
        if (!isNaN(temp)) {
          setSensorData(prev => ({ ...prev, temperature: temp }));
        }
      }
    );

    // Subscribe humidity
    const unsubHumidity = subscribeToCharacteristic(
      IOT_SERVICE_UUID,
      CHARACTERISTICS.HUMIDITY,
      (value) => {
        const humidity = parseFloat(value);
        if (!isNaN(humidity)) {
          setSensorData(prev => ({ ...prev, humidity: humidity }));
        }
      }
    );

    // อ่านค่าเริ่มต้น
    readInitialValues();

    return () => {
      unsubTemp();
      unsubHumidity();
    };
  }, [status, connectedDevice]);

  const readInitialValues = async () => {
    try {
      // อ่านสถานะ LED
      const ledValue = await readCharacteristic(IOT_SERVICE_UUID, CHARACTERISTICS.LED_CONTROL);
      if (ledValue) setLedState(ledValue === '1');
      
      // อ่านค่า Temperature
      const tempValue = await readCharacteristic(IOT_SERVICE_UUID, CHARACTERISTICS.TEMPERATURE);
      if (tempValue) setSensorData(prev => ({ ...prev, temperature: parseFloat(tempValue) }));
      
    } catch (err) {
      console.error('Error reading initial values:', err);
    }
  };

  const toggleLED = async () => {
    if (!connectedDevice) return;
    
    setIsLoadingLED(true);
    try {
      const newState = !ledState;
      const command = newState ? COMMANDS.LED_ON : COMMANDS.LED_OFF;
      
      const success = await writeCharacteristic(
        IOT_SERVICE_UUID,
        CHARACTERISTICS.LED_CONTROL,
        command
      );
      
      if (success) {
        setLedState(newState);
      } else {
        Alert.alert('ข้อผิดพลาด', 'ไม่สามารถควบคุม LED ได้');
      }
    } finally {
      setIsLoadingLED(false);
    }
  };

  const toggleRelay = async (relayNum: 1 | 2) => {
    if (!connectedDevice) return;
    
    const currentState = relayNum === 1 ? relay1State : relay2State;
    const uuid = relayNum === 1 ? CHARACTERISTICS.RELAY_1 : CHARACTERISTICS.RELAY_2;
    const command = currentState ? COMMANDS.RELAY_OFF : COMMANDS.RELAY_ON;
    
    const success = await writeCharacteristic(IOT_SERVICE_UUID, uuid, command);
    
    if (success) {
      if (relayNum === 1) setRelay1State(!currentState);
      else setRelay2State(!currentState);
    }
  };

  const getStatusColor = () => {
    switch (status) {
      case 'connected': return '#4CAF50';
      case 'connecting': return '#FF9800';
      case 'error': return '#f44336';
      default: return '#9E9E9E';
    }
  };

  const getStatusText = () => {
    switch (status) {
      case 'connected': return 'เชื่อมต่อแล้ว';
      case 'connecting': return 'กำลังเชื่อมต่อ...';
      case 'error': return 'เกิดข้อผิดพลาด';
      default: return 'ไม่ได้เชื่อมต่อ';
    }
  };

  return (
    <ScrollView style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.deviceName}>{device.name || 'IoT Device'}</Text>
        <View style={[styles.statusDot, { backgroundColor: getStatusColor() }]} />
        <Text style={[styles.statusText, { color: getStatusColor() }]}>
          {getStatusText()}
        </Text>
      </View>

      {status === 'connecting' && (
        <View style={styles.loadingContainer}>
          <ActivityIndicator size="large" color="#2196F3" />
          <Text style={styles.loadingText}>กำลังเชื่อมต่อ...</Text>
        </View>
      )}

      {status === 'connected' && (
        <>
          {/* Sensor Data */}
          <View style={styles.sensorCard}>
            <Text style={styles.sectionTitle}>ข้อมูลเซนเซอร์</Text>
            <View style={styles.sensorRow}>
              <View style={styles.sensorItem}>
                <Text style={styles.sensorIcon}>🌡️</Text>
                <Text style={styles.sensorValue}>
                  {sensorData.temperature !== null 
                    ? `${sensorData.temperature.toFixed(1)}°C` 
                    : '--°C'}
                </Text>
                <Text style={styles.sensorLabel}>อุณหภูมิ</Text>
              </View>
              <View style={styles.sensorItem}>
                <Text style={styles.sensorIcon}>💧</Text>
                <Text style={styles.sensorValue}>
                  {sensorData.humidity !== null 
                    ? `${sensorData.humidity.toFixed(1)}%` 
                    : '--%'}
                </Text>
                <Text style={styles.sensorLabel}>ความชื้น</Text>
              </View>
            </View>
          </View>

          {/* LED Control */}
          <View style={styles.controlCard}>
            <Text style={styles.sectionTitle}>ควบคุมไฟ LED</Text>
            <View style={styles.controlRow}>
              <Text style={styles.controlLabel}>
                LED {ledState ? '🔆 เปิด' : '🔅 ปิด'}
              </Text>
              {isLoadingLED ? (
                <ActivityIndicator size="small" color="#2196F3" />
              ) : (
                <Switch
                  value={ledState}
                  onValueChange={toggleLED}
                  trackColor={{ false: '#767577', true: '#81b0ff' }}
                  thumbColor={ledState ? '#2196F3' : '#f4f3f4'}
                />
              )}
            </View>
          </View>

          {/* Relay Controls */}
          <View style={styles.controlCard}>
            <Text style={styles.sectionTitle}>ควบคุม Relay</Text>
            
            <View style={styles.controlRow}>
              <Text style={styles.controlLabel}>
                Relay 1 {relay1State ? '⚡ ON' : '○ OFF'}
              </Text>
              <Switch
                value={relay1State}
                onValueChange={() => toggleRelay(1)}
                trackColor={{ false: '#767577', true: '#81b0ff' }}
                thumbColor={relay1State ? '#2196F3' : '#f4f3f4'}
              />
            </View>
            
            <View style={[styles.controlRow, styles.separator]}>
              <Text style={styles.controlLabel}>
                Relay 2 {relay2State ? '⚡ ON' : '○ OFF'}
              </Text>
              <Switch
                value={relay2State}
                onValueChange={() => toggleRelay(2)}
                trackColor={{ false: '#767577', true: '#81b0ff' }}
                thumbColor={relay2State ? '#2196F3' : '#f4f3f4'}
              />
            </View>
          </View>

          {/* Quick Actions */}
          <View style={styles.controlCard}>
            <Text style={styles.sectionTitle}>Quick Actions</Text>
            <View style={styles.quickActions}>
              <TouchableOpacity 
                style={[styles.quickButton, styles.allOnButton]}
                onPress={async () => {
                  await toggleLED();
                  await toggleRelay(1);
                  await toggleRelay(2);
                }}
              >
                <Text style={styles.quickButtonText}>เปิดทั้งหมด</Text>
              </TouchableOpacity>
              
              <TouchableOpacity 
                style={[styles.quickButton, styles.allOffButton]}
                onPress={async () => {
                  if (ledState) await toggleLED();
                  if (relay1State) await toggleRelay(1);
                  if (relay2State) await toggleRelay(2);
                }}
              >
                <Text style={styles.quickButtonText}>ปิดทั้งหมด</Text>
              </TouchableOpacity>
            </View>
          </View>
        </>
      )}

      {/* Disconnect Button */}
      {status === 'connected' && (
        <TouchableOpacity style={styles.disconnectButton} onPress={disconnect}>
          <Text style={styles.disconnectText}>ตัดการเชื่อมต่อ</Text>
        </TouchableOpacity>
      )}

      {status === 'error' && (
        <TouchableOpacity style={styles.retryButton} onPress={connect}>
          <Text style={styles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      )}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    backgroundColor: 'white',
    padding: 20,
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 2,
  },
  deviceName: {
    fontSize: 20,
    fontWeight: 'bold',
    color: '#333',
    flex: 1,
  },
  statusDot: {
    width: 10,
    height: 10,
    borderRadius: 5,
    marginRight: 6,
  },
  statusText: {
    fontSize: 14,
    fontWeight: '500',
  },
  loadingContainer: {
    alignItems: 'center',
    padding: 40,
  },
  loadingText: {
    marginTop: 12,
    color: '#666',
  },
  sensorCard: {
    backgroundColor: 'white',
    margin: 16,
    marginTop: 0,
    padding: 16,
    borderRadius: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 2,
  },
  sectionTitle: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
    marginBottom: 12,
  },
  sensorRow: {
    flexDirection: 'row',
    justifyContent: 'space-around',
  },
  sensorItem: {
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#f8f9fa',
    borderRadius: 12,
    width: '45%',
  },
  sensorIcon: {
    fontSize: 32,
    marginBottom: 8,
  },
  sensorValue: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#2196F3',
  },
  sensorLabel: {
    fontSize: 12,
    color: '#666',
    marginTop: 4,
  },
  controlCard: {
    backgroundColor: 'white',
    margin: 16,
    marginTop: 0,
    padding: 16,
    borderRadius: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 2,
  },
  controlRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 8,
  },
  separator: {
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    marginTop: 8,
    paddingTop: 16,
  },
  controlLabel: {
    fontSize: 16,
    color: '#333',
  },
  quickActions: {
    flexDirection: 'row',
    gap: 12,
  },
  quickButton: {
    flex: 1,
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  allOnButton: {
    backgroundColor: '#4CAF50',
  },
  allOffButton: {
    backgroundColor: '#f44336',
  },
  quickButtonText: {
    color: 'white',
    fontWeight: '600',
  },
  disconnectButton: {
    margin: 16,
    padding: 16,
    backgroundColor: '#ff5252',
    borderRadius: 12,
    alignItems: 'center',
    marginBottom: 32,
  },
  disconnectText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  retryButton: {
    margin: 16,
    padding: 16,
    backgroundColor: '#2196F3',
    borderRadius: 12,
    alignItems: 'center',
  },
  retryText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default DeviceControlScreen;
```

---

## 7. App Navigation Setup

### App.tsx

```typescript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import ScanScreen from './screens/ScanScreen';
import DeviceControlScreen from './screens/DeviceControlScreen';

const Stack = createNativeStackNavigator();

const App: React.FC = () => {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen 
          name="Scan" 
          component={ScanScreen}
          options={{ title: 'BLE Scanner' }}
        />
        <Stack.Screen 
          name="DeviceControl" 
          component={DeviceControlScreen}
          options={{ title: 'IoT Control' }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

export default App;
```

---

## 8. Arduino/ESP32 Code (สำหรับทดสอบ)

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <DHT.h>

// Service UUID
#define SERVICE_UUID "12345678-1234-1234-1234-123456789012"
#define LED_CHAR_UUID "12345678-1234-1234-1234-123456789013"
#define TEMP_CHAR_UUID "12345678-1234-1234-1234-123456789014"
#define HUMIDITY_CHAR_UUID "12345678-1234-1234-1234-123456789015"
#define RELAY1_CHAR_UUID "12345678-1234-1234-1234-123456789016"
#define RELAY2_CHAR_UUID "12345678-1234-1234-1234-123456789017"

#define LED_PIN 2
#define RELAY1_PIN 14
#define RELAY2_PIN 27
#define DHT_PIN 4
#define DHT_TYPE DHT22

DHT dht(DHT_PIN, DHT_TYPE);
BLECharacteristic *tempCharacteristic;
BLECharacteristic *humidityCharacteristic;
bool deviceConnected = false;

class ServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer* server) {
    deviceConnected = true;
  }
  void onDisconnect(BLEServer* server) {
    deviceConnected = false;
    BLEDevice::startAdvertising();
  }
};

class LEDCallback : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *characteristic) {
    String value = characteristic->getValue().c_str();
    digitalWrite(LED_PIN, value == "1" ? HIGH : LOW);
  }
};

void setup() {
  Serial.begin(115200);
  dht.begin();
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);

  BLEDevice::init("IoT-Controller");
  BLEServer *server = BLEDevice::createServer();
  server->setCallbacks(new ServerCallbacks());
  
  BLEService *service = server->createService(SERVICE_UUID);
  
  // LED Characteristic
  BLECharacteristic *ledChar = service->createCharacteristic(
    LED_CHAR_UUID,
    BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_WRITE
  );
  ledChar->setCallbacks(new LEDCallback());
  
  // Temperature Characteristic
  tempCharacteristic = service->createCharacteristic(
    TEMP_CHAR_UUID,
    BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_NOTIFY
  );
  tempCharacteristic->addDescriptor(new BLE2902());
  
  // Humidity Characteristic
  humidityCharacteristic = service->createCharacteristic(
    HUMIDITY_CHAR_UUID,
    BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_NOTIFY
  );
  humidityCharacteristic->addDescriptor(new BLE2902());
  
  service->start();
  BLEAdvertising *advertising = BLEDevice::getAdvertising();
  advertising->addServiceUUID(SERVICE_UUID);
  advertising->start();
}

void loop() {
  if (deviceConnected) {
    float temperature = dht.readTemperature();
    float humidity = dht.readHumidity();
    
    if (!isnan(temperature) && !isnan(humidity)) {
      char tempStr[10];
      char humStr[10];
      dtostrf(temperature, 4, 1, tempStr);
      dtostrf(humidity, 4, 1, humStr);
      
      tempCharacteristic->setValue(tempStr);
      tempCharacteristic->notify();
      
      humidityCharacteristic->setValue(humStr);
      humidityCharacteristic->notify();
    }
  }
  delay(2000);
}
```

---

## Best Practices

### 1. จัดการ Memory Leaks

```typescript
useEffect(() => {
  const subscription = manager.onStateChange(state => {
    if (state === 'PoweredOn') {
      startScan();
    }
  }, true);
  
  return () => {
    subscription.remove(); // สำคัญ! ต้อง cleanup
    stopScan();
  };
}, []);
```

### 2. Error Handling

```typescript
try {
  await device.connect();
} catch (error) {
  if (error.errorCode === BleErrorCode.DeviceConnectionFailed) {
    // ลองเชื่อมต่อใหม่
  } else if (error.errorCode === BleErrorCode.OperationCancelled) {
    // ผู้ใช้ยกเลิก
  }
}
```

### 3. Background Mode (iOS)

```typescript
// ใน Info.plist
// <key>UIBackgroundModes</key>
// <array>
//   <string>bluetooth-central</string>
// </array>
```

---

## Workshop Exercises

### แบบฝึกหัดที่ 1: สร้าง Device List with Favorites
- เพิ่ม favorite device ที่เคยเชื่อมต่อ
- บันทึก device ID ลงใน AsyncStorage
- แสดง favorites ก่อนรายการอื่น

### แบบฝึกหัดที่ 2: Auto-Reconnect
- เพิ่มระบบ auto-reconnect เมื่อการเชื่อมต่อหลุด
- สร้าง reconnect strategy (exponential backoff)
- แจ้งเตือนผู้ใช้เมื่อกำลัง reconnect

### แบบฝึกหัดที่ 3: Data Logging
- บันทึกข้อมูล sensor ทุก 30 วินาที
- แสดง graph ด้วย react-native-chart-kit
- Export ข้อมูลเป็น CSV

### แบบฝึกหัดที่ 4: Multiple Device Management
- เชื่อมต่อกับหลาย device พร้อมกัน
- สร้าง dashboard แสดงข้อมูลทุก device
- จัดการ connection pool

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **การติดตั้ง** react-native-ble-plx และตั้งค่า permissions
2. **BLE Scanning** - การค้นหาอุปกรณ์ BLE รอบข้าง
3. **การเชื่อมต่อ** - Connect/Disconnect กับอุปกรณ์ BLE
4. **Characteristics** - อ่านและเขียนข้อมูลผ่าน BLE
5. **Real-time data** - รับข้อมูลแบบ real-time ผ่าน notifications
6. **IoT Control** - สร้าง app ควบคุมอุปกรณ์ IoT จริง

> **Tips:** สำหรับ production app ควรเพิ่ม retry logic, error recovery, และ battery optimization เพื่อให้ app ทำงานได้เสถียรมากขึ้น
