# Part 055: Biometric Authentication ใน React Native

## ความเข้าใจ Biometric Authentication

Biometric Authentication คือการยืนยันตัวตนโดยใช้ข้อมูลทางชีววิทยา เช่น ลายนิ้วมือ (Touch ID/Fingerprint) หรือใบหน้า (Face ID) แทนการใช้รหัสผ่าน

### ทำไมต้องใช้ Biometric?

- **สะดวก**: ไม่ต้องจำรหัสผ่าน
- **ปลอดภัย**: ข้อมูล biometric เก็บใน Secure Enclave/TEE ไม่ส่งไปไหน
- **UX ดี**: login ได้เร็วกว่า
- **ลด friction**: ผู้ใช้ยอมทำ biometric มากกว่าพิมพ์รหัส

### ประเภท Biometric บน mobile

```
iOS:
- Face ID (iPhone X+, iPad Pro)
- Touch ID (iPhone 8-, iPad, MacBook)

Android:
- Fingerprint (ส่วนใหญ่ทุก device)
- Face Unlock (บางรุ่น)
- Iris Scanner (Samsung Galaxy บางรุ่น)
```

---

## react-native-biometrics

### การติดตั้ง

```bash
npm install react-native-biometrics
cd ios && pod install
```

### iOS - Info.plist

```xml
<key>NSFaceIDUsageDescription</key>
<string>แอปนี้ใช้ Face ID เพื่อรักษาความปลอดภัยของข้อมูล</string>
```

### Android - AndroidManifest.xml

```xml
<uses-permission android:name="android.permission.USE_BIOMETRIC" />
<uses-permission android:name="android.permission.USE_FINGERPRINT" />
```

---

## การใช้งานพื้นฐาน

```typescript
import ReactNativeBiometrics, { BiometryTypes } from 'react-native-biometrics';

const rnBiometrics = new ReactNativeBiometrics({ allowDeviceCredentials: true });

// ตรวจสอบว่า device รองรับ biometric หรือไม่
const checkBiometrics = async () => {
  const { biometryType, available, error } = await rnBiometrics.isSensorAvailable();
  
  if (!available) {
    console.log('Biometric not available:', error);
    return null;
  }
  
  switch (biometryType) {
    case BiometryTypes.FaceID:
      return 'Face ID';
    case BiometryTypes.TouchID:
      return 'Touch ID';
    case BiometryTypes.Biometrics:
      return 'Fingerprint';
    default:
      return null;
  }
};

// Simple Biometric Authentication
const authenticateSimple = async () => {
  const { success, error } = await rnBiometrics.simplePrompt({
    promptMessage: 'ยืนยันตัวตนเพื่อเข้าสู่ระบบ',
    cancelButtonText: 'ยกเลิก',
    fallbackPromptMessage: 'ใช้รหัส PIN แทน',
  });
  
  if (success) {
    console.log('Authentication successful');
  } else {
    console.log('Authentication failed:', error);
  }
  
  return success;
};
```

---

## Biometric with Cryptography

สำหรับการรักษาความปลอดภัยสูงสุด ควรใช้ biometric ร่วมกับ cryptographic signing

