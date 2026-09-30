# Part 061: Internationalization (i18n) และ Localization ใน React Native

## บทนำ

Internationalization (i18n) และ Localization (l10n) คือกระบวนการสำคัญในการพัฒนาแอปพลิเคชันที่รองรับผู้ใช้จากหลากหลายประเทศและภาษา ในบทนี้เราจะเรียนรู้วิธีการสร้างแอป React Native ที่รองรับหลายภาษาอย่างมืออาชีพ

---

## 1. ทำความเข้าใจ i18n และ l10n

### ความแตกต่างระหว่าง i18n และ l10n

- **Internationalization (i18n)**: กระบวนการออกแบบและพัฒนาซอฟต์แวร์ให้สามารถรองรับภาษาและวัฒนธรรมต่างๆ ได้ง่าย
- **Localization (l10n)**: กระบวนการปรับแต่งซอฟต์แวร์ให้เหมาะสมกับภาษา วัฒนธรรม และภูมิภาคเฉพาะ

### ทำไมต้องทำ i18n?

```
- ขยายตลาดไปยังผู้ใช้ทั่วโลก
- เพิ่มความพึงพอใจของผู้ใช้
- ปฏิบัติตามกฎหมายและข้อบังคับในบางประเทศ
- เพิ่มยอดดาวน์โหลดและรายได้
```

---

## 2. ติดตั้ง react-i18next

### การติดตั้ง Dependencies

```bash
# ติดตั้ง packages ที่จำเป็น
npm install react-i18next i18next
npm install @react-native-async-storage/async-storage
npm install i18next-react-native-language-detector
npm install intl-pluralrules

# สำหรับ iOS
cd ios && pod install && cd ..
```

### โครงสร้างโปรเจค

```
src/
├── i18n/
│   ├── index.ts
│   ├── languageDetector.ts
│   └── translations/
│       ├── en.json
│       ├── th.json
│       ├── ja.json
│       └── ar.json
├── hooks/
│   └── useTranslation.ts
└── components/
    └── LanguageSwitcher.tsx
```

---

## 3. การตั้งค่า i18next

### สร้างไฟล์ภาษา (Translation Files)

**src/i18n/translations/en.json**:
```json
{
  "common": {
    "welcome": "Welcome",
    "hello": "Hello, {{name}}!",
    "loading": "Loading...",
    "error": "An error occurred",
    "retry": "Retry",
    "cancel": "Cancel",
    "confirm": "Confirm",
    "save": "Save",
    "delete": "Delete",
    "edit": "Edit",
    "search": "Search",
    "back": "Back",
    "next": "Next",
    "done": "Done"
  },
  "auth": {
    "login": "Login",
    "logout": "Logout",
    "register": "Register",
    "email": "Email Address",
    "password": "Password",
    "forgotPassword": "Forgot Password?",
    "loginButton": "Sign In",
    "registerButton": "Create Account",
    "alreadyHaveAccount": "Already have an account?",
    "dontHaveAccount": "Don't have an account?"
  },
  "home": {
    "title": "Home",
    "greeting": "Good {{timeOfDay}}, {{name}}!",
    "recentActivity": "Recent Activity",
    "noActivity": "No recent activity",
    "items_one": "{{count}} item",
    "items_other": "{{count}} items"
  },
  "settings": {
    "title": "Settings",
    "language": "Language",
    "theme": "Theme",
    "notifications": "Notifications",
    "privacy": "Privacy",
    "about": "About"
  },
  "errors": {
    "network": "Network connection error",
    "unauthorized": "You are not authorized",
    "notFound": "Resource not found",
    "validation": {
      "required": "This field is required",
      "email": "Please enter a valid email",
      "minLength": "Minimum {{min}} characters required",
      "maxLength": "Maximum {{max}} characters allowed"
    }
  },
  "dates": {
    "today": "Today",
    "yesterday": "Yesterday",
    "tomorrow": "Tomorrow",
    "daysAgo": "{{count}} days ago",
    "daysAgo_one": "{{count}} day ago",
    "daysAgo_other": "{{count}} days ago"
  }
}
```

