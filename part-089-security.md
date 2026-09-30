# Part 089: Advanced Security ใน React Native

## บทนำ

ความปลอดภัยของ Mobile App เป็นสิ่งสำคัญอย่างยิ่ง โดยเฉพาะ app ที่จัดการข้อมูลส่วนตัวหรือการเงิน เราจะเรียนรู้เทคนิค security ขั้นสูงสำหรับ React Native

## หัวข้อที่จะเรียน

1. SSL Certificate Pinning
2. Jailbreak/Root Detection
3. Code Obfuscation
4. Secure Storage
5. Anti-Reverse Engineering
6. Workshop: Security Hardening

---

## 1. SSL Certificate Pinning

SSL Pinning ป้องกัน Man-in-the-Middle attacks โดยตรวจสอบว่า certificate ของ server ตรงกับที่ app เก็บไว้

### การติดตั้ง

```bash
npm install react-native-ssl-pinning
```

### utils/secureHttp.ts

```typescript
import { fetch as sslFetch } from 'react-native-ssl-pinning';

const SSL_CERTIFICATES = {
  production: 'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=', // hash จาก cert
  staging: 'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=',
};

export const createSecureClient = (baseURL: string) => {
  const makeRequest = async (
    path: string,
    options: {
      method: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
      body?: object;
      headers?: Record<string, string>;
    }
  ) => {
    try {
      const response = await sslFetch(`${baseURL}${path}`, {
        method: options.method,
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
        },
        body: options.body ? JSON.stringify(options.body) : undefined,
        sslPinning: {
          certs: ['cert_production', 'cert_backup'], // ชื่อไฟล์ cert
        },
        timeoutInterval: 30000,
      });

      const data = await response.json();
      return { data, status: response.status };
    } catch (error: any) {
      if (error.message?.includes('SSL')) {
        // Log security incident
        await logSecurityIncident('ssl_pinning_failed', {
          url: path,
          error: error.message,
        });
        throw new Error('การเชื่อมต่อไม่ปลอดภัย กรุณาตรวจสอบ network');
      }
      throw error;
    }
  };

  return {
    get: (path: string, headers?: Record<string, string>) =>
      makeRequest(path, { method: 'GET', headers }),
    post: (path: string, body: object, headers?: Record<string, string>) =>
      makeRequest(path, { method: 'POST', body, headers }),
    put: (path: string, body: object, headers?: Record<string, string>) =>
      makeRequest(path, { method: 'PUT', body, headers }),
    delete: (path: string, headers?: Record<string, string>) =>
      makeRequest(path, { method: 'DELETE', headers }),
  };
};

const logSecurityIncident = async (type: string, details: object) => {
  // ส่งไปยัง security monitoring service
  console.error(`[SECURITY INCIDENT] ${type}:`, details);
};
```

### การเพิ่ม Certificate ใน iOS

```bash
# ดึง certificate จาก server
openssl s_client -connect api.yourapp.com:443 </dev/null | \
  openssl x509 -outform DER > certificate.der

# คำนวณ hash
openssl dgst -sha256 -binary certificate.der | openssl base64
```

เพิ่มไฟล์ `.cer` ใน Xcode project:
1. ลาก `.cer` file ไปยัง Xcode project
2. เลือก "Add to target"

### การเพิ่ม Certificate ใน Android

```bash
# คัดลอก certificate
cp certificate.der android/app/src/main/res/raw/cert_production.cer
```

---

## 2. Jailbreak/Root Detection

### การติดตั้ง

```bash
npm install jail-monkey
```

### hooks/useSecurityCheck.ts