```typescript
import ReactNativeBiometrics from 'react-native-biometrics';
import AsyncStorage from '@react-native-async-storage/async-storage';

const rnBiometrics = new ReactNativeBiometrics();
const KEY_ALIAS = 'com.yourapp.biometric';

// สร้าง key pair สำหรับ biometric signing
const createBiometricKeys = async (): Promise<string | null> => {
  try {
    // ลบ keys เก่าถ้ามี
    await rnBiometrics.deleteKeys();
    
    const { publicKey } = await rnBiometrics.createKeys();
    
    // ส่ง publicKey ไปยัง server เพื่อเก็บไว้
    await registerPublicKeyWithServer(publicKey);
    
    // บันทึกว่า biometric setup แล้ว
    await AsyncStorage.setItem('@biometric_enabled', 'true');
    
    return publicKey;
  } catch (error) {
    console.error('Failed to create biometric keys:', error);
    return null;
  }
};

// Authenticate ด้วย biometric และ sign payload
const authenticateWithSignature = async (payload: string): Promise<string | null> => {
  try {
    const { success, signature, error } = await rnBiometrics.createSignature({
      promptMessage: 'ยืนยันตัวตนเพื่อเข้าสู่ระบบ',
      payload,
      cancelButtonText: 'ยกเลิก',
    });
    
    if (success && signature) {
      return signature;
    }
    
    console.log('Signature failed:', error);
    return null;
  } catch (error) {
    console.error('Biometric error:', error);
    return null;
  }
};

// ลงทะเบียน public key กับ server
const registerPublicKeyWithServer = async (publicKey: string): Promise<void> => {
  const userId = await AsyncStorage.getItem('@user_id');
  
  await fetch('https://api.yourapp.com/auth/biometric/register', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${await AsyncStorage.getItem('@auth_token')}`,
    },
    body: JSON.stringify({ publicKey, userId }),
  });
};

// ตรวจสอบ signature กับ server
const verifySignatureWithServer = async (
  signature: string,
  payload: string
): Promise<boolean> => {
  const userId = await AsyncStorage.getItem('@user_id');
  
  const response = await fetch('https://api.yourapp.com/auth/biometric/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ signature, payload, userId }),
  });
  
  const data = await response.json();
  return data.verified;
};
```

---

## Biometric Login Screen

```typescript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  Platform,
  KeyboardAvoidingView,
  ScrollView,
  ActivityIndicator,
} from 'react-native';
import ReactNativeBiometrics, { BiometryTypes } from 'react-native-biometrics';
import AsyncStorage from '@react-native-async-storage/async-storage';

const rnBiometrics = new ReactNativeBiometrics({ allowDeviceCredentials: true });

interface AuthState {
  isLoggedIn: boolean;
  biometricEnabled: boolean;
  biometricType: string | null;
  loading: boolean;
}