**src/i18n/translations/th.json**:
```json
{
  "common": {
    "welcome": "ยินดีต้อนรับ",
    "hello": "สวัสดี, {{name}}!",
    "loading": "กำลังโหลด...",
    "error": "เกิดข้อผิดพลาด",
    "retry": "ลองใหม่",
    "cancel": "ยกเลิก",
    "confirm": "ยืนยัน",
    "save": "บันทึก",
    "delete": "ลบ",
    "edit": "แก้ไข",
    "search": "ค้นหา",
    "back": "กลับ",
    "next": "ถัดไป",
    "done": "เสร็จสิ้น"
  },
  "auth": {
    "login": "เข้าสู่ระบบ",
    "logout": "ออกจากระบบ",
    "register": "สมัครสมาชิก",
    "email": "อีเมล",
    "password": "รหัสผ่าน",
    "forgotPassword": "ลืมรหัสผ่าน?",
    "loginButton": "เข้าสู่ระบบ",
    "registerButton": "สร้างบัญชี",
    "alreadyHaveAccount": "มีบัญชีอยู่แล้ว?",
    "dontHaveAccount": "ยังไม่มีบัญชี?"
  },
  "home": {
    "title": "หน้าหลัก",
    "greeting": "{{timeOfDay}}ดี, {{name}}!",
    "recentActivity": "กิจกรรมล่าสุด",
    "noActivity": "ไม่มีกิจกรรมล่าสุด",
    "items_one": "{{count}} รายการ",
    "items_other": "{{count}} รายการ"
  },
  "settings": {
    "title": "การตั้งค่า",
    "language": "ภาษา",
    "theme": "ธีม",
    "notifications": "การแจ้งเตือน",
    "privacy": "ความเป็นส่วนตัว",
    "about": "เกี่ยวกับ"
  },
  "errors": {
    "network": "ข้อผิดพลาดการเชื่อมต่อเครือข่าย",
    "unauthorized": "คุณไม่มีสิทธิ์เข้าถึง",
    "notFound": "ไม่พบข้อมูลที่ต้องการ",
    "validation": {
      "required": "กรุณากรอกข้อมูลในช่องนี้",
      "email": "กรุณากรอกอีเมลที่ถูกต้อง",
      "minLength": "ต้องมีอย่างน้อย {{min}} ตัวอักษร",
      "maxLength": "ไม่เกิน {{max}} ตัวอักษร"
    }
  },
  "dates": {
    "today": "วันนี้",
    "yesterday": "เมื่อวาน",
    "tomorrow": "พรุ่งนี้",
    "daysAgo": "{{count}} วันที่แล้ว",
    "daysAgo_one": "{{count}} วันที่แล้ว",
    "daysAgo_other": "{{count}} วันที่แล้ว"
  }
}
```

**src/i18n/translations/ja.json**:
```json
{
  "common": {
    "welcome": "ようこそ",
    "hello": "こんにちは、{{name}}さん！",
    "loading": "読み込み中...",
    "error": "エラーが発生しました",
    "retry": "再試行",
    "cancel": "キャンセル",
    "confirm": "確認",
    "save": "保存",
    "delete": "削除",
    "edit": "編集",
    "search": "検索",
    "back": "戻る",
    "next": "次へ",
    "done": "完了"
  },
  "auth": {
    "login": "ログイン",
    "logout": "ログアウト",
    "register": "登録",
    "email": "メールアドレス",
    "password": "パスワード",
    "forgotPassword": "パスワードを忘れた方",
    "loginButton": "サインイン",
    "registerButton": "アカウント作成",
    "alreadyHaveAccount": "すでにアカウントをお持ちの方",
    "dontHaveAccount": "アカウントをお持ちでない方"
  },
  "home": {
    "title": "ホーム",
    "greeting": "{{timeOfDay}}、{{name}}さん！",
    "recentActivity": "最近のアクティビティ",
    "noActivity": "最近のアクティビティはありません",
    "items_one": "{{count}}件",
    "items_other": "{{count}}件"
  },
  "settings": {
    "title": "設定",
    "language": "言語",
    "theme": "テーマ",
    "notifications": "通知",
    "privacy": "プライバシー",
    "about": "について"
  }
}
```

**src/i18n/translations/ar.json** (Arabic - RTL):
```json
{
  "common": {
    "welcome": "مرحبا",
    "hello": "مرحبا، {{name}}!",
    "loading": "جار التحميل...",
    "error": "حدث خطأ",
    "retry": "إعادة المحاولة",
    "cancel": "إلغاء",
    "confirm": "تأكيد",
    "save": "حفظ",
    "delete": "حذف",
    "edit": "تعديل",
    "search": "بحث",
    "back": "رجوع",
    "next": "التالي",
    "done": "تم"
  },
  "auth": {
    "login": "تسجيل الدخول",
    "logout": "تسجيل الخروج",
    "register": "إنشاء حساب",
    "email": "البريد الإلكتروني",
    "password": "كلمة المرور",
    "forgotPassword": "نسيت كلمة المرور؟",
    "loginButton": "دخول",
    "registerButton": "إنشاء حساب جديد"
  },
  "home": {
    "title": "الرئيسية",
    "greeting": "{{timeOfDay}}، {{name}}!",
    "recentActivity": "النشاط الأخير",
    "noActivity": "لا يوجد نشاط حديث"
  },
  "settings": {
    "title": "الإعدادات",
    "language": "اللغة",
    "theme": "المظهر",
    "notifications": "الإشعارات"
  }
}
```