```typescript
import { useState, useEffect } from 'react';
import JailMonkey from 'jail-monkey';
import { Platform, Alert } from 'react-native';
import DeviceInfo from 'react-native-device-info';

interface SecurityStatus {
  isSecure: boolean;
  issues: SecurityIssue[];
  riskLevel: 'low' | 'medium' | 'high' | 'critical';
}

interface SecurityIssue {
  type: string;
  severity: 'low' | 'medium' | 'high' | 'critical';
  description: string;
}

export const useSecurityCheck = () => {
  const [securityStatus, setSecurityStatus] = useState<SecurityStatus>({
    isSecure: true,
    issues: [],
    riskLevel: 'low',
  });
  const [checking, setChecking] = useState(true);

  useEffect(() => {
    performSecurityCheck();
  }, []);

  const performSecurityCheck = async () => {
    setChecking(true);
    const issues: SecurityIssue[] = [];

    try {
      // ตรวจสอบ Jailbreak/Root
      if (JailMonkey.isJailBroken()) {
        issues.push({
          type: 'jailbreak',
          severity: 'critical',
          description: 'อุปกรณ์นี้ถูก Jailbreak/Root แล้ว',
        });
      }

      // ตรวจสอบ Debug Mode
      if (JailMonkey.hookDetected()) {
        issues.push({
          type: 'hook_detected',
          severity: 'critical',
          description: 'ตรวจพบการ Hook runtime',
        });
      }

      // ตรวจสอบ Emulator
      if (JailMonkey.isOnExternalStorage()) {
        issues.push({
          type: 'external_storage',
          severity: 'medium',
          description: 'app ทำงานจาก external storage',
        });
      }

      // ตรวจสอบว่าเป็น Emulator
      const isEmulator = await DeviceInfo.isEmulator();
      if (isEmulator && __DEV__ === false) {
        issues.push({
          type: 'emulator',
          severity: 'high',
          description: 'app กำลังทำงานบน emulator',
        });
      }

      // ตรวจสอบ Developer Mode (Android)
      if (Platform.OS === 'android') {
        const inDebugBuild = await DeviceInfo.isDebugBuild();
        if (!__DEV__ && inDebugBuild) {
          issues.push({
            type: 'debug_build',
            severity: 'high',
            description: 'Debug build ใน production',
          });
        }
      }

      // ตรวจสอบ ADB (Android Debug Bridge)
      if (Platform.OS === 'android' && JailMonkey.AdbEnabled()) {
        issues.push({
          type: 'adb_enabled',
          severity: 'medium',
          description: 'ADB เปิดใช้งาน',
        });
      }

      // คำนวณ risk level
      const riskLevel = calculateRiskLevel(issues);
      const isSecure = !issues.some(i => i.severity === 'critical');

      setSecurityStatus({ isSecure, issues, riskLevel });

      // ถ้ามีความเสี่ยงสูง แสดง alert
      if (issues.some(i => i.severity === 'critical')) {
        showSecurityWarning(issues);
      }

    } catch (error) {
      console.error('Security check error:', error);
    } finally {
      setChecking(false);
    }
  };

  const calculateRiskLevel = (issues: SecurityIssue[]): 'low' | 'medium' | 'high' | 'critical' => {
    if (issues.some(i => i.severity === 'critical')) return 'critical';
    if (issues.some(i => i.severity === 'high')) return 'high';
    if (issues.some(i => i.severity === 'medium')) return 'medium';
    return 'low';
  };

  const showSecurityWarning = (issues: SecurityIssue[]) => {
    const criticalIssues = issues.filter(i => i.severity === 'critical');
    
    Alert.alert(
      '⚠️ ตรวจพบความเสี่ยงด้านความปลอดภัย',
      `พบปัญหา ${criticalIssues.length} รายการที่อาจทำให้ข้อมูลของคุณไม่ปลอดภัย:\n\n` +
        criticalIssues.map(i => `• ${i.description}`).join('\n'),
      [
        {
          text: 'ออกจากแอป',
          style: 'destructive',
          // ใน production ควรปิด app
        },
        { text: 'ยังคงใช้งาน', style: 'cancel' },
      ]
    );
  };

  return { securityStatus, checking, performSecurityCheck };
};
```

---

## 3. Secure Storage

### การติดตั้ง

```bash
npm install react-native-keychain
npm install @react-native-async-storage/async-storage
```

### utils/secureStorage.ts

