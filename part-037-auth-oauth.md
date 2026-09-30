# Part 037: Authentication - OAuth และ Social Login

## OAuth คืออะไร?

OAuth 2.0 เป็น authorization framework ที่อนุญาตให้ third-party applications เข้าถึงข้อมูลของ user โดยไม่ต้องรู้ password

### OAuth Flow

```
User → App → Google → User Consent → Authorization Code → App → Access Token → Google API
```

1. App redirect user ไป Google login page
2. User login และ grant permissions
3. Google redirect กลับมาพร้อม authorization code
4. App แลก code เป็น access token
5. App ใช้ token เพื่อ access API

---

## การติดตั้ง

```bash
# Expo
npx expo install expo-auth-session expo-web-browser expo-crypto

# หรือ React Native (bare)
npm install @react-native-google-signin/google-signin
npm install react-native-fbsdk-next
```

---

## Google Sign-In

### Setup Google OAuth

1. ไปที่ [Google Cloud Console](https://console.cloud.google.com)
2. สร้าง OAuth 2.0 Client IDs สำหรับ iOS และ Android
3. เพิ่ม redirect URIs

### ใช้ Expo Auth Session

```typescript
// src/services/googleAuth.ts
import * as Google from 'expo-auth-session/providers/google';
import * as WebBrowser from 'expo-web-browser';
import { useEffect } from 'react';

WebBrowser.maybeCompleteAuthSession();

const GOOGLE_CONFIG = {
  androidClientId: 'YOUR_ANDROID_CLIENT_ID.apps.googleusercontent.com',
  iosClientId: 'YOUR_IOS_CLIENT_ID.apps.googleusercontent.com',
  webClientId: 'YOUR_WEB_CLIENT_ID.apps.googleusercontent.com',
};

export function useGoogleAuth() {
  const [request, response, promptAsync] = Google.useAuthRequest({
    androidClientId: GOOGLE_CONFIG.androidClientId,
    iosClientId: GOOGLE_CONFIG.iosClientId,
    webClientId: GOOGLE_CONFIG.webClientId,
    scopes: ['openid', 'profile', 'email'],
  });

  return { request, response, promptAsync };
}

// ดึงข้อมูล user จาก Google
export async function fetchGoogleUserInfo(
  accessToken: string
): Promise<GoogleUser> {
  const response = await fetch(
    'https://www.googleapis.com/userinfo/v2/me',
    {
      headers: { Authorization: `Bearer ${accessToken}` },
    }
  );
  return response.json();
}

export interface GoogleUser {
  id: string;
  email: string;
  name: string;
  picture: string;
  verified_email: boolean;
}
```

### Google Sign-In Component

```typescript
// src/components/auth/GoogleSignInButton.tsx
import React, { useEffect, useState } from 'react';
import { TouchableOpacity, Text, View, StyleSheet, Alert, ActivityIndicator } from 'react-native';
import { useGoogleAuth, fetchGoogleUserInfo } from '../../services/googleAuth';
import { authApi } from '../../services/api/authApi';
import { useAuth } from '../../contexts/AuthContext';

export const GoogleSignInButton: React.FC = () => {
  const { loginWithSocial } = useAuth();
  const { request, response, promptAsync } = useGoogleAuth();
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    if (response?.type === 'success') {
      handleGoogleResponse(response);
    } else if (response?.type === 'error') {
      Alert.alert('ผิดพลาด', 'Google login ล้มเหลว');
    }
  }, [response]);

  const handleGoogleResponse = async (response: any) => {
    setIsLoading(true);
    try {
      const { authentication } = response;
      const googleUser = await fetchGoogleUserInfo(authentication.accessToken);

      // ส่ง token ไปยัง backend
      await authApi.socialLogin({
        provider: 'google',
        token: authentication.accessToken,
        idToken: authentication.idToken,
      });

    } catch (error: any) {
      Alert.alert('ผิดพลาด', error.message || 'เข้าสู่ระบบด้วย Google ล้มเหลว');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <TouchableOpacity
      onPress={() => promptAsync()}
      style={[styles.button, (!request || isLoading) && styles.disabled]}
      disabled={!request || isLoading}
    >
      {isLoading ? (
        <ActivityIndicator color="#333" size="small" />
      ) : (
        <>
          <Text style={styles.googleIcon}>G</Text>
          <Text style={styles.buttonText}>เข้าสู่ระบบด้วย Google</Text>
        </>
      )}
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  button: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: '#fff',
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    gap: 12,
    elevation: 1,
  },
  disabled: { opacity: 0.6 },
  googleIcon: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#4285F4',
    width: 24,
    textAlign: 'center',
  },
  buttonText: { fontSize: 15, color: '#333', fontWeight: '500' },
});
```

---

## Facebook Login

### Setup

```bash
# Expo
npx expo install expo-auth-session

# หรือ Native SDK
npm install react-native-fbsdk-next
```

### Facebook Auth ด้วย Expo

```typescript
// src/services/facebookAuth.ts
import * as Facebook from 'expo-auth-session/providers/facebook';
import * as WebBrowser from 'expo-web-browser';

WebBrowser.maybeCompleteAuthSession();

const FB_APP_ID = 'YOUR_FACEBOOK_APP_ID';

export function useFacebookAuth() {
  const [request, response, promptAsync] = Facebook.useAuthRequest({
    clientId: FB_APP_ID,
    scopes: ['public_profile', 'email'],
  });

  return { request, response, promptAsync };
}

export async function fetchFacebookUserInfo(
  accessToken: string
): Promise<FacebookUser> {
  const response = await fetch(
    `https://graph.facebook.com/me?fields=id,name,email,picture&access_token=${accessToken}`
  );
  return response.json();
}