const BiometricLoginScreen: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [authState, setAuthState] = useState<AuthState>({
    isLoggedIn: false,
    biometricEnabled: false,
    biometricType: null,
    loading: true,
  });
  const [authLoading, setAuthLoading] = useState(false);

  useEffect(() => {
    initAuth();
  }, []);

  const initAuth = async () => {
    try {
      // ตรวจสอบว่า login อยู่หรือไม่
      const token = await AsyncStorage.getItem('@auth_token');
      
      // ตรวจสอบ biometric availability
      const { available, biometryType } = await rnBiometrics.isSensorAvailable();
      const biometricEnabled = await AsyncStorage.getItem('@biometric_enabled') === 'true';
      
      let biometricTypeName: string | null = null;
      if (available) {
        switch (biometryType) {
          case BiometryTypes.FaceID:
            biometricTypeName = 'Face ID';
            break;
          case BiometryTypes.TouchID:
            biometricTypeName = 'Touch ID';
            break;
          case BiometryTypes.Biometrics:
            biometricTypeName = 'ลายนิ้วมือ';
            break;
        }
      }
      
      setAuthState({
        isLoggedIn: !!token,
        biometricEnabled: available && biometricEnabled,
        biometricType: biometricTypeName,
        loading: false,
      });
      
      // Auto-trigger biometric ถ้าเปิดใช้งาน
      if (available && biometricEnabled && token) {
        setTimeout(() => loginWithBiometric(), 500);
      }
    } catch (error) {
      setAuthState(prev => ({ ...prev, loading: false }));
    }
  };

  const loginWithPassword = async () => {
    if (!email || !password) {
      Alert.alert('ข้อผิดพลาด', 'กรุณากรอก Email และ Password');
      return;
    }
    
    setAuthLoading(true);
    try {
      // จำลอง API call
      await new Promise(resolve => setTimeout(resolve, 1500));
      
      // Mock authentication
      if (email === 'test@example.com' && password === 'password123') {
        await AsyncStorage.setItem('@auth_token', 'mock-token-123');
        await AsyncStorage.setItem('@user_id', 'user-123');
        
        setAuthState(prev => ({ ...prev, isLoggedIn: true }));
        
        // ถามว่าต้องการเปิด biometric หรือไม่
        const { available } = await rnBiometrics.isSensorAvailable();
        if (available && authState.biometricType) {
          promptEnableBiometric();
        }
      } else {
        Alert.alert('Login ล้มเหลว', 'Email หรือ Password ไม่ถูกต้อง');
      }
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถ login ได้');
    } finally {
      setAuthLoading(false);
    }
  };

  const loginWithBiometric = async () => {
    setAuthLoading(true);
    try {
      // สร้าง payload ที่ไม่ซ้ำ (ควรมาจาก server เพื่อความปลอดภัย)
      const payload = `login:${Date.now()}:${await AsyncStorage.getItem('@user_id')}`;
      
      const { success, error } = await rnBiometrics.simplePrompt({
        promptMessage: `Login ด้วย ${authState.biometricType}`,
        cancelButtonText: 'ใช้รหัสผ่านแทน',
        fallbackPromptMessage: 'ใช้รหัส PIN แทน',
      });
      
      if (success) {
        // ในระบบจริง: สร้าง signature และส่ง verify กับ server
        await AsyncStorage.setItem('@last_biometric_login', new Date().toISOString());
        setAuthState(prev => ({ ...prev, isLoggedIn: true }));
        Alert.alert('เข้าสู่ระบบสำเร็จ', `ยืนยันตัวตนด้วย ${authState.biometricType}`);
      } else if (error) {
        console.log('Biometric failed:', error);
      }
    } catch (error) {
      console.error('Biometric error:', error);
    } finally {
      setAuthLoading(false);
    }
  };

  const promptEnableBiometric = () => {
    Alert.alert(
      `เปิดใช้ ${authState.biometricType}?`,
      `ต้องการใช้ ${authState.biometricType} ในการ login ครั้งต่อไปหรือไม่?`,
      [
        { text: 'ไม่ใช้', style: 'cancel' },
        {
          text: 'เปิดใช้งาน',
          onPress: enableBiometric,
        },
      ]
    );
  };

  const enableBiometric = async () => {
    try {
      await rnBiometrics.deleteKeys();
      const { publicKey } = await rnBiometrics.createKeys();
      
      await AsyncStorage.setItem('@biometric_enabled', 'true');
      
      setAuthState(prev => ({ ...prev, biometricEnabled: true }));
      Alert.alert('สำเร็จ', `เปิดใช้ ${authState.biometricType} แล้ว`);
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถเปิดใช้ biometric ได้');
    }
  };

  const disableBiometric = async () => {
    Alert.alert(
      'ปิดการใช้งาน Biometric',
      'คุณแน่ใจว่าต้องการปิดการใช้งาน?',
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'ปิด',
          style: 'destructive',
          onPress: async () => {
            await rnBiometrics.deleteKeys();
            await AsyncStorage.setItem('@biometric_enabled', 'false');
            setAuthState(prev => ({ ...prev, biometricEnabled: false }));
          },
        },
      ]
    );
  };

  const logout = async () => {
    await AsyncStorage.removeItem('@auth_token');
    setAuthState(prev => ({ ...prev, isLoggedIn: false }));
  };

  if (authState.loading) {
    return (
      <View style={bioStyles.loadingContainer}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    );
  }

  if (authState.isLoggedIn) {
    return (
      <View style={bioStyles.container}>
        <Text style={bioStyles.title}>เข้าสู่ระบบแล้ว ✅</Text>
        
        <View style={bioStyles.settingsCard}>
          <Text style={bioStyles.settingsTitle}>การตั้งค่า Biometric</Text>
          
          {authState.biometricType ? (
            <View style={bioStyles.settingRow}>
              <View>
                <Text style={bioStyles.settingLabel}>{authState.biometricType}</Text>
                <Text style={bioStyles.settingStatus}>
                  {authState.biometricEnabled ? 'เปิดใช้งาน' : 'ปิดใช้งาน'}
                </Text>
              </View>
              <TouchableOpacity
                style={[
                  bioStyles.toggleButton,
                  authState.biometricEnabled ? bioStyles.toggleOn : bioStyles.toggleOff,
                ]}
                onPress={authState.biometricEnabled ? disableBiometric : enableBiometric}
              >
                <Text style={bioStyles.toggleText}>
                  {authState.biometricEnabled ? 'ปิด' : 'เปิด'}
                </Text>
              </TouchableOpacity>
            </View>
          ) : (
            <Text style={bioStyles.noBiometric}>
              Device นี้ไม่รองรับ Biometric Authentication
            </Text>
          )}
        </View>
        
        <TouchableOpacity style={bioStyles.logoutButton} onPress={logout}>
          <Text style={bioStyles.logoutText}>ออกจากระบบ</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <KeyboardAvoidingView
      style={bioStyles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView contentContainerStyle={bioStyles.scrollContent}>
        <Text style={bioStyles.title}>เข้าสู่ระบบ</Text>
        <Text style={bioStyles.subtitle}>MySecure App</Text>

        {/* Login Form */}
        <View style={bioStyles.formCard}>
          <TextInput
            style={bioStyles.input}
            value={email}
            onChangeText={setEmail}
            placeholder="Email"
            keyboardType="email-address"
            autoCapitalize="none"
          />
          <TextInput
            style={bioStyles.input}
            value={password}
            onChangeText={setPassword}
            placeholder="Password"
            secureTextEntry
          />
          
          <TouchableOpacity
            style={bioStyles.loginButton}
            onPress={loginWithPassword}
            disabled={authLoading}
          >
            {authLoading ? (
              <ActivityIndicator color="white" />
            ) : (
              <Text style={bioStyles.loginButtonText}>เข้าสู่ระบบ</Text>
            )}
          </TouchableOpacity>
          
          <Text style={bioStyles.hintText}>
            ทดสอบ: test@example.com / password123
          </Text>
        </View>

        {/* Biometric Login */}
        {authState.biometricType && authState.biometricEnabled && (
          <View style={bioStyles.biometricSection}>
            <Text style={bioStyles.orText}>หรือ</Text>
            
            <TouchableOpacity
              style={bioStyles.biometricButton}
              onPress={loginWithBiometric}
              disabled={authLoading}
            >
              <Text style={bioStyles.biometricIcon}>
                {authState.biometricType === 'Face ID' ? '👤' : '👆'}
              </Text>
              <Text style={bioStyles.biometricText}>
                Login ด้วย {authState.biometricType}
              </Text>
            </TouchableOpacity>
          </View>
        )}
      </ScrollView>
    </KeyboardAvoidingView>
  );
};

const bioStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f0f2f5' },
  loadingContainer: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  scrollContent: { flexGrow: 1, justifyContent: 'center', padding: 24 },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#1a1a2e',
    textAlign: 'center',
    marginBottom: 4,
  },
  subtitle: {
    fontSize: 16,
    color: '#888',
    textAlign: 'center',
    marginBottom: 30,
  },
  formCard: {
    backgroundColor: 'white',
    borderRadius: 16,
    padding: 20,
    elevation: 4,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.15,
    shadowRadius: 8,
  },
  input: {
    borderWidth: 1,
    borderColor: '#e0e0e0',
    borderRadius: 10,
    padding: 14,
    fontSize: 15,
    marginBottom: 14,
    backgroundColor: '#fafafa',
  },
  loginButton: {
    backgroundColor: '#6200EE',
    padding: 16,
    borderRadius: 10,
    alignItems: 'center',
    marginTop: 5,
  },
  loginButtonText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
  hintText: { textAlign: 'center', color: '#bbb', fontSize: 12, marginTop: 12 },
  biometricSection: { alignItems: 'center', marginTop: 25 },
  orText: { color: '#999', fontSize: 14, marginBottom: 15 },
  biometricButton: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'white',
    paddingHorizontal: 24,
    paddingVertical: 14,
    borderRadius: 25,
    borderWidth: 2,
    borderColor: '#6200EE',
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
  biometricIcon: { fontSize: 24, marginRight: 10 },
  biometricText: { color: '#6200EE', fontWeight: 'bold', fontSize: 16 },
  settingsCard: {
    backgroundColor: 'white',
    borderRadius: 16,
    padding: 20,
    margin: 20,
    elevation: 3,
  },
  settingsTitle: { fontSize: 18, fontWeight: 'bold', color: '#333', marginBottom: 15 },
  settingRow: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' },
  settingLabel: { fontSize: 16, color: '#333', fontWeight: '500' },
  settingStatus: { fontSize: 13, color: '#888', marginTop: 3 },
  toggleButton: {
    paddingHorizontal: 20,
    paddingVertical: 8,
    borderRadius: 20,
  },
  toggleOn: { backgroundColor: '#4CAF50' },
  toggleOff: { backgroundColor: '#9E9E9E' },
  toggleText: { color: 'white', fontWeight: 'bold' },
  noBiometric: { color: '#999', fontSize: 14 },
  logoutButton: {
    margin: 20,
    backgroundColor: '#F44336',
    padding: 16,
    borderRadius: 10,
    alignItems: 'center',
  },
  logoutText: { color: 'white', fontWeight: 'bold', fontSize: 16 },
});