---

## 4. การตั้งค่า Language Detector

**src/i18n/languageDetector.ts**:
```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { NativeModules, Platform } from 'react-native';

const LANGUAGE_KEY = '@app_language';

// ดึงภาษาจากเครื่อง
const getDeviceLanguage = (): string => {
  const deviceLanguage =
    Platform.OS === 'ios'
      ? NativeModules.SettingsManager?.settings?.AppleLocale ||
        NativeModules.SettingsManager?.settings?.AppleLanguages?.[0]
      : NativeModules.I18nManager?.localeIdentifier;

  if (deviceLanguage) {
    // แปลง locale เช่น 'th_TH' หรือ 'th-TH' เป็น 'th'
    return deviceLanguage.split(/[-_]/)[0];
  }

  return 'en'; // ค่าเริ่มต้น
};

// Language Detector สำหรับ i18next
export const languageDetector = {
  type: 'languageDetector' as const,
  async: true,
  
  detect: async (callback: (lang: string) => void) => {
    try {
      // ตรวจสอบภาษาที่บันทึกไว้ใน AsyncStorage ก่อน
      const savedLanguage = await AsyncStorage.getItem(LANGUAGE_KEY);
      
      if (savedLanguage) {
        callback(savedLanguage);
        return;
      }

      // ถ้าไม่มี ใช้ภาษาจากเครื่อง
      const deviceLanguage = getDeviceLanguage();
      callback(deviceLanguage);
    } catch (error) {
      console.error('Language detection error:', error);
      callback('en');
    }
  },

  init: () => {},

  cacheUserLanguage: async (language: string) => {
    try {
      await AsyncStorage.setItem(LANGUAGE_KEY, language);
    } catch (error) {
      console.error('Language caching error:', error);
    }
  },
};

export const saveLanguage = async (language: string): Promise<void> => {
  await AsyncStorage.setItem(LANGUAGE_KEY, language);
};

export const getSavedLanguage = async (): Promise<string | null> => {
  return AsyncStorage.getItem(LANGUAGE_KEY);
};
```

---

## 5. การตั้งค่า i18n หลัก

**src/i18n/index.ts**:
```typescript
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import 'intl-pluralrules';

import { languageDetector } from './languageDetector';
import en from './translations/en.json';
import th from './translations/th.json';
import ja from './translations/ja.json';
import ar from './translations/ar.json';

// รายชื่อภาษาที่รองรับ
export const SUPPORTED_LANGUAGES = [
  { code: 'en', name: 'English', nativeName: 'English', isRTL: false },
  { code: 'th', name: 'Thai', nativeName: 'ภาษาไทย', isRTL: false },
  { code: 'ja', name: 'Japanese', nativeName: '日本語', isRTL: false },
  { code: 'ar', name: 'Arabic', nativeName: 'العربية', isRTL: true },
] as const;

export type LanguageCode = typeof SUPPORTED_LANGUAGES[number]['code'];

const resources = {
  en: { translation: en },
  th: { translation: th },
  ja: { translation: ja },
  ar: { translation: ar },
};

i18n
  .use(languageDetector)
  .use(initReactI18next)
  .init({
    resources,
    fallbackLng: 'en',
    debug: __DEV__,
    
    interpolation: {
      escapeValue: false, // React จัดการ XSS เอง
    },

    react: {
      useSuspense: false, // สำคัญสำหรับ React Native
    },

    // ตั้งค่า namespace
    defaultNS: 'translation',
    ns: ['translation'],

    // ตั้งค่า plural
    pluralSeparator: '_',
    keySeparator: '.',

    // Handle missing keys
    saveMissing: __DEV__,
    missingKeyHandler: (lngs, ns, key) => {
      if (__DEV__) {
        console.warn(`Missing translation key: ${key} for languages: ${lngs.join(', ')}`);
      }
    },
  });

export default i18n;
```

---