export interface FacebookUser {
  id: string;
  name: string;
  email: string;
  picture: {
    data: {
      url: string;
    };
  };
}
```

### Facebook Login Button

```typescript
// src/components/auth/FacebookLoginButton.tsx
import React, { useEffect, useState } from 'react';
import { TouchableOpacity, Text, View, StyleSheet, Alert, ActivityIndicator } from 'react-native';
import { useFacebookAuth, fetchFacebookUserInfo } from '../../services/facebookAuth';
import { authApi } from '../../services/api/authApi';

export const FacebookLoginButton: React.FC = () => {
  const { request, response, promptAsync } = useFacebookAuth();
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    if (response?.type === 'success' && response.authentication?.accessToken) {
      handleFacebookResponse(response.authentication.accessToken);
    }
  }, [response]);

  const handleFacebookResponse = async (accessToken: string) => {
    setIsLoading(true);
    try {
      const fbUser = await fetchFacebookUserInfo(accessToken);

      await authApi.socialLogin({
        provider: 'facebook',
        token: accessToken,
        userId: fbUser.id,
      });

    } catch (error: any) {
      Alert.alert('ผิดพลาด', error.message || 'Facebook login ล้มเหลว');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <TouchableOpacity
      onPress={() => promptAsync()}
      style={[styles.button, (!request || isLoading) && styles.disabled]}
      disabled={!request || isLoading}
    >
      {isLoading ? (
        <ActivityIndicator color="#fff" size="small" />
      ) : (
        <>
          <Text style={styles.fbIcon}>f</Text>
          <Text style={styles.buttonText}>เข้าสู่ระบบด้วย Facebook</Text>
        </>
      )}
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  button: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: '#1877F2',
    borderRadius: 12,
    padding: 14,
    gap: 12,
  },
  disabled: { opacity: 0.6 },
  fbIcon: {
    fontSize: 20,
    fontWeight: 'bold',
    color: '#fff',
    width: 24,
    textAlign: 'center',
  },
  buttonText: { fontSize: 15, color: '#fff', fontWeight: '500' },
});
```

---

## Apple Sign-In

Apple Sign-In จำเป็นสำหรับ iOS app ที่ใช้ social login

### Setup

```bash
npx expo install expo-apple-authentication
```

เพิ่มใน `app.json`:
```json
{
  "expo": {
    "ios": {
      "usesAppleSignIn": true
    }
  }
}
```

### Apple Sign-In Component

```typescript
// src/components/auth/AppleSignInButton.tsx
import React, { useState } from 'react';
import { Platform, Alert } from 'react-native';
import * as AppleAuthentication from 'expo-apple-authentication';
import { authApi } from '../../services/api/authApi';