export default BiometricLoginScreen;
```

---

## Secure Storage สำหรับ Sensitive Data

```typescript
import * as Keychain from 'react-native-keychain';

class SecureStorageService {
  // บันทึกข้อมูล sensitive ที่ต้องการ biometric
  static async saveWithBiometric(key: string, value: string): Promise<boolean> {
    try {
      await Keychain.setGenericPassword(key, value, {
        service: key,
        accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_ANY_OR_DEVICE_PASSCODE,
        accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
        securityLevel: Keychain.SECURITY_LEVEL.SECURE_HARDWARE,
      });
      return true;
    } catch (error) {
      console.error('Failed to save with biometric:', error);
      return false;
    }
  }

  // อ่านข้อมูลที่ต้องใช้ biometric
  static async getWithBiometric(key: string, promptMessage: string): Promise<string | null> {
    try {
      const credentials = await Keychain.getGenericPassword({
        service: key,
        authenticationPrompt: {
          title: promptMessage,
          subtitle: 'ยืนยันตัวตน',
          cancel: 'ยกเลิก',
        },
      });
      
      if (credentials) {
        return credentials.password;
      }
      return null;
    } catch (error) {
      console.error('Failed to get with biometric:', error);
      return null;
    }
  }

  // ลบข้อมูล
  static async delete(key: string): Promise<boolean> {
    try {
      await Keychain.resetGenericPassword({ service: key });
      return true;
    } catch (error) {
      return false;
    }
  }
}