## 6. การใช้งานใน App

**App.tsx**:
```typescript
import React, { useEffect, useState } from 'react';
import { I18nManager, View, ActivityIndicator } from 'react-native';
import i18n from './src/i18n';
import './src/i18n'; // import เพื่อ initialize

const App: React.FC = () => {
  const [i18nInitialized, setI18nInitialized] = useState(false);

  useEffect(() => {
    // รอให้ i18n initialize เสร็จ
    const checkI18n = async () => {
      await i18n.init();
      
      // ตั้งค่า RTL ตามภาษา
      const currentLanguage = i18n.language;
      const isRTL = ['ar', 'he', 'fa', 'ur'].includes(currentLanguage);
      
      if (I18nManager.isRTL !== isRTL) {
        I18nManager.allowRTL(isRTL);
        I18nManager.forceRTL(isRTL);
        // ต้องรีสตาร์ทแอปหลังเปลี่ยน RTL
      }
      
      setI18nInitialized(true);
    };

    checkI18n();
  }, []);

  if (!i18nInitialized) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  return (
    // ... rest of app
    <></>
  );
};

export default App;
```

---

## 7. Custom Hook สำหรับ Translation

**src/hooks/useAppTranslation.ts**:
```typescript
import { useTranslation } from 'react-i18next';
import { useCallback } from 'react';
import { I18nManager } from 'react-native';
import i18n, { SUPPORTED_LANGUAGES, LanguageCode } from '../i18n';
import { saveLanguage } from '../i18n/languageDetector';

interface UseAppTranslationReturn {
  t: (key: string, options?: Record<string, unknown>) => string;
  language: string;
  isRTL: boolean;
  changeLanguage: (lang: LanguageCode) => Promise<void>;
  supportedLanguages: typeof SUPPORTED_LANGUAGES;
  getCurrentLanguageInfo: () => typeof SUPPORTED_LANGUAGES[number] | undefined;
}

export const useAppTranslation = (): UseAppTranslationReturn => {
  const { t, i18n: i18nInstance } = useTranslation();

  const changeLanguage = useCallback(async (lang: LanguageCode) => {
    try {
      await i18n.changeLanguage(lang);
      await saveLanguage(lang);

      // ตรวจสอบว่าต้อง RTL หรือไม่
      const langInfo = SUPPORTED_LANGUAGES.find(l => l.code === lang);
      if (langInfo && langInfo.isRTL !== I18nManager.isRTL) {
        I18nManager.allowRTL(langInfo.isRTL);
        I18nManager.forceRTL(langInfo.isRTL);
        // แจ้งผู้ใช้ว่าต้องรีสตาร์ทแอป
      }
    } catch (error) {
      console.error('Failed to change language:', error);
    }
  }, []);

  const getCurrentLanguageInfo = useCallback(() => {
    return SUPPORTED_LANGUAGES.find(l => l.code === i18nInstance.language);
  }, [i18nInstance.language]);

  return {
    t,
    language: i18nInstance.language,
    isRTL: I18nManager.isRTL,
    changeLanguage,
    supportedLanguages: SUPPORTED_LANGUAGES,
    getCurrentLanguageInfo,
  };
};
```

---

## 8. Language Switcher Component