```typescript
import * as Keychain from 'react-native-keychain';
import CryptoJS from 'crypto-js';
import AsyncStorage from '@react-native-async-storage/async-storage';

const ENCRYPTION_KEY = process.env.STORAGE_ENCRYPTION_KEY || 'default-key';

class SecureStorage {
  private static instance: SecureStorage;

  static getInstance(): SecureStorage {
    if (!SecureStorage.instance) {
      SecureStorage.instance = new SecureStorage();
    }
    return SecureStorage.instance;
  }

  // เก็บ credentials ที่ sensitive (passwords, tokens)
  async storeCredentials(
    service: string,
    username: string,
    password: string
  ): Promise<boolean> {
    try {
      await Keychain.setInternetCredentials(service, username, password, {
        accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_ANY_OR_DEVICE_PASSCODE,
        accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
      });
      return true;
    } catch (error) {
      console.error('Keychain store error:', error);
      return false;
    }
  }

  // ดึง credentials
  async getCredentials(service: string): Promise<{
    username: string;
    password: string;
  } | null> {
    try {
      const credentials = await Keychain.getInternetCredentials(service);
      if (credentials) {
        return {
          username: credentials.username,
          password: credentials.password,
        };
      }
      return null;
    } catch (error) {
      console.error('Keychain get error:', error);
      return null;
    }
  }

  // ลบ credentials
  async removeCredentials(service: string): Promise<boolean> => {
    try {
      await Keychain.resetInternetCredentials(service);
      return true;
    } catch {
      return false;
    }
  }

  // เก็บ generic data (encrypted)
  async setItem(key: string, value: any): Promise<void> {
    try {
      const jsonValue = JSON.stringify(value);
      const encrypted = CryptoJS.AES.encrypt(jsonValue, ENCRYPTION_KEY).toString();
      await AsyncStorage.setItem(`@secure_${key}`, encrypted);
    } catch (error) {
      console.error('SecureStorage set error:', error);
      throw error;
    }
  }

  // อ่าน generic data (decrypted)
  async getItem<T>(key: string): Promise<T | null> {
    try {
      const encrypted = await AsyncStorage.getItem(`@secure_${key}`);
      if (!encrypted) return null;

      const bytes = CryptoJS.AES.decrypt(encrypted, ENCRYPTION_KEY);
      const decrypted = bytes.toString(CryptoJS.enc.Utf8);
      return JSON.parse(decrypted);
    } catch (error) {
      console.error('SecureStorage get error:', error);
      return null;
    }
  }

  // ลบ item
  async removeItem(key: string): Promise<void> {
    await AsyncStorage.removeItem(`@secure_${key}`);
  }

  // เก็บ biometric token
  async storeBiometricToken(token: string): Promise<boolean> {
    try {
      const supported = await Keychain.getSupportedBiometryType();
      if (!supported) {
        console.log('Biometry ไม่รองรับ');
        return false;
      }

      await Keychain.setGenericPassword('biometric_user', token, {
        service: 'biometric_auth',
        accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET,
        accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
      });
      return true;
    } catch (error) {
      console.error('Biometric store error:', error);
      return false;
    }
  }

  // ตรวจสอบ biometric
  async verifyBiometric(): Promise<string | null> {
    try {
      const credentials = await Keychain.getGenericPassword({
        service: 'biometric_auth',
        authenticationPrompt: {
          title: 'ยืนยันตัวตน',
          subtitle: 'ใช้ Face ID หรือลายนิ้วมือ',
          cancel: 'ยกเลิก',
        },
      });

      if (credentials) {
        return credentials.password; // token
      }
      return null;
    } catch (error) {
      console.error('Biometric verify error:', error);
      return null;
    }
  }
}

export default SecureStorage;
```

---

## 4. Anti-Reverse Engineering

### utils/antiTampering.ts