export const AppleSignInButton: React.FC = () => {
  const [isLoading, setIsLoading] = useState(false);

  if (Platform.OS !== 'ios') return null;

  const handleAppleSignIn = async () => {
    setIsLoading(true);
    try {
      const credential = await AppleAuthentication.signInAsync({
        requestedScopes: [
          AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
          AppleAuthentication.AppleAuthenticationScope.EMAIL,
        ],
      });

      // ส่ง identity token ไป backend
      await authApi.socialLogin({
        provider: 'apple',
        identityToken: credential.identityToken!,
        authorizationCode: credential.authorizationCode!,
        fullName: credential.fullName
          ? `${credential.fullName.givenName} ${credential.fullName.familyName}`
          : undefined,
      });

    } catch (error: any) {
      if (error.code !== 'ERR_REQUEST_CANCELED') {
        Alert.alert('ผิดพลาด', 'Apple Sign-In ล้มเหลว');
      }
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <AppleAuthentication.AppleAuthenticationButton
      buttonType={AppleAuthentication.AppleAuthenticationButtonType.SIGN_IN}
      buttonStyle={AppleAuthentication.AppleAuthenticationButtonStyle.BLACK}
      cornerRadius={12}
      style={{ width: '100%', height: 50 }}
      onPress={handleAppleSignIn}
    />
  );
};
```

---

## OAuth Flow แบบ Custom

สำหรับ providers อื่น ๆ ที่ไม่มี dedicated library

```typescript
// src/services/oauthService.ts
import * as WebBrowser from 'expo-web-browser';
import * as Linking from 'expo-linking';
import { makeRedirectUri } from 'expo-auth-session';
import CryptoJS from 'crypto-js';

interface OAuthConfig {
  clientId: string;
  authorizationEndpoint: string;
  tokenEndpoint: string;
  scope: string[];
  provider: string;
}

export class OAuthService {
  private config: OAuthConfig;

  constructor(config: OAuthConfig) {
    this.config = config;
  }

  private generateCodeVerifier(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return btoa(String.fromCharCode(...array))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=/g, '');
  }

  private generateCodeChallenge(verifier: string): string {
    return CryptoJS.SHA256(verifier)
      .toString(CryptoJS.enc.Base64)
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=/g, '');
  }

  async signIn(): Promise<{ code: string; state: string }> {
    const redirectUri = makeRedirectUri({ scheme: 'myapp' });
    const state = Math.random().toString(36).substring(2);
    const codeVerifier = this.generateCodeVerifier();
    const codeChallenge = this.generateCodeChallenge(codeVerifier);

    const params = new URLSearchParams({
      client_id: this.config.clientId,
      redirect_uri: redirectUri,
      response_type: 'code',
      scope: this.config.scope.join(' '),
      state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
    });

    const authUrl = `${this.config.authorizationEndpoint}?${params}`;

    const result = await WebBrowser.openAuthSessionAsync(authUrl, redirectUri);

    if (result.type !== 'success') {
      throw new Error('OAuth cancelled');
    }

    const url = new URL(result.url);
    const code = url.searchParams.get('code');
    const returnedState = url.searchParams.get('state');

    if (!code || returnedState !== state) {
      throw new Error('Invalid OAuth response');
    }

    return { code, state };
  }
}