**src/components/LanguageSwitcher.tsx**:
```typescript
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  FlatList,
  Modal,
} from 'react-native';
import { useAppTranslation } from '../hooks/useAppTranslation';
import { LanguageCode } from '../i18n';

interface LanguageSwitcherProps {
  visible: boolean;
  onClose: () => void;
}

const LanguageSwitcher: React.FC<LanguageSwitcherProps> = ({ visible, onClose }) => {
  const { t, language, changeLanguage, supportedLanguages } = useAppTranslation();

  const handleLanguageSelect = async (lang: LanguageCode) => {
    await changeLanguage(lang);
    onClose();
  };

  return (
    <Modal
      visible={visible}
      transparent
      animationType="slide"
      onRequestClose={onClose}
    >
      <View style={styles.overlay}>
        <View style={styles.container}>
          <Text style={styles.title}>{t('settings.language')}</Text>
          
          <FlatList
            data={supportedLanguages}
            keyExtractor={(item) => item.code}
            renderItem={({ item }) => (
              <TouchableOpacity
                style={[
                  styles.languageItem,
                  language === item.code && styles.selectedLanguage,
                ]}
                onPress={() => handleLanguageSelect(item.code)}
              >
                <Text style={styles.languageName}>{item.nativeName}</Text>
                <Text style={styles.languageSubName}>{item.name}</Text>
                {language === item.code && (
                  <Text style={styles.checkmark}>✓</Text>
                )}
              </TouchableOpacity>
            )}
          />

          <TouchableOpacity style={styles.closeButton} onPress={onClose}>
            <Text style={styles.closeButtonText}>{t('common.cancel')}</Text>
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );
};

const styles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0, 0, 0, 0.5)',
    justifyContent: 'flex-end',
  },
  container: {
    backgroundColor: '#fff',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    padding: 20,
    maxHeight: '70%',
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 16,
    textAlign: 'center',
  },
  languageItem: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    borderRadius: 8,
    marginBottom: 8,
    backgroundColor: '#f5f5f5',
  },
  selectedLanguage: {
    backgroundColor: '#e3f2fd',
    borderWidth: 2,
    borderColor: '#2196F3',
  },
  languageName: {
    fontSize: 16,
    fontWeight: '600',
    flex: 1,
  },
  languageSubName: {
    fontSize: 14,
    color: '#666',
    marginRight: 8,
  },
  checkmark: {
    fontSize: 18,
    color: '#2196F3',
    fontWeight: 'bold',
  },
  closeButton: {
    marginTop: 16,
    padding: 16,
    backgroundColor: '#f0f0f0',
    borderRadius: 8,
    alignItems: 'center',
  },
  closeButtonText: {
    fontSize: 16,
    color: '#333',
  },
});

export default LanguageSwitcher;
```

---

## 9. Date และ Number Formatting

### การใช้ Intl API สำหรับ Format วันที่และตัวเลข

**src/utils/formatters.ts**:
```typescript
import i18n from '../i18n';

// Locale mapping สำหรับ Intl API
const LOCALE_MAP: Record<string, string> = {
  en: 'en-US',
  th: 'th-TH',
  ja: 'ja-JP',
  ar: 'ar-SA',
};

const getLocale = (): string => {
  const lang = i18n.language || 'en';
  return LOCALE_MAP[lang] || 'en-US';
};

// Format วันที่
export const formatDate = (
  date: Date | string | number,
  options?: Intl.DateTimeFormatOptions
): string => {
  const locale = getLocale();
  const dateObj = typeof date === 'string' || typeof date === 'number' 
    ? new Date(date) 
    : date;

  const defaultOptions: Intl.DateTimeFormatOptions = {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    ...options,
  };

  return new Intl.DateTimeFormat(locale, defaultOptions).format(dateObj);
};

// Format เวลา
export const formatTime = (
  date: Date | string | number,
  options?: Intl.DateTimeFormatOptions
): string => {
  const locale = getLocale();
  const dateObj = typeof date === 'string' || typeof date === 'number' 
    ? new Date(date) 
    : date;

  const defaultOptions: Intl.DateTimeFormatOptions = {
    hour: '2-digit',
    minute: '2-digit',
    ...options,
  };

  return new Intl.DateTimeFormat(locale, defaultOptions).format(dateObj);
};

// Format วันที่และเวลา
export const formatDateTime = (
  date: Date | string | number,
  options?: Intl.DateTimeFormatOptions
): string => {
  const locale = getLocale();
  const dateObj = typeof date === 'string' || typeof date === 'number' 
    ? new Date(date) 
    : date;

  const defaultOptions: Intl.DateTimeFormatOptions = {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
    ...options,
  };

  return new Intl.DateTimeFormat(locale, defaultOptions).format(dateObj);
};

// Format ตัวเลข
export const formatNumber = (
  number: number,
  options?: Intl.NumberFormatOptions
): string => {
  const locale = getLocale();
  return new Intl.NumberFormat(locale, options).format(number);
};

// Format สกุลเงิน
export const formatCurrency = (
  amount: number,
  currency: string = 'USD'
): string => {
  const locale = getLocale();
  
  // Map ภาษาเป็นสกุลเงินเริ่มต้น
  const defaultCurrencies: Record<string, string> = {
    en: 'USD',
    th: 'THB',
    ja: 'JPY',
    ar: 'SAR',
  };

  const lang = i18n.language || 'en';
  const actualCurrency = currency || defaultCurrencies[lang] || 'USD';

  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency: actualCurrency,
  }).format(amount);
};

// Format เปอร์เซ็นต์
export const formatPercent = (
  value: number,
  options?: Intl.NumberFormatOptions
): string => {
  const locale = getLocale();
  return new Intl.NumberFormat(locale, {
    style: 'percent',
    minimumFractionDigits: 0,
    maximumFractionDigits: 2,
    ...options,
  }).format(value);
};

// Format ระยะห่างวันที่ (relative time)
export const formatRelativeTime = (date: Date | string | number): string => {
  const { t } = i18n;
  const dateObj = typeof date === 'string' || typeof date === 'number' 
    ? new Date(date) 
    : date;
  
  const now = new Date();
  const diffInMs = now.getTime() - dateObj.getTime();
  const diffInDays = Math.floor(diffInMs / (1000 * 60 * 60 * 24));

  if (diffInDays === 0) return t('dates.today');
  if (diffInDays === 1) return t('dates.yesterday');
  if (diffInDays === -1) return t('dates.tomorrow');
  if (diffInDays > 0) return t('dates.daysAgo', { count: diffInDays });
  
  return formatDate(dateObj);
};
```