// ตัวอย่างการใช้งาน: บันทึก API key อย่างปลอดภัย
const saveApiKeySecurely = async (apiKey: string) => {
  const saved = await SecureStorageService.saveWithBiometric(
    'api_key',
    apiKey
  );
  
  if (saved) {
    Alert.alert('สำเร็จ', 'บันทึก API Key อย่างปลอดภัยแล้ว');
  }
};

const getApiKey = async () => {
  const apiKey = await SecureStorageService.getWithBiometric(
    'api_key',
    'ยืนยันตัวตนเพื่อเข้าถึง API Key'
  );
  
  if (apiKey) {
    Alert.alert('API Key', apiKey);
  }
};
```

---

## Workshop: Biometric Login App

```typescript
import React, { useState, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Alert,
  Switch,
  ScrollView,
  Vibration,
} from 'react-native';
import ReactNativeBiometrics from 'react-native-biometrics';
import AsyncStorage from '@react-native-async-storage/async-storage';

const rnBiometrics = new ReactNativeBiometrics();

interface AppSettings {
  biometricLogin: boolean;
  biometricForPayment: boolean;
  biometricForSensitiveData: boolean;
  autoLockTimeout: number; // minutes
}

const BiometricSettingsWorkshop: React.FC = () => {
  const [settings, setSettings] = useState<AppSettings>({
    biometricLogin: false,
    biometricForPayment: true,
    biometricForSensitiveData: true,
    autoLockTimeout: 5,
  });
  const [biometricType, setBiometricType] = useState<string>('');
  const [isAvailable, setIsAvailable] = useState(false);
  const [lastAuthTime, setLastAuthTime] = useState<Date | null>(null);

  useEffect(() => {
    checkBiometricAvailability();
    loadSettings();
  }, []);

  const checkBiometricAvailability = async () => {
    const { available, biometryType } = await rnBiometrics.isSensorAvailable();
    setIsAvailable(available);
    
    if (available) {
      const typeMap: Record<string, string> = {
        FaceID: 'Face ID',
        TouchID: 'Touch ID',
        Biometrics: 'ลายนิ้วมือ',
      };
      setBiometricType(typeMap[biometryType || ''] || 'Biometric');
    }
  };

  const loadSettings = async () => {
    try {
      const stored = await AsyncStorage.getItem('@app_settings');
      if (stored) setSettings(JSON.parse(stored));
    } catch {}
  };

  const saveSetting = async (key: keyof AppSettings, value: boolean | number) => {
    const newSettings = { ...settings, [key]: value };
    setSettings(newSettings);
    await AsyncStorage.setItem('@app_settings', JSON.stringify(newSettings));
  };

  const toggleBiometricSetting = async (
    key: keyof AppSettings,
    currentValue: boolean
  ) => {
    if (!currentValue) {
      // กำลังเปิด - ต้องยืนยัน biometric ก่อน
      const { success } = await rnBiometrics.simplePrompt({
        promptMessage: `ยืนยัน ${biometricType} เพื่อเปิดใช้งาน`,
        cancelButtonText: 'ยกเลิก',
      });
      
      if (success) {
        Vibration.vibrate(50);
        saveSetting(key, true);
        setLastAuthTime(new Date());
      }
    } else {
      // ปิด - ต้องยืนยันก่อนเช่นกัน
      const { success } = await rnBiometrics.simplePrompt({
        promptMessage: `ยืนยัน ${biometricType} เพื่อปิดการใช้งาน`,
        cancelButtonText: 'ยกเลิก',
      });
      
      if (success) {
        saveSetting(key, false);
      }
    }
  };

  const testBiometric = async () => {
    const { success, error } = await rnBiometrics.simplePrompt({
      promptMessage: `ทดสอบ ${biometricType}`,
      cancelButtonText: 'ยกเลิก',
    });
    
    if (success) {
      Vibration.vibrate([50, 50, 50]);
      setLastAuthTime(new Date());
      Alert.alert('สำเร็จ!', `${biometricType} ทำงานได้ปกติ ✅`);
    } else {
      Alert.alert('ล้มเหลว', error || 'ไม่สามารถยืนยันตัวตนได้');
    }
  };

  const timeoutOptions = [1, 5, 15, 30, 60];

  return (
    <ScrollView style={wsStyles.container}>
      <Text style={wsStyles.title}>การตั้งค่า Biometric</Text>

      {!isAvailable ? (
        <View style={wsStyles.unavailableCard}>
          <Text style={wsStyles.unavailableText}>
            ⚠️ Device นี้ไม่รองรับ Biometric Authentication
          </Text>
        </View>
      ) : (
        <>
          <View style={wsStyles.infoCard}>
            <Text style={wsStyles.infoTitle}>🔐 {biometricType}</Text>
            <Text style={wsStyles.infoDesc}>
              พร้อมใช้งาน
            </Text>
            {lastAuthTime && (
              <Text style={wsStyles.lastAuth}>
                ยืนยันล่าสุด: {lastAuthTime.toLocaleTimeString('th-TH')}
              </Text>
            )}
            <TouchableOpacity style={wsStyles.testButton} onPress={testBiometric}>
              <Text style={wsStyles.testButtonText}>ทดสอบ {biometricType}</Text>
            </TouchableOpacity>
          </View>

          <View style={wsStyles.settingsSection}>
            <Text style={wsStyles.sectionTitle}>การตั้งค่า</Text>

            <SettingRow
              label={`Login ด้วย ${biometricType}`}
              description="ใช้ biometric แทนรหัสผ่านตอน login"
              value={settings.biometricLogin}
              onToggle={() => toggleBiometricSetting('biometricLogin', settings.biometricLogin)}
            />

            <SettingRow
              label="ยืนยันการชำระเงิน"
              description="ต้องยืนยัน biometric ก่อนทำธุรกรรม"
              value={settings.biometricForPayment}
              onToggle={() => toggleBiometricSetting('biometricForPayment', settings.biometricForPayment)}
            />

            <SettingRow
              label="เข้าถึงข้อมูลสำคัญ"
              description="ยืนยัน biometric ก่อนดูข้อมูล sensitive"
              value={settings.biometricForSensitiveData}
              onToggle={() => toggleBiometricSetting('biometricForSensitiveData', settings.biometricForSensitiveData)}
            />
          </View>

          <View style={wsStyles.timeoutSection}>
            <Text style={wsStyles.sectionTitle}>Auto-Lock หลังไม่ใช้งาน</Text>
            <View style={wsStyles.timeoutGrid}>
              {timeoutOptions.map(minutes => (
                <TouchableOpacity
                  key={minutes}
                  style={[
                    wsStyles.timeoutOption,
                    settings.autoLockTimeout === minutes && wsStyles.timeoutSelected,
                  ]}
                  onPress={() => saveSetting('autoLockTimeout', minutes)}
                >
                  <Text style={[
                    wsStyles.timeoutText,
                    settings.autoLockTimeout === minutes && wsStyles.timeoutTextSelected,
                  ]}>
                    {minutes} {minutes === 60 ? 'ชม.' : 'นาที'}
                  </Text>
                </TouchableOpacity>
              ))}
            </View>
          </View>
        </>
      )}
    </ScrollView>
  );
};