// GitHub OAuth
export const githubOAuth = new OAuthService({
  clientId: 'YOUR_GITHUB_CLIENT_ID',
  authorizationEndpoint: 'https://github.com/login/oauth/authorize',
  tokenEndpoint: 'https://github.com/login/oauth/access_token',
  scope: ['user:email', 'read:user'],
  provider: 'github',
});
```

---

## Workshop: Social Login App

### Complete Social Login Screen

```typescript
// src/screens/auth/SocialLoginScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
  Alert,
  ActivityIndicator,
  SafeAreaView,
  Image,
  Platform,
} from 'react-native';
import { GoogleSignInButton } from '../../components/auth/GoogleSignInButton';
import { FacebookLoginButton } from '../../components/auth/FacebookLoginButton';
import { AppleSignInButton } from '../../components/auth/AppleSignInButton';

export const SocialLoginScreen: React.FC = ({ navigation }: any) => {
  return (
    <SafeAreaView style={styles.container}>
      <View style={styles.content}>
        {/* Header */}
        <View style={styles.header}>
          <Text style={styles.logoText}>🚀</Text>
          <Text style={styles.title}>ยินดีต้อนรับ</Text>
          <Text style={styles.subtitle}>
            เข้าสู่ระบบเพื่อเริ่มต้นใช้งาน
          </Text>
        </View>

        {/* Social Buttons */}
        <View style={styles.socialButtons}>
          <GoogleSignInButton />

          <View style={styles.spacer} />

          <FacebookLoginButton />

          {Platform.OS === 'ios' && (
            <>
              <View style={styles.spacer} />
              <AppleSignInButton />
            </>
          )}
        </View>

        {/* Divider */}
        <View style={styles.divider}>
          <View style={styles.dividerLine} />
          <Text style={styles.dividerText}>หรือ</Text>
          <View style={styles.dividerLine} />
        </View>

        {/* Email Login */}
        <TouchableOpacity
          onPress={() => navigation.navigate('Login')}
          style={styles.emailButton}
        >
          <Text style={styles.emailButtonIcon}>📧</Text>
          <Text style={styles.emailButtonText}>เข้าสู่ระบบด้วยอีเมล</Text>
        </TouchableOpacity>

        {/* Register */}
        <View style={styles.registerRow}>
          <Text style={styles.registerText}>ยังไม่มีบัญชี? </Text>
          <TouchableOpacity onPress={() => navigation.navigate('Register')}>
            <Text style={styles.registerLink}>สมัครสมาชิก</Text>
          </TouchableOpacity>
        </View>

        {/* Terms */}
        <Text style={styles.terms}>
          การเข้าสู่ระบบถือว่าคุณยอมรับ{' '}
          <Text style={styles.termsLink}>เงื่อนไขการใช้งาน</Text>
          {' '}และ{' '}
          <Text style={styles.termsLink}>นโยบายความเป็นส่วนตัว</Text>
        </Text>
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  content: { flex: 1, paddingHorizontal: 24, justifyContent: 'center' },
  header: { alignItems: 'center', marginBottom: 40 },
  logoText: { fontSize: 72, marginBottom: 16 },
  title: { fontSize: 28, fontWeight: 'bold', color: '#333', marginBottom: 8 },
  subtitle: { fontSize: 16, color: '#666', textAlign: 'center' },
  socialButtons: { marginBottom: 24 },
  spacer: { height: 12 },
  divider: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 24,
  },
  dividerLine: { flex: 1, height: 1, backgroundColor: '#E0E0E0' },
  dividerText: { marginHorizontal: 12, color: '#999', fontSize: 14 },
  emailButton: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 12,
    padding: 14,
    gap: 8,
    marginBottom: 20,
  },
  emailButtonIcon: { fontSize: 18 },
  emailButtonText: { fontSize: 15, color: '#333' },
  registerRow: {
    flexDirection: 'row',
    justifyContent: 'center',
    marginBottom: 24,
  },
  registerText: { fontSize: 14, color: '#666' },
  registerLink: { fontSize: 14, color: '#6200EE', fontWeight: 'bold' },
  terms: {
    fontSize: 12,
    color: '#999',
    textAlign: 'center',
    lineHeight: 18,
  },
  termsLink: { color: '#6200EE' },
});
```

### Auth API Service

```typescript
// src/services/api/authApi.ts
import { apiClient } from '../axiosInstance';