### ตัวอย่างการใช้งาน Formatters

```typescript
import React from 'react';
import { View, Text } from 'react-native';
import { formatDate, formatCurrency, formatNumber, formatRelativeTime } from '../utils/formatters';
import { useTranslation } from 'react-i18next';

const FormattingExample: React.FC = () => {
  const { t } = useTranslation();
  const now = new Date();
  const yesterday = new Date(Date.now() - 86400000);
  
  return (
    <View style={{ padding: 20 }}>
      {/* วันที่ */}
      <Text>วันที่ปัจจุบัน: {formatDate(now)}</Text>
      
      {/* เวลาสัมพัทธ์ */}
      <Text>เมื่อวาน: {formatRelativeTime(yesterday)}</Text>
      
      {/* สกุลเงิน */}
      <Text>ราคา: {formatCurrency(1500, 'THB')}</Text>
      <Text>Price: {formatCurrency(49.99, 'USD')}</Text>
      
      {/* ตัวเลข */}
      <Text>จำนวน: {formatNumber(1234567.89)}</Text>
      
      {/* Plural */}
      <Text>{t('home.items', { count: 1 })}</Text>
      <Text>{t('home.items', { count: 5 })}</Text>
    </View>
  );
};
```

---

## 10. RTL Support

### การจัดการ RTL Layout

**src/utils/rtl.ts**:
```typescript
import { I18nManager, StyleSheet } from 'react-native';

// ตรวจสอบว่าเป็น RTL หรือไม่
export const isRTL = (): boolean => I18nManager.isRTL;

// สร้าง style ที่รองรับ RTL
export const rtlStyle = (ltrStyle: object, rtlOverride: object): object => {
  return I18nManager.isRTL ? { ...ltrStyle, ...rtlOverride } : ltrStyle;
};

// Flip margin/padding ตาม RTL
export const rtlFlipHorizontal = (value: number) => ({
  marginLeft: I18nManager.isRTL ? undefined : value,
  marginRight: I18nManager.isRTL ? value : undefined,
});

// ทิศทางสำหรับ Icon
export const rtlIconDirection = (): 'auto' | 'ltr' | 'rtl' => {
  return I18nManager.isRTL ? 'rtl' : 'ltr';
};

// Style helpers
export const rtlStyles = StyleSheet.create({
  textAlignAuto: {
    textAlign: I18nManager.isRTL ? 'right' : 'left',
  },
  flexRowAuto: {
    flexDirection: I18nManager.isRTL ? 'row-reverse' : 'row',
  },
  absoluteStart: {
    left: I18nManager.isRTL ? undefined : 0,
    right: I18nManager.isRTL ? 0 : undefined,
  },
  absoluteEnd: {
    left: I18nManager.isRTL ? 0 : undefined,
    right: I18nManager.isRTL ? undefined : 0,
  },
});
```

### RTL-Aware Component ตัวอย่าง