// Helper component
const SettingRow: React.FC<{
  label: string;
  description: string;
  value: boolean;
  onToggle: () => void;
}> = ({ label, description, value, onToggle }) => (
  <View style={wsStyles.settingRow}>
    <View style={wsStyles.settingInfo}>
      <Text style={wsStyles.settingLabel}>{label}</Text>
      <Text style={wsStyles.settingDesc}>{description}</Text>
    </View>
    <Switch
      value={value}
      onValueChange={onToggle}
      trackColor={{ false: '#ddd', true: '#6200EE' }}
      thumbColor={value ? '#fff' : '#f4f3f4'}
    />
  </View>
);

const wsStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  title: { fontSize: 24, fontWeight: 'bold', padding: 20, paddingBottom: 15, color: '#1a1a2e' },
  unavailableCard: {
    margin: 20,
    backgroundColor: '#FFF3E0',
    padding: 20,
    borderRadius: 12,
    borderLeftWidth: 4,
    borderLeftColor: '#FF9800',
  },
  unavailableText: { color: '#E65100', fontSize: 15 },
  infoCard: {
    margin: 20,
    marginTop: 0,
    backgroundColor: '#EDE7F6',
    borderRadius: 16,
    padding: 20,
    borderLeftWidth: 4,
    borderLeftColor: '#6200EE',
  },
  infoTitle: { fontSize: 20, fontWeight: 'bold', color: '#4527A0', marginBottom: 5 },
  infoDesc: { color: '#7B1FA2', fontSize: 14 },
  lastAuth: { color: '#9E9E9E', fontSize: 12, marginTop: 8 },
  testButton: {
    backgroundColor: '#6200EE',
    padding: 10,
    borderRadius: 8,
    marginTop: 12,
    alignItems: 'center',
  },
  testButtonText: { color: 'white', fontWeight: 'bold' },
  settingsSection: {
    backgroundColor: 'white',
    marginHorizontal: 20,
    borderRadius: 12,
    overflow: 'hidden',
    elevation: 2,
    marginBottom: 15,
  },
  sectionTitle: { fontSize: 16, fontWeight: 'bold', color: '#444', padding: 15, paddingBottom: 5 },
  settingRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 15,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  settingInfo: { flex: 1, marginRight: 15 },
  settingLabel: { fontSize: 15, fontWeight: '500', color: '#333' },
  settingDesc: { fontSize: 12, color: '#888', marginTop: 3 },
  timeoutSection: {
    backgroundColor: 'white',
    marginHorizontal: 20,
    borderRadius: 12,
    padding: 15,
    marginBottom: 20,
    elevation: 2,
  },
  timeoutGrid: { flexDirection: 'row', flexWrap: 'wrap', gap: 10, marginTop: 10 },
  timeoutOption: {
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    borderWidth: 2,
    borderColor: '#E0E0E0',
  },
  timeoutSelected: { borderColor: '#6200EE', backgroundColor: '#EDE7F6' },
  timeoutText: { fontSize: 14, color: '#666' },
  timeoutTextSelected: { color: '#6200EE', fontWeight: 'bold' },
});

