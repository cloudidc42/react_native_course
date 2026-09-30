# Part 038: Firebase Authentication

## Firebase คืออะไร?

Firebase เป็น Backend-as-a-Service (BaaS) จาก Google ที่ให้บริการ:
- Authentication (รองรับหลาย providers)
- Firestore Database (NoSQL)
- Realtime Database
- Storage (ไฟล์/รูปภาพ)
- Cloud Functions
- Analytics
- Crashlytics

### ข้อดีของ Firebase Auth

- รองรับ Email, Phone, Google, Facebook, Apple, Twitter, GitHub
- Secure โดย default
- ไม่ต้อง setup backend เอง
- SDK มีให้ทุก platform
- Free tier ใจกว้าง

---

## Setup Firebase

### 1. สร้าง Firebase Project

1. ไปที่ [Firebase Console](https://console.firebase.google.com)
2. คลิก "Add project"
3. ตั้งชื่อ project
4. Enable Google Analytics (optional)

### 2. เพิ่ม App

1. คลิก Android icon และ/หรือ iOS icon
2. ใส่ package name / bundle ID
3. ดาวน์โหลด `google-services.json` (Android) และ `GoogleService-Info.plist` (iOS)

### 3. ติดตั้ง Firebase SDK

```bash
# Expo
npx expo install @react-native-firebase/app @react-native-firebase/auth

# หรือ
npm install @react-native-firebase/app @react-native-firebase/auth

# สำหรับ Google Sign-In
npm install @react-native-google-signin/google-signin
```

### 4. Configuration

```typescript
// firebase.config.ts - ถ้าใช้ Firebase JS SDK (Web)
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';

const firebaseConfig = {
  apiKey: 'YOUR_API_KEY',
  authDomain: 'YOUR_PROJECT.firebaseapp.com',
  projectId: 'YOUR_PROJECT_ID',
  storageBucket: 'YOUR_PROJECT.appspot.com',
  messagingSenderId: 'YOUR_MESSAGING_SENDER_ID',
  appId: 'YOUR_APP_ID',
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
export default app;
```

---

## Email/Password Auth

```typescript
// src/services/firebase/emailAuth.ts
import auth from '@react-native-firebase/auth';

export const emailAuth = {
  // ลงทะเบียน
  register: async (email: string, password: string, displayName: string) => {
    const userCredential = await auth().createUserWithEmailAndPassword(
      email,
      password
    );

    // อัปเดต display name
    await userCredential.user.updateProfile({ displayName });

    // ส่ง verification email
    await userCredential.user.sendEmailVerification();

    return userCredential.user;
  },

  // เข้าสู่ระบบ
  login: async (email: string, password: string) => {
    const userCredential = await auth().signInWithEmailAndPassword(
      email,
      password
    );
    return userCredential.user;
  },

  // ออกจากระบบ
  logout: () => auth().signOut(),

  // ลืมรหัสผ่าน
  forgotPassword: (email: string) =>
    auth().sendPasswordResetEmail(email),

  // เปลี่ยนรหัสผ่าน
  changePassword: async (currentPassword: string, newPassword: string) => {
    const user = auth().currentUser;
    if (!user || !user.email) throw new Error('Not authenticated');

    // Re-authenticate ก่อนเปลี่ยน password
    const credential = auth.EmailAuthProvider.credential(
      user.email,
      currentPassword
    );
    await user.reauthenticateWithCredential(credential);
    await user.updatePassword(newPassword);
  },

  // อัปเดต email
  updateEmail: async (newEmail: string, currentPassword: string) => {
    const user = auth().currentUser;
    if (!user || !user.email) throw new Error('Not authenticated');

    const credential = auth.EmailAuthProvider.credential(
      user.email,
      currentPassword
    );
    await user.reauthenticateWithCredential(credential);
    await user.updateEmail(newEmail);
    await user.sendEmailVerification();
  },

  // ตรวจสอบ email verification
  isEmailVerified: () => auth().currentUser?.emailVerified ?? false,

  // Reload user data
  reloadUser: () => auth().currentUser?.reload(),
};
```

### Email Auth Screen

```typescript
// src/screens/auth/EmailAuthScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  ActivityIndicator,
  KeyboardAvoidingView,
  Platform,
  ScrollView,
} from 'react-native';
import { emailAuth } from '../../services/firebase/emailAuth';
import auth from '@react-native-firebase/auth';

type Mode = 'login' | 'register' | 'forgot';

export const EmailAuthScreen: React.FC = () => {
  const [mode, setMode] = useState<Mode>('login');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [name, setName] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async () => {
    if (!email.trim()) {
      Alert.alert('ผิดพลาด', 'กรุณาใส่ email');
      return;
    }

    setIsLoading(true);

    try {
      if (mode === 'login') {
        await emailAuth.login(email, password);
        Alert.alert('สำเร็จ', 'เข้าสู่ระบบแล้ว');
      } else if (mode === 'register') {
        if (!name.trim()) {
          Alert.alert('ผิดพลาด', 'กรุณาใส่ชื่อ');
          return;
        }
        await emailAuth.register(email, password, name);
        Alert.alert(
          'สำเร็จ',
          'สมัครสมาชิกแล้ว กรุณาตรวจสอบ email เพื่อยืนยัน'
        );
      } else if (mode === 'forgot') {
        await emailAuth.forgotPassword(email);
        Alert.alert('สำเร็จ', 'ส่ง email รีเซ็ตรหัสผ่านแล้ว');
        setMode('login');
      }
    } catch (error: any) {
      let message = 'เกิดข้อผิดพลาด';

      // Firebase error codes
      switch (error.code) {
        case 'auth/user-not-found':
          message = 'ไม่พบ email นี้ในระบบ';
          break;
        case 'auth/wrong-password':
          message = 'รหัสผ่านไม่ถูกต้อง';
          break;
        case 'auth/email-already-in-use':
          message = 'email นี้ถูกใช้งานแล้ว';
          break;
        case 'auth/weak-password':
          message = 'รหัสผ่านไม่แข็งแรงพอ';
          break;
        case 'auth/invalid-email':
          message = 'รูปแบบ email ไม่ถูกต้อง';
          break;
        case 'auth/too-many-requests':
          message = 'มีการพยายามเข้าสู่ระบบมากเกินไป กรุณารอสักครู่';
          break;
        default:
          message = error.message || 'เกิดข้อผิดพลาด';
      }

      Alert.alert('ผิดพลาด', message);
    } finally {
      setIsLoading(false);
    }
  };

  const titles = {
    login: 'เข้าสู่ระบบ',
    register: 'สมัครสมาชิก',
    forgot: 'ลืมรหัสผ่าน',
  };

  const buttonLabels = {
    login: 'เข้าสู่ระบบ',
    register: 'สมัครสมาชิก',
    forgot: 'ส่ง email รีเซ็ต',
  };

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
    >
      <ScrollView contentContainerStyle={styles.content}>
        <Text style={styles.title}>{titles[mode]}</Text>

        {mode === 'register' && (
          <View style={styles.inputGroup}>
            <Text style={styles.label}>ชื่อ</Text>
            <TextInput
              value={name}
              onChangeText={setName}
              placeholder="ชื่อ-นามสกุล"
              style={styles.input}
              autoCapitalize="words"
            />
          </View>
        )}

        <View style={styles.inputGroup}>
          <Text style={styles.label}>อีเมล</Text>
          <TextInput
            value={email}
            onChangeText={setEmail}
            placeholder="example@email.com"
            keyboardType="email-address"
            autoCapitalize="none"
            style={styles.input}
          />
        </View>

        {mode !== 'forgot' && (
          <View style={styles.inputGroup}>
            <Text style={styles.label}>รหัสผ่าน</Text>
            <TextInput
              value={password}
              onChangeText={setPassword}
              placeholder="รหัสผ่าน"
              secureTextEntry
              style={styles.input}
            />
          </View>
        )}

        <TouchableOpacity
          onPress={handleSubmit}
          style={[styles.submitButton, isLoading && styles.disabled]}
          disabled={isLoading}
        >
          {isLoading ? (
            <ActivityIndicator color="#fff" />
          ) : (
            <Text style={styles.submitText}>{buttonLabels[mode]}</Text>
          )}
        </TouchableOpacity>

        <View style={styles.links}>
          {mode === 'login' && (
            <>
              <TouchableOpacity onPress={() => setMode('forgot')}>
                <Text style={styles.link}>ลืมรหัสผ่าน?</Text>
              </TouchableOpacity>
              <TouchableOpacity onPress={() => setMode('register')}>
                <Text style={styles.link}>สมัครสมาชิก</Text>
              </TouchableOpacity>
            </>
          )}
          {(mode === 'register' || mode === 'forgot') && (
            <TouchableOpacity onPress={() => setMode('login')}>
              <Text style={styles.link}>กลับไปเข้าสู่ระบบ</Text>
            </TouchableOpacity>
          )}
        </View>
      </ScrollView>
    </KeyboardAvoidingView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  content: { padding: 24, paddingTop: 48 },
  title: { fontSize: 28, fontWeight: 'bold', marginBottom: 32 },
  inputGroup: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', marginBottom: 6, color: '#333' },
  input: {
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    fontSize: 16,
  },
  submitButton: {
    backgroundColor: '#FF6F00',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginTop: 8,
  },
  disabled: { opacity: 0.7 },
  submitText: { color: '#fff', fontSize: 16, fontWeight: 'bold' },
  links: { flexDirection: 'row', justifyContent: 'space-between', marginTop: 16 },
  link: { color: '#FF6F00', fontSize: 14 },
});
```

---

## Phone Number Auth

```typescript
// src/services/firebase/phoneAuth.ts
import auth from '@react-native-firebase/auth';

export const phoneAuth = {
  sendCode: async (phoneNumber: string) => {
    // phoneNumber ต้องมี country code: +66812345678
    const confirmation = await auth().signInWithPhoneNumber(phoneNumber);
    return confirmation;
  },

  verifyCode: async (
    confirmation: auth.ConfirmationResult,
    code: string
  ) => {
    const userCredential = await confirmation.confirm(code);
    return userCredential?.user;
  },
};
```

### Phone Auth Screen

```typescript
// src/screens/auth/PhoneAuthScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  ActivityIndicator,
} from 'react-native';
import auth from '@react-native-firebase/auth';

type Step = 'phone' | 'code';

export const PhoneAuthScreen: React.FC = () => {
  const [step, setStep] = useState<Step>('phone');
  const [phone, setPhone] = useState('');
  const [code, setCode] = useState('');
  const [confirmation, setConfirmation] = useState<any>(null);
  const [isLoading, setIsLoading] = useState(false);

  const handleSendCode = async () => {
    if (phone.length < 9) {
      Alert.alert('ผิดพลาด', 'กรุณาใส่เบอร์โทรที่ถูกต้อง');
      return;
    }

    setIsLoading(true);
    try {
      // แปลงเป็น international format
      const phoneNumber = phone.startsWith('0')
        ? `+66${phone.substring(1)}`
        : phone;

      const result = await auth().signInWithPhoneNumber(phoneNumber);
      setConfirmation(result);
      setStep('code');
      Alert.alert('สำเร็จ', `ส่ง OTP ไปที่ ${phoneNumber}`);
    } catch (error: any) {
      Alert.alert('ผิดพลาด', error.message);
    } finally {
      setIsLoading(false);
    }
  };

  const handleVerifyCode = async () => {
    if (code.length !== 6) {
      Alert.alert('ผิดพลาด', 'กรุณาใส่ OTP 6 หลัก');
      return;
    }

    setIsLoading(true);
    try {
      await confirmation.confirm(code);
      Alert.alert('สำเร็จ', 'ยืนยันเบอร์โทรสำเร็จ');
    } catch (error: any) {
      Alert.alert('ผิดพลาด', 'OTP ไม่ถูกต้องหรือหมดอายุ');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>
        {step === 'phone' ? '📱 ยืนยันเบอร์โทร' : '🔢 ใส่ OTP'}
      </Text>

      {step === 'phone' ? (
        <>
          <Text style={styles.description}>
            ใส่เบอร์โทรศัพท์เพื่อรับ OTP
          </Text>
          <View style={styles.phoneInput}>
            <View style={styles.countryCode}>
              <Text style={styles.countryCodeText}>🇹🇭 +66</Text>
            </View>
            <TextInput
              value={phone}
              onChangeText={setPhone}
              placeholder="8X-XXXX-XXXX"
              keyboardType="phone-pad"
              style={styles.phoneNumber}
              maxLength={10}
            />
          </View>
          <TouchableOpacity
            onPress={handleSendCode}
            style={[styles.button, isLoading && styles.disabled]}
            disabled={isLoading}
          >
            {isLoading ? (
              <ActivityIndicator color="#fff" />
            ) : (
              <Text style={styles.buttonText}>ส่ง OTP</Text>
            )}
          </TouchableOpacity>
        </>
      ) : (
        <>
          <Text style={styles.description}>
            ใส่รหัส OTP 6 หลักที่ส่งไปยัง {phone}
          </Text>
          <TextInput
            value={code}
            onChangeText={setCode}
            placeholder="000000"
            keyboardType="number-pad"
            maxLength={6}
            style={styles.codeInput}
            textAlign="center"
          />
          <TouchableOpacity
            onPress={handleVerifyCode}
            style={[styles.button, isLoading && styles.disabled]}
            disabled={isLoading}
          >
            {isLoading ? (
              <ActivityIndicator color="#fff" />
            ) : (
              <Text style={styles.buttonText}>ยืนยัน OTP</Text>
            )}
          </TouchableOpacity>
          <TouchableOpacity
            onPress={() => setStep('phone')}
            style={styles.backButton}
          >
            <Text style={styles.backText}>เปลี่ยนเบอร์โทร</Text>
          </TouchableOpacity>
        </>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24, backgroundColor: '#fff' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 8 },
  description: { fontSize: 14, color: '#666', marginBottom: 24 },
  phoneInput: { flexDirection: 'row', marginBottom: 16 },
  countryCode: {
    backgroundColor: '#F5F5F5',
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    marginRight: 8,
    justifyContent: 'center',
  },
  countryCodeText: { fontSize: 14 },
  phoneNumber: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    fontSize: 16,
  },
  codeInput: {
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    fontSize: 28,
    letterSpacing: 8,
    marginBottom: 16,
  },
  button: {
    backgroundColor: '#FF6F00',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginBottom: 12,
  },
  disabled: { opacity: 0.7 },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: 'bold' },
  backButton: { alignItems: 'center', padding: 12 },
  backText: { color: '#FF6F00', fontSize: 14 },
});
```

---

## Google/Facebook Auth กับ Firebase

```typescript
// src/services/firebase/socialAuth.ts
import auth from '@react-native-firebase/auth';
import { GoogleSignin } from '@react-native-google-signin/google-signin';
import { LoginManager, AccessToken } from 'react-native-fbsdk-next';

// Setup Google Sign-In
GoogleSignin.configure({
  webClientId: 'YOUR_WEB_CLIENT_ID.apps.googleusercontent.com',
});

export const firebaseSocialAuth = {
  googleSignIn: async () => {
    // ตรวจสอบว่า Google Play Services พร้อม
    await GoogleSignin.hasPlayServices({ showPlayServicesUpdateDialog: true });

    // Sign in กับ Google
    const signInResult = await GoogleSignin.signIn();
    const idToken = signInResult.data?.idToken;

    if (!idToken) throw new Error('No Google ID token');

    // สร้าง Google credential
    const googleCredential = auth.GoogleAuthProvider.credential(idToken);

    // Sign in กับ Firebase
    return auth().signInWithCredential(googleCredential);
  },

  facebookSignIn: async () => {
    // Login กับ Facebook
    const result = await LoginManager.logInWithPermissions([
      'public_profile',
      'email',
    ]);

    if (result.isCancelled) {
      throw new Error('Facebook login cancelled');
    }

    // ดึง access token
    const data = await AccessToken.getCurrentAccessToken();
    if (!data) throw new Error('No Facebook access token');

    // สร้าง Facebook credential
    const facebookCredential = auth.FacebookAuthProvider.credential(
      data.accessToken
    );

    // Sign in กับ Firebase
    return auth().signInWithCredential(facebookCredential);
  },

  appleSignIn: async () => {
    const { appleAuth } = require('@invertase/react-native-apple-authentication');

    // Request Apple credential
    const appleAuthRequestResponse = await appleAuth.performRequest({
      requestedOperation: appleAuth.Operation.LOGIN,
      requestedScopes: [appleAuth.Scope.EMAIL, appleAuth.Scope.FULL_NAME],
    });

    const { identityToken, nonce } = appleAuthRequestResponse;
    if (!identityToken) throw new Error('No Apple identity token');

    // สร้าง Apple credential
    const appleCredential = auth.AppleAuthProvider.credential(
      identityToken,
      nonce
    );

    // Sign in กับ Firebase
    return auth().signInWithCredential(appleCredential);
  },
};
```

---

## Email Verification

```typescript
// src/screens/auth/EmailVerificationScreen.tsx
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Alert,
  ActivityIndicator,
} from 'react-native';
import auth from '@react-native-firebase/auth';

export const EmailVerificationScreen: React.FC = () => {
  const [isLoading, setIsLoading] = useState(false);
  const [countdown, setCountdown] = useState(0);
  const user = auth().currentUser;

  useEffect(() => {
    if (countdown > 0) {
      const timer = setTimeout(() => setCountdown(countdown - 1), 1000);
      return () => clearTimeout(timer);
    }
  }, [countdown]);

  // Check verification status ทุก 3 วินาที
  useEffect(() => {
    const interval = setInterval(async () => {
      await user?.reload();
      if (user?.emailVerified) {
        clearInterval(interval);
        Alert.alert('สำเร็จ', 'ยืนยัน email แล้ว!');
      }
    }, 3000);

    return () => clearInterval(interval);
  }, []);

  const handleResend = async () => {
    if (countdown > 0) return;

    setIsLoading(true);
    try {
      await user?.sendEmailVerification();
      setCountdown(60); // รอ 60 วินาทีก่อนส่งใหม่
      Alert.alert('สำเร็จ', 'ส่ง email ยืนยันแล้ว');
    } catch (error: any) {
      Alert.alert('ผิดพลาด', error.message);
    } finally {
      setIsLoading(false);
    }
  };

  const handleLogout = () => {
    auth().signOut();
  };

  return (
    <View style={styles.container}>
      <Text style={styles.icon}>📧</Text>
      <Text style={styles.title}>ยืนยัน Email ของคุณ</Text>
      <Text style={styles.description}>
        ส่ง email ยืนยันไปที่{'\n'}
        <Text style={styles.email}>{user?.email}</Text>
      </Text>

      <Text style={styles.hint}>
        กรุณาตรวจสอบ inbox และ spam folder
        และคลิกลิงก์ในอีเมลเพื่อยืนยัน
      </Text>

      <TouchableOpacity
        onPress={handleResend}
        style={[
          styles.resendButton,
          (isLoading || countdown > 0) && styles.disabled,
        ]}
        disabled={isLoading || countdown > 0}
      >
        {isLoading ? (
          <ActivityIndicator color="#fff" />
        ) : (
          <Text style={styles.resendText}>
            {countdown > 0 ? `ส่งใหม่ใน ${countdown} วินาที` : 'ส่ง email ใหม่'}
          </Text>
        )}
      </TouchableOpacity>

      <TouchableOpacity onPress={handleLogout} style={styles.logoutButton}>
        <Text style={styles.logoutText}>ใช้ email อื่น</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 24,
    backgroundColor: '#fff',
  },
  icon: { fontSize: 72, marginBottom: 16 },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 12, textAlign: 'center' },
  description: { fontSize: 16, color: '#666', textAlign: 'center', marginBottom: 16 },
  email: { fontWeight: 'bold', color: '#FF6F00' },
  hint: {
    fontSize: 14,
    color: '#999',
    textAlign: 'center',
    lineHeight: 20,
    marginBottom: 32,
  },
  resendButton: {
    backgroundColor: '#FF6F00',
    paddingHorizontal: 32,
    paddingVertical: 16,
    borderRadius: 12,
    marginBottom: 12,
    minWidth: 200,
    alignItems: 'center',
  },
  disabled: { opacity: 0.6 },
  resendText: { color: '#fff', fontWeight: 'bold', fontSize: 16 },
  logoutButton: { padding: 12 },
  logoutText: { color: '#FF6F00', fontSize: 14 },
});
```

---

## Workshop: Firebase Auth App

### Auth State Management

```typescript
// src/hooks/useFirebaseAuth.ts
import { useState, useEffect } from 'react';
import auth, { FirebaseAuthTypes } from '@react-native-firebase/auth';

export function useFirebaseAuth() {
  const [user, setUser] = useState<FirebaseAuthTypes.User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const unsubscribe = auth().onAuthStateChanged((firebaseUser) => {
      setUser(firebaseUser);
      setIsLoading(false);
    });

    return unsubscribe; // Cleanup
  }, []);

  return { user, isLoading };
}
```

### Navigation ตาม Auth State

```typescript
// src/navigation/AppNavigator.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { View, ActivityIndicator } from 'react-native';
import { useFirebaseAuth } from '../hooks/useFirebaseAuth';

const Stack = createNativeStackNavigator();

export function AppNavigator() {
  const { user, isLoading } = useFirebaseAuth();

  if (isLoading) {
    return (
      <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
        <ActivityIndicator size="large" color="#FF6F00" />
      </View>
    );
  }

  return (
    <NavigationContainer>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {!user ? (
          // Auth screens
          <>
            <Stack.Screen name="SocialLogin" component={SocialLoginScreen} />
            <Stack.Screen name="EmailAuth" component={EmailAuthScreen} />
            <Stack.Screen name="PhoneAuth" component={PhoneAuthScreen} />
          </>
        ) : !user.emailVerified && user.providerData[0]?.providerId === 'password' ? (
          // Email verification
          <Stack.Screen name="EmailVerification" component={EmailVerificationScreen} />
        ) : (
          // Main app
          <>
            <Stack.Screen name="Home" component={HomeScreen} />
            <Stack.Screen name="Profile" component={ProfileScreen} />
          </>
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

---

## Tips and Best Practices

### 1. Error Code Mapping

```typescript
export function getFirebaseAuthErrorMessage(errorCode: string): string {
  const messages: { [key: string]: string } = {
    'auth/user-not-found': 'ไม่พบบัญชีนี้',
    'auth/wrong-password': 'รหัสผ่านไม่ถูกต้อง',
    'auth/email-already-in-use': 'Email นี้ถูกใช้งานแล้ว',
    'auth/weak-password': 'รหัสผ่านไม่ปลอดภัยพอ',
    'auth/invalid-email': 'รูปแบบ email ไม่ถูกต้อง',
    'auth/too-many-requests': 'บัญชีถูก lock ชั่วคราว',
    'auth/network-request-failed': 'ไม่มีการเชื่อมต่ออินเทอร์เน็ต',
    'auth/requires-recent-login': 'กรุณา login ใหม่',
    'auth/popup-closed-by-user': 'ยกเลิกการ login',
    'auth/account-exists-with-different-credential':
      'บัญชีนี้มีอยู่แล้วด้วย provider อื่น',
  };
  return messages[errorCode] || 'เกิดข้อผิดพลาด กรุณาลองใหม่';
}
```

### 2. ใช้ onAuthStateChanged ไม่ใช่ currentUser โดยตรง

```typescript
// ❌ อาจได้ null ก่อน auth state ถูก restore
const user = auth().currentUser;

// ✅ รอ auth state
auth().onAuthStateChanged((user) => {
  if (user) {
    // User signed in
  } else {
    // User signed out
  }
});
```

### 3. Security Rules

เพิ่ม Firestore Security Rules:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users สามารถอ่าน/เขียน document ของตัวเองเท่านั้น
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

---

## สรุป

Firebase Authentication ครอบคลุม:

1. **Email/Password** - สมัครสมาชิก, Login, Reset password
2. **Phone Auth** - OTP สำหรับ mobile users
3. **Social Login** - Google, Facebook, Apple
4. **Email Verification** - ยืนยัน email ก่อนใช้งาน
5. **Auth State** - `onAuthStateChanged` สำหรับ real-time auth state

ใน Part 039 จะเรียนเรื่อง **Firestore Database** สำหรับเก็บข้อมูล