```typescript
import React from 'react';
import { View, Text, I18nManager, StyleSheet } from 'react-native';

interface ListItemProps {
  icon: string;
  title: string;
  subtitle?: string;
  rightElement?: React.ReactNode;
}

const RTLListItem: React.FC<ListItemProps> = ({ 
  icon, 
  title, 
  subtitle, 
  rightElement 
}) => {
  const isRTL = I18nManager.isRTL;
  
  return (
    <View style={[
      styles.container,
      { flexDirection: isRTL ? 'row-reverse' : 'row' }
    ]}>
      <Text style={styles.icon}>{icon}</Text>
      
      <View style={[
        styles.textContainer,
        { 
          marginLeft: isRTL ? 0 : 12,
          marginRight: isRTL ? 12 : 0,
        }
      ]}>
        <Text style={[
          styles.title,
          { textAlign: isRTL ? 'right' : 'left' }
        ]}>
          {title}
        </Text>
        {subtitle && (
          <Text style={[
            styles.subtitle,
            { textAlign: isRTL ? 'right' : 'left' }
          ]}>
            {subtitle}
          </Text>
        )}
      </View>
      
      {rightElement && (
        <View style={styles.rightElement}>
          {rightElement}
        </View>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    borderRadius: 8,
    marginBottom: 8,
  },
  icon: {
    fontSize: 24,
  },
  textContainer: {
    flex: 1,
    marginLeft: 12,
  },
  title: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
  },
  subtitle: {
    fontSize: 14,
    color: '#666',
    marginTop: 2,
  },
  rightElement: {
    marginLeft: 8,
  },
});
```

---

## 11. Pluralization ขั้นสูง

### การจัดการ Plural ในหลายภาษา

```typescript
// ใน translation file
{
  "message": {
    "unread_zero": "ไม่มีข้อความ",
    "unread_one": "{{count}} ข้อความที่ยังไม่ได้อ่าน",
    "unread_other": "{{count}} ข้อความที่ยังไม่ได้อ่าน"
  }
}

// การใช้งาน
const { t } = useTranslation();
t('message.unread', { count: 0 });  // "ไม่มีข้อความ"
t('message.unread', { count: 1 });  // "1 ข้อความที่ยังไม่ได้อ่าน"
t('message.unread', { count: 5 });  // "5 ข้อความที่ยังไม่ได้อ่าน"
```

---

## 12. Namespace Organization

### แยก Translation เป็น Namespaces

```typescript
// i18n configuration with multiple namespaces
i18n.init({
  resources: {
    en: {
      common: require('./translations/en/common.json'),
      auth: require('./translations/en/auth.json'),
      home: require('./translations/en/home.json'),
      errors: require('./translations/en/errors.json'),
    },
    th: {
      common: require('./translations/th/common.json'),
      auth: require('./translations/th/auth.json'),
      home: require('./translations/th/home.json'),
      errors: require('./translations/th/errors.json'),
    },
  },
  ns: ['common', 'auth', 'home', 'errors'],
  defaultNS: 'common',
});

// การใช้งาน
const { t } = useTranslation(['auth', 'common']);
t('auth:login');      // จาก namespace auth
t('common.cancel');   // จาก namespace common (default)
```

---

## Workshop: Multi-language App

### สร้าง Full Multi-language Application

**screens/SettingsScreen.tsx**:
```typescript
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  Switch,
  StyleSheet,
  Alert,
  ScrollView,
} from 'react-native';
import { useAppTranslation } from '../hooks/useAppTranslation';
import LanguageSwitcher from '../components/LanguageSwitcher';
import { formatDate } from '../utils/formatters';

const SettingsScreen: React.FC = () => {
  const { t, language, getCurrentLanguageInfo } = useAppTranslation();
  const [showLanguageSwitcher, setShowLanguageSwitcher] = useState(false);
  const [notificationsEnabled, setNotificationsEnabled] = useState(true);

  const currentLang = getCurrentLanguageInfo();

  const handleLogout = () => {
    Alert.alert(
      t('auth.logout'),
      t('common.confirm'),
      [
        { text: t('common.cancel'), style: 'cancel' },
        { text: t('common.confirm'), onPress: () => console.log('Logged out') },
      ]
    );
  };

  return (
    <ScrollView style={styles.container}>
      <Text style={styles.header}>{t('settings.title')}</Text>
      
      {/* Language Setting */}
      <TouchableOpacity 
        style={styles.settingItem}
        onPress={() => setShowLanguageSwitcher(true)}
      >
        <View style={styles.settingLeft}>
          <Text style={styles.settingIcon}>🌐</Text>
          <View>
            <Text style={styles.settingLabel}>{t('settings.language')}</Text>
            <Text style={styles.settingValue}>{currentLang?.nativeName}</Text>
          </View>
        </View>
        <Text style={styles.arrow}>›</Text>
      </TouchableOpacity>

      {/* Notifications Setting */}
      <View style={styles.settingItem}>
        <View style={styles.settingLeft}>
          <Text style={styles.settingIcon}>🔔</Text>
          <Text style={styles.settingLabel}>{t('settings.notifications')}</Text>
        </View>
        <Switch
          value={notificationsEnabled}
          onValueChange={setNotificationsEnabled}
        />
      </View>

      {/* Current Date Formatted */}
      <View style={styles.infoCard}>
        <Text style={styles.infoLabel}>Current Date:</Text>
        <Text style={styles.infoValue}>{formatDate(new Date())}</Text>
        <Text style={styles.infoLabel}>Language Code:</Text>
        <Text style={styles.infoValue}>{language}</Text>
      </View>

      {/* Logout Button */}
      <TouchableOpacity style={styles.logoutButton} onPress={handleLogout}>
        <Text style={styles.logoutText}>{t('auth.logout')}</Text>
      </TouchableOpacity>

      <LanguageSwitcher
        visible={showLanguageSwitcher}
        onClose={() => setShowLanguageSwitcher(false)}
      />
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    fontSize: 24,
    fontWeight: 'bold',
    padding: 20,
    color: '#333',
  },
  settingItem: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    backgroundColor: '#fff',
    padding: 16,
    marginBottom: 1,
  },
  settingLeft: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  settingIcon: {
    fontSize: 24,
    marginRight: 12,
  },
  settingLabel: {
    fontSize: 16,
    color: '#333',
  },
  settingValue: {
    fontSize: 14,
    color: '#666',
    marginTop: 2,
  },
  arrow: {
    fontSize: 20,
    color: '#ccc',
  },
  infoCard: {
    backgroundColor: '#fff',
    margin: 16,
    padding: 16,
    borderRadius: 8,
  },
  infoLabel: {
    fontSize: 14,
    color: '#666',
    marginBottom: 4,
  },
  infoValue: {
    fontSize: 16,
    fontWeight: '600',
    color: '#333',
    marginBottom: 12,
  },
  logoutButton: {
    margin: 16,
    padding: 16,
    backgroundColor: '#f44336',
    borderRadius: 8,
    alignItems: 'center',
  },
  logoutText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: 'bold',
  },
});

export default SettingsScreen;
```