export default BiometricSettingsWorkshop;
```

---

## Tips และ Best Practices

### 1. Fallback Authentication
```typescript
const authenticateWithFallback = async () => {
  const { success } = await rnBiometrics.simplePrompt({
    promptMessage: 'ยืนยันตัวตน',
    fallbackPromptMessage: 'ใช้รหัส PIN แทน', // iOS
    cancelButtonText: 'ยกเลิก',
    // allowDeviceCredentials: true จะอนุญาต PIN/Pattern/Password
  });
};
```

### 2. ตรวจสอบก่อนแสดง UI
```typescript
const [canUseBiometric, setCanUseBiometric] = useState(false);

useEffect(() => {
  rnBiometrics.isSensorAvailable().then(({ available }) => {
    setCanUseBiometric(available);
  });
}, []);

// แสดงปุ่ม biometric เฉพาะเมื่อรองรับ
{canUseBiometric && <BiometricButton />}
```

### 3. Handle Edge Cases
```typescript
const handleBiometricError = (errorCode: string) => {
  switch (errorCode) {
    case 'AuthenticationFailed':
      Alert.alert('ล้มเหลว', 'ไม่สามารถยืนยันตัวตนได้');
      break;
    case 'UserCancel':
      // ไม่ต้องทำอะไร
      break;
    case 'UserFallback':
      // นำไปยังหน้า PIN
      navigation.navigate('PinAuth');
      break;
    case 'SystemCancel':
      // แอปถูก interrupt
      break;
    case 'NotEnrolled':
      Alert.alert('ไม่มีข้อมูล', 'กรุณาตั้งค่า biometric ในการตั้งค่าอุปกรณ์ก่อน');
      break;
    case 'NotAvailable':
      Alert.alert('ไม่รองรับ', 'อุปกรณ์นี้ไม่รองรับ biometric');
      break;
  }
};
```

---

## สรุป

Biometric Authentication ช่วยให้แอปมีความปลอดภัยและ UX ที่ดีขึ้น:
- ใช้ **react-native-biometrics** สำหรับ biometric ทั้งสองแพลตฟอร์ม
- เสมอมี **fallback** เผื่อ biometric ล้มเหลว
- ข้อมูล biometric เก็บใน **Secure Enclave** ปลอดภัยมาก
- ทำ **server-side verification** สำหรับ critical operations
- แจ้งผู้ใช้ชัดเจนว่า biometric ใช้ทำอะไร