interface SocialLoginRequest {
  provider: 'google' | 'facebook' | 'apple';
  token?: string;
  idToken?: string;
  identityToken?: string;
  authorizationCode?: string;
  userId?: string;
  fullName?: string;
}

interface AuthResponse {
  user: {
    id: string;
    name: string;
    email: string;
    avatar?: string;
    role: string;
  };
  accessToken: string;
  refreshToken: string;
}

export const authApi = {
  login: (data: { email: string; password: string }) =>
    apiClient.post<AuthResponse>('/auth/login', data),

  register: (data: { name: string; email: string; password: string }) =>
    apiClient.post<AuthResponse>('/auth/register', data),

  socialLogin: (data: SocialLoginRequest) =>
    apiClient.post<AuthResponse>('/auth/social', data),

  refreshToken: (refreshToken: string) =>
    apiClient.post<{ accessToken: string; refreshToken: string }>(
      '/auth/refresh',
      { refreshToken }
    ),

  logout: () => apiClient.post('/auth/logout'),

  forgotPassword: (email: string) =>
    apiClient.post('/auth/forgot-password', { email }),

  resetPassword: (token: string, newPassword: string) =>
    apiClient.post('/auth/reset-password', { token, newPassword }),

  verifyEmail: (token: string) =>
    apiClient.post('/auth/verify-email', { token }),

  resendVerification: () =>
    apiClient.post('/auth/resend-verification'),
};
```

---

## Backend Implementation Guide

เมื่อ social login สำเร็จ frontend ส่ง token ไป backend

### Backend ตรวจสอบ Google Token

```
POST /auth/social
{
  "provider": "google",
  "token": "ya29.xxx...",
  "idToken": "eyJhbGci..."
}
```

Backend process:
1. ตรวจสอบ idToken กับ Google API
2. ดึงข้อมูล user (email, name, picture)
3. สร้างหรือ login user ในระบบ
4. Return access token และ refresh token ของระบบ

---

## Tips and Best Practices

### 1. Handle "Sign in with Apple" Privacy

Apple อนุญาตให้ user ซ่อน email ได้ ต้องรองรับ relay email

```typescript
const handleAppleSignIn = async () => {
  const credential = await AppleAuthentication.signInAsync({...});

  // email อาจเป็น relay email หรือ null ในครั้งต่อไป
  const email = credential.email; // อาจเป็น null หลัง login ครั้งแรก

  await authApi.socialLogin({
    provider: 'apple',
    identityToken: credential.identityToken!,
    // ส่ง fullName เฉพาะครั้งแรก
    fullName: credential.fullName?.givenName
      ? `${credential.fullName.givenName} ${credential.fullName.familyName}`
      : undefined,
    email: email || undefined,
  });
};
```

### 2. Silent Re-authentication

```typescript
// ตรวจสอบว่า Apple credential ยังใช้ได้ไหม
const checkAppleCredential = async (userId: string) => {
  const state = await AppleAuthentication.getCredentialStateAsync(userId);

  if (state === AppleAuthentication.AppleAuthenticationCredentialState.REVOKED) {
    // User revoked access
    await logout();
  }
};
```

### 3. Linking Accounts

```typescript
// ให้ user เชื่อม social accounts เข้ากับบัญชีเดิม
const linkGoogleAccount = async () => {
  const { response, promptAsync } = useGoogleAuth();

  // ... get Google token
  await authApi.linkSocialAccount({
    provider: 'google',
    token: googleAccessToken,
  });
};
```

---

## สรุป

Social Login ด้วย OAuth:

1. **Google Sign-In** - ใช้ expo-auth-session/providers/google
2. **Facebook Login** - ใช้ expo-auth-session/providers/facebook
3. **Apple Sign-In** - ใช้ expo-apple-authentication (iOS เท่านั้น)
4. **Custom OAuth** - ใช้ expo-web-browser + expo-auth-session
5. **Backend Verification** - ส่ง token ไป backend เพื่อ verify

ใน Part 038 จะเรียนเรื่อง **Firebase Authentication** ที่ครอบคลุมทุก auth methods