---

## Tips และ Best Practices

### 1. ใช้ Constants สำหรับ Translation Keys

```typescript
// constants/translationKeys.ts
export const T_KEYS = {
  COMMON: {
    WELCOME: 'common.welcome',
    LOADING: 'common.loading',
    ERROR: 'common.error',
  },
  AUTH: {
    LOGIN: 'auth.login',
    LOGOUT: 'auth.logout',
  },
} as const;

// การใช้งาน
t(T_KEYS.AUTH.LOGIN);
```

### 2. Lazy Loading Translations

```typescript
// สำหรับแอปที่มีหลาย locale ขนาดใหญ่
const loadTranslations = async (lang: string) => {
  const translations = await import(`./translations/${lang}.json`);
  i18n.addResourceBundle(lang, 'translation', translations.default, true, true);
};
```

### 3. ทดสอบ i18n

```typescript
// __tests__/i18n.test.ts
import i18n from '../src/i18n';

describe('i18n', () => {
  beforeAll(() => {
    return i18n.init();
  });

  it('should translate English correctly', () => {
    i18n.changeLanguage('en');
    expect(i18n.t('common.welcome')).toBe('Welcome');
  });

  it('should translate Thai correctly', async () => {
    await i18n.changeLanguage('th');
    expect(i18n.t('common.welcome')).toBe('ยินดีต้อนรับ');
  });

  it('should handle interpolation', () => {
    i18n.changeLanguage('en');
    expect(i18n.t('common.hello', { name: 'John' })).toBe('Hello, John!');
  });

  it('should handle plural correctly', () => {
    i18n.changeLanguage('en');
    expect(i18n.t('home.items', { count: 1 })).toBe('1 item');
    expect(i18n.t('home.items', { count: 5 })).toBe('5 items');
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **react-i18next**: การตั้งค่าและใช้งาน library สำหรับ i18n
2. **Translation Files**: การจัดการไฟล์แปลภาษาแบบ JSON
3. **Language Switching**: การเปลี่ยนภาษาแบบ Dynamic
4. **Date/Number Formatting**: การ format ข้อมูลตาม locale
5. **RTL Support**: การรองรับภาษา Right-to-Left
6. **Best Practices**: เทคนิคสำหรับโปรเจคขนาดใหญ่

### แบบฝึกหัด

1. เพิ่มการรองรับภาษาเกาหลี (ko)
2. สร้าง component ที่แสดงวันเกิดตาม locale ที่เลือก
3. ทำการทดสอบ unit test สำหรับ i18n hooks
4. เพิ่ม namespace แยกสำหรับแต่ละ screen