```typescript
import { NativeModules, Platform } from 'react-native';
import DeviceInfo from 'react-native-device-info';
import CryptoJS from 'crypto-js';

class AntiTampering {
  // ตรวจสอบ app signature
  static async verifyAppSignature(): Promise<boolean> {
    if (Platform.OS === 'android') {
      try {
        // ดึง signing certificate fingerprint
        const { signatures } = NativeModules.AppSignatureModule;
        const expectedFingerprint = 'SHA256:EXPECTED_FINGERPRINT_HERE';
        return signatures.includes(expectedFingerprint);
      } catch {
        return false;
      }
    }
    return true; // iOS ตรวจสอบผ่าน App Store signing
  }

  // ตรวจสอบ app integrity
  static async checkIntegrity(): Promise<boolean> {
    const bundleId = await DeviceInfo.getBundleId();
    const version = await DeviceInfo.getVersion();
    const buildNumber = await DeviceInfo.getBuildNumber();

    // ตรวจสอบว่า app ถูกแก้ไขหรือไม่
    const expectedBundleId = 'com.yourcompany.yourapp';
    const expectedVersion = '1.0.0';

    if (bundleId !== expectedBundleId) {
      console.error('Bundle ID ไม่ถูกต้อง');
      return false;
    }

    return true;
  }

  // สร้าง request signature
  static signRequest(body: object, secretKey: string): string {
    const timestamp = Date.now().toString();
    const bodyString = JSON.stringify(body);
    const message = `${timestamp}.${bodyString}`;
    const signature = CryptoJS.HmacSHA256(message, secretKey).toString();
    return `t=${timestamp},v1=${signature}`;
  }

  // ตรวจสอบ request signature (server-side logic)
  static verifyRequestSignature(
    signature: string,
    body: object,
    secretKey: string,
    maxAgeSeconds: number = 300
  ): boolean {
    const parts = signature.split(',');
    const timestampPart = parts.find(p => p.startsWith('t='));
    const sigPart = parts.find(p => p.startsWith('v1='));

    if (!timestampPart || !sigPart) return false;

    const timestamp = parseInt(timestampPart.split('=')[1]);
    const providedSig = sigPart.split('=')[1];

    // ตรวจสอบ timestamp ไม่เกิน 5 นาที
    if (Date.now() - timestamp > maxAgeSeconds * 1000) {
      return false;
    }

    const expectedSig = CryptoJS.HmacSHA256(
      `${timestamp}.${JSON.stringify(body)}`,
      secretKey
    ).toString();

    return expectedSig === providedSig;
  }

  // Obfuscate sensitive data สำหรับ logging
  static maskSensitiveData(data: any): any {
    if (typeof data !== 'object') return data;
    
    const sensitiveFields = ['password', 'token', 'credit_card', 'cvv', 'pin', 'secret'];
    const masked = { ...data };
    
    for (const key of Object.keys(masked)) {
      if (sensitiveFields.some(f => key.toLowerCase().includes(f))) {
        masked[key] = '***MASKED***';
      } else if (typeof masked[key] === 'object') {
        masked[key] = AntiTampering.maskSensitiveData(masked[key]);
      }
    }
    
    return masked;
  }
}

export default AntiTampering;
```

---

## 5. Code Obfuscation

### Metro Bundler Obfuscation

```javascript
// metro.config.js
const { getDefaultConfig } = require('@react-native/metro-config');

module.exports = (async () => {
  const config = await getDefaultConfig(__dirname);
  
  if (process.env.NODE_ENV === 'production') {
    config.transformer = {
      ...config.transformer,
      minifierConfig: {
        // Advanced minification
        keep_fnames: false,
        keep_classnames: false,
        mangle: {
          toplevel: true,
          eval: true,
        },
        compress: {
          drop_console: true,
          drop_debugger: true,
          pure_funcs: ['console.log', 'console.info', 'console.warn'],
          passes: 2,
        },
      },
    };
  }
  
  return config;
})();
```

### ProGuard (Android)

```
# android/app/proguard-rules.pro

# React Native
-keep class com.facebook.react.** { *; }
-keep class com.facebook.hermes.** { *; }

# ซ่อนชื่อ class
-renamesourcefileattribute SourceFile
-keepattributes SourceFile,LineNumberTable

# Remove logging
-assumenosideeffects class android.util.Log {
    public static boolean isLoggable(java.lang.String, int);
    public static int v(...);
    public static int i(...);
    public static int w(...);
    public static int d(...);
    public static int e(...);
}

# Encrypt strings (ต้องใช้ plugin)
-obfuscatedstring

# ป้องกัน decompile
-dontskipnonpubliclibraryclassmembers
```

---

## 6. Workshop: Security Hardening

### SecurityProvider.tsx

```typescript
import React, { createContext, useContext, useEffect, useState, ReactNode } from 'react';
import { Alert, AppState } from 'react-native';
import { useSecurityCheck } from '../hooks/useSecurityCheck';
import AntiTampering from '../utils/antiTampering';

interface SecurityContextType {
  isSecure: boolean;
  riskLevel: string;
  performCheck: () => void;
}

const SecurityContext = createContext<SecurityContextType>({
  isSecure: true,
  riskLevel: 'low',
  performCheck: () => {},
});

export const SecurityProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const { securityStatus, checking, performSecurityCheck } = useSecurityCheck();
  const [appState, setAppState] = useState(AppState.currentState);

  useEffect(() => {
    // ตรวจสอบเมื่อ app กลับมา foreground
    const subscription = AppState.addEventListener('change', nextAppState => {
      if (appState.match(/inactive|background/) && nextAppState === 'active') {
        performSecurityCheck();
      }
      setAppState(nextAppState);
    });

    return () => subscription.remove();
  }, [appState]);

  useEffect(() => {
    // ตรวจสอบ app integrity เมื่อ start
    const checkIntegrity = async () => {
      const isValid = await AntiTampering.checkIntegrity();
      if (!isValid) {
        Alert.alert(
          'แอปถูกแก้ไข',
          'กรุณาดาวน์โหลดแอปจาก App Store/Play Store เท่านั้น'
        );
      }
    };
    checkIntegrity();
  }, []);

  return (
    <SecurityContext.Provider
      value={{
        isSecure: securityStatus.isSecure,
        riskLevel: securityStatus.riskLevel,
        performCheck: performSecurityCheck,
      }}
    >
      {children}
    </SecurityContext.Provider>
  );
};

export const useSecurity = () => useContext(SecurityContext);
```

### screens/SecurityDashboard.tsx

```typescript
import React from 'react';
import { View, Text, StyleSheet, ScrollView, TouchableOpacity } from 'react-native';
import { useSecurityCheck } from '../hooks/useSecurityCheck';

const SecurityDashboard: React.FC = () => {
  const { securityStatus, checking, performSecurityCheck } = useSecurityCheck();

  const getRiskColor = (level: string) => {
    switch (level) {
      case 'critical': return '#f44336';
      case 'high': return '#ff5722';
      case 'medium': return '#FF9800';
      default: return '#4CAF50';
    }
  };

  const getSeverityIcon = (severity: string) => {
    switch (severity) {
      case 'critical': return '🔴';
      case 'high': return '🟠';
      case 'medium': return '🟡';
      default: return '🟢';
    }
  };

  return (
    <ScrollView style={styles.container}>
      <View style={[
        styles.statusCard,
        { backgroundColor: getRiskColor(securityStatus.riskLevel) }
      ]}>
        <Text style={styles.statusIcon}>
          {securityStatus.isSecure ? '🛡️' : '⚠️'}
        </Text>
        <Text style={styles.statusTitle}>
          {securityStatus.isSecure ? 'อุปกรณ์ปลอดภัย' : 'พบความเสี่ยง'}
        </Text>
        <Text style={styles.statusLevel}>
          Risk Level: {securityStatus.riskLevel.toUpperCase()}
        </Text>
      </View>

      {securityStatus.issues.length > 0 ? (
        <View style={styles.issuesCard}>
          <Text style={styles.issuesTitle}>
            ปัญหาที่พบ ({securityStatus.issues.length})
          </Text>
          {securityStatus.issues.map((issue, index) => (
            <View key={index} style={styles.issueItem}>
              <Text style={styles.issueIcon}>
                {getSeverityIcon(issue.severity)}
              </Text>
              <View style={styles.issueInfo}>
                <Text style={styles.issueType}>{issue.type}</Text>
                <Text style={styles.issueDescription}>{issue.description}</Text>
              </View>
              <View style={[
                styles.severityBadge,
                { backgroundColor: getRiskColor(issue.severity) }
              ]}>
                <Text style={styles.severityText}>{issue.severity}</Text>
              </View>
            </View>
          ))}
        </View>
      ) : (
        <View style={styles.allClearCard}>
          <Text style={styles.allClearText}>✅ ไม่พบปัญหาด้านความปลอดภัย</Text>
        </View>
      )}

      <TouchableOpacity
        style={styles.checkButton}
        onPress={performSecurityCheck}
        disabled={checking}
      >
        <Text style={styles.checkButtonText}>
          {checking ? 'กำลังตรวจสอบ...' : 'ตรวจสอบอีกครั้ง'}
        </Text>
      </TouchableOpacity>

      {/* Security Tips */}
      <View style={styles.tipsCard}>
        <Text style={styles.tipsTitle}>คำแนะนำด้านความปลอดภัย</Text>
        {[
          'อย่า Jailbreak/Root อุปกรณ์',
          'ติดตั้งแอปจาก official store เท่านั้น',
          'อัปเดต OS ให้เป็นเวอร์ชันล่าสุด',
          'ใช้ VPN เมื่อเชื่อมต่อ public WiFi',
          'เปิดใช้งาน 2FA เสมอ',
        ].map((tip, index) => (
          <Text key={index} style={styles.tipItem}>• {tip}</Text>
        ))}
      </View>
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5', padding: 16 },
  statusCard: {
    borderRadius: 16, padding: 24, alignItems: 'center', marginBottom: 16,
  },
  statusIcon: { fontSize: 48, marginBottom: 8 },
  statusTitle: { fontSize: 24, fontWeight: 'bold', color: 'white' },
  statusLevel: { fontSize: 14, color: 'rgba(255,255,255,0.8)', marginTop: 4 },
  issuesCard: {
    backgroundColor: 'white', borderRadius: 12, padding: 16, marginBottom: 16,
    shadowColor: '#000', shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1, shadowRadius: 4, elevation: 2,
  },
  issuesTitle: { fontSize: 16, fontWeight: '600', color: '#333', marginBottom: 12 },
  issueItem: {
    flexDirection: 'row', alignItems: 'center', paddingVertical: 8,
    borderBottomWidth: 1, borderBottomColor: '#f0f0f0',
  },
  issueIcon: { fontSize: 20, marginRight: 8 },
  issueInfo: { flex: 1 },
  issueType: { fontSize: 13, fontWeight: '600', color: '#333', textTransform: 'capitalize' },
  issueDescription: { fontSize: 12, color: '#666', marginTop: 2 },
  severityBadge: { paddingHorizontal: 8, paddingVertical: 3, borderRadius: 12 },
  severityText: { color: 'white', fontSize: 11, fontWeight: '600' },
  allClearCard: {
    backgroundColor: '#e8f5e9', borderRadius: 12, padding: 16,
    marginBottom: 16, alignItems: 'center',
  },
  allClearText: { color: '#2e7d32', fontSize: 16 },
  checkButton: {
    backgroundColor: '#1565C0', padding: 16, borderRadius: 12,
    alignItems: 'center', marginBottom: 16,
  },
  checkButtonText: { color: 'white', fontSize: 16, fontWeight: '600' },
  tipsCard: {
    backgroundColor: 'white', borderRadius: 12, padding: 16, marginBottom: 32,
    shadowColor: '#000', shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1, shadowRadius: 4, elevation: 2,
  },
  tipsTitle: { fontSize: 16, fontWeight: '600', color: '#333', marginBottom: 8 },
  tipItem: { fontSize: 14, color: '#555', paddingVertical: 4, lineHeight: 20 },
});

export default SecurityDashboard;
```

---

## Best Practices

### Security Checklist

```typescript
// security-checklist.ts
export const SECURITY_CHECKLIST = [
  {
    category: 'Network',
    items: [
      '✅ SSL/TLS 1.3 หรือสูงกว่า',
      '✅ Certificate Pinning',
      '✅ Request Signing',
      '✅ Rate Limiting',
    ],
  },
  {
    category: 'Storage',
    items: [
      '✅ Keychain/Keystore สำหรับ sensitive data',
      '✅ ไม่เก็บ password ใน plain text',
      '✅ Encrypt local database',
      '✅ Clear data เมื่อ logout',
    ],
  },
  {
    category: 'Code',
    items: [
      '✅ Code obfuscation',
      '✅ ลบ console.log ใน production',
      '✅ ไม่ hardcode secrets',
      '✅ Use environment variables',
    ],
  },
  {
    category: 'Runtime',
    items: [
      '✅ Jailbreak/Root detection',
      '✅ App integrity check',
      '✅ Biometric authentication',
      '✅ Auto logout เมื่อไม่ใช้งาน',
    ],
  },
];
```

---

## Workshop Exercises

1. **Implement Certificate Pinning** ใน existing app
2. **Add Biometric Auth** ก่อนเข้าถึง sensitive screens
3. **Security Logger** บันทึก security events ทั้งหมด
4. **Auto Logout** เมื่อ app อยู่ background เกิน 5 นาที

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **SSL Pinning** - ป้องกัน MITM attacks
2. **Jailbreak Detection** - ตรวจสอบ device security
3. **Secure Storage** - เก็บข้อมูล sensitive อย่างปลอดภัย
4. **Anti-Tampering** - ป้องกันการแก้ไข app
5. **Obfuscation** - ซ่อน business logic

> **หมายเหตุ:** Security เป็น ongoing process ต้องอัปเดตและตรวจสอบสม่ำเสมอ
