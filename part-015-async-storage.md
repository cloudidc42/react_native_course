# Part 015: AsyncStorage - จัดเก็บข้อมูล

## สารบัญ
1. AsyncStorage คืออะไร
2. setItem และ getItem
3. removeItem
4. multiSet และ multiGet
5. getAllKeys และ clear
6. Error Handling
7. Workshop: Settings Screen with Persistence

---

## 1. AsyncStorage คืออะไร

**AsyncStorage** คือ key-value storage system ที่ทำงานแบบ asynchronous (ไม่บล็อก main thread) สำหรับเก็บข้อมูลบน device ของผู้ใช้

### การติดตั้ง

```bash
npm install @react-native-async-storage/async-storage
# หรือสำหรับ Expo
npx expo install @react-native-async-storage/async-storage
```

### การ import

```javascript
import AsyncStorage from '@react-native-async-storage/async-storage';
```

### เมื่อไหร่ควรใช้ AsyncStorage

| ใช้ AsyncStorage | ไม่ควรใช้ AsyncStorage |
|-----------------|----------------------|
| User preferences | ข้อมูลจำนวนมาก (>6MB) |
| Theme settings | ข้อมูล sensitive (รหัสผ่าน) |
| Language settings | Binary data (ใช้ expo-file-system) |
| Last opened screen | ข้อมูลที่ต้องการ search ซับซ้อน |
| JWT Token cache | Database-like operations |
| Shopping cart items | |

### ข้อจำกัด

- ขนาดสูงสุด: ขึ้นกับ OS (Android ~6MB, iOS ไม่จำกัด)
- เก็บได้เฉพาะ string เท่านั้น (ต้อง JSON.stringify/parse สำหรับ object)
- Asynchronous เท่านั้น ไม่มี synchronous version
- ไม่ encrypt data (ใช้ react-native-encrypted-storage สำหรับข้อมูล sensitive)

---

## 2. setItem และ getItem

### setItem - บันทึกข้อมูล

```javascript
// บันทึก string
const storeData = async (value) => {
  try {
    await AsyncStorage.setItem('@storage_key', value);
    console.log('บันทึกสำเร็จ');
  } catch (error) {
    console.error('เกิดข้อผิดพลาด:', error);
  }
};

// บันทึก object (ต้อง stringify)
const storeObject = async (key, value) => {
  try {
    const jsonValue = JSON.stringify(value);
    await AsyncStorage.setItem(key, jsonValue);
  } catch (error) {
    console.error('Error saving:', error);
  }
};

// ตัวอย่างการใช้งาน
await storeObject('@user_settings', {
  theme: 'dark',
  language: 'th',
  notifications: true,
  fontSize: 16,
});
```

### getItem - ดึงข้อมูล

```javascript
// ดึง string
const getData = async (key) => {
  try {
    const value = await AsyncStorage.getItem(key);
    if (value !== null) {
      return value;
    }
    return null;  // ไม่พบ key
  } catch (error) {
    console.error('Error reading:', error);
    return null;
  }
};

// ดึง object
const getObject = async (key) => {
  try {
    const jsonValue = await AsyncStorage.getItem(key);
    return jsonValue != null ? JSON.parse(jsonValue) : null;
  } catch (error) {
    console.error('Error reading:', error);
    return null;
  }
};

// ตัวอย่าง
const settings = await getObject('@user_settings');
if (settings) {
  console.log('Theme:', settings.theme);
}
```

### ใช้งานใน Component

```javascript
import React, { useState, useEffect } from 'react';
import { View, Text, Switch, StyleSheet } from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';

function SettingsScreen() {
  const [notifications, setNotifications] = useState(false);
  const [darkMode, setDarkMode] = useState(false);
  const [loading, setLoading] = useState(true);

  // โหลดการตั้งค่าเมื่อ component mount
  useEffect(() => {
    loadSettings();
  }, []);

  const loadSettings = async () => {
    try {
      const savedSettings = await AsyncStorage.getItem('@settings');
      if (savedSettings !== null) {
        const { notifications: n, darkMode: d } = JSON.parse(savedSettings);
        setNotifications(n);
        setDarkMode(d);
      }
    } catch (error) {
      console.error('Load error:', error);
    } finally {
      setLoading(false);
    }
  };

  const saveSettings = async (newSettings) => {
    try {
      await AsyncStorage.setItem('@settings', JSON.stringify(newSettings));
    } catch (error) {
      console.error('Save error:', error);
    }
  };

  const handleNotificationsChange = (value) => {
    setNotifications(value);
    saveSettings({ notifications: value, darkMode });
  };

  const handleDarkModeChange = (value) => {
    setDarkMode(value);
    saveSettings({ notifications, darkMode: value });
  };

  if (loading) return <Text>กำลังโหลด...</Text>;

  return (
    <View style={styles.container}>
      <View style={styles.row}>
        <Text style={styles.label}>การแจ้งเตือน</Text>
        <Switch value={notifications} onValueChange={handleNotificationsChange} />
      </View>
      <View style={styles.row}>
        <Text style={styles.label}>Dark Mode</Text>
        <Switch value={darkMode} onValueChange={handleDarkModeChange} />
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#EEE',
  },
  label: { fontSize: 16 },
});
```

---

## 3. removeItem

```javascript
// ลบ key เดียว
const removeValue = async (key) => {
  try {
    await AsyncStorage.removeItem(key);
    console.log('ลบสำเร็จ');
  } catch (error) {
    console.error('Error removing:', error);
  }
};

// ตัวอย่าง: ลบ token เมื่อ logout
const handleLogout = async () => {
  await AsyncStorage.removeItem('@auth_token');
  await AsyncStorage.removeItem('@user_data');
  navigation.reset({
    index: 0,
    routes: [{ name: 'Login' }],
  });
};
```

---

## 4. multiSet และ multiGet

### multiSet - บันทึกหลายค่าพร้อมกัน

```javascript
// บันทึกหลายค่าในครั้งเดียว (มีประสิทธิภาพมากกว่าการเรียก setItem หลายครั้ง)
const multiSave = async () => {
  try {
    const pairs = [
      ['@theme', 'dark'],
      ['@language', 'th'],
      ['@font_size', '16'],
      ['@notifications', 'true'],
    ];
    await AsyncStorage.multiSet(pairs);
    console.log('บันทึกทั้งหมดสำเร็จ');
  } catch (error) {
    console.error('Error:', error);
  }
};

// สำหรับ object values
const saveUserData = async (userData, settings) => {
  try {
    const pairs = [
      ['@user_data', JSON.stringify(userData)],
      ['@user_settings', JSON.stringify(settings)],
      ['@last_login', new Date().toISOString()],
    ];
    await AsyncStorage.multiSet(pairs);
  } catch (error) {
    throw error;
  }
};
```

### multiGet - ดึงหลายค่าพร้อมกัน

```javascript
// ดึงหลายค่าในครั้งเดียว
const multiLoad = async () => {
  try {
    const keys = ['@theme', '@language', '@font_size', '@notifications'];
    const values = await AsyncStorage.multiGet(keys);
    
    // values เป็น array ของ [key, value] pairs
    // [['@theme', 'dark'], ['@language', 'th'], ...]
    
    const result = {};
    values.forEach(([key, value]) => {
      result[key] = value;
    });
    
    return result;
  } catch (error) {
    console.error('Error:', error);
    return {};
  }
};

// ใช้งาน
const settings = await multiLoad();
console.log('Theme:', settings['@theme']);
```

### multiRemove

```javascript
const multiDelete = async () => {
  try {
    const keys = ['@temp_data', '@cache', '@session'];
    await AsyncStorage.multiRemove(keys);
    console.log('ลบทั้งหมดสำเร็จ');
  } catch (error) {
    console.error('Error:', error);
  }
};
```

---

## 5. getAllKeys และ clear

### getAllKeys - ดึง keys ทั้งหมด

```javascript
const getAllStoredKeys = async () => {
  try {
    const keys = await AsyncStorage.getAllKeys();
    console.log('All keys:', keys);
    // ['@theme', '@language', '@user_data', '@token', ...]
    return keys;
  } catch (error) {
    console.error('Error:', error);
    return [];
  }
};

// ดูข้อมูลทั้งหมดที่เก็บ (debug)
const debugStorage = async () => {
  const keys = await AsyncStorage.getAllKeys();
  const values = await AsyncStorage.multiGet(keys);
  
  console.log('=== AsyncStorage Contents ===');
  values.forEach(([key, value]) => {
    console.log(`${key}: ${value}`);
  });
};
```

### clear - ล้างข้อมูลทั้งหมด

```javascript
// ⚠️ ระวัง: ลบ ALL data ทั้งหมด
const clearAll = async () => {
  try {
    await AsyncStorage.clear();
    console.log('ล้างข้อมูลทั้งหมดแล้ว');
  } catch (error) {
    console.error('Error:', error);
  }
};

// วิธีที่ปลอดภัยกว่า: ลบเฉพาะ keys ของแอปเรา
const clearAppData = async () => {
  try {
    const allKeys = await AsyncStorage.getAllKeys();
    // กรองเฉพาะ keys ที่ขึ้นต้นด้วย @myapp
    const appKeys = allKeys.filter(key => key.startsWith('@myapp'));
    await AsyncStorage.multiRemove(appKeys);
  } catch (error) {
    console.error('Error:', error);
  }
};
```

---

## 6. Error Handling

### Pattern การจัดการ Error

```javascript
// Helper function สำหรับ AsyncStorage operations
const storage = {
  async set(key, value) {
    try {
      const jsonValue = typeof value === 'string' ? value : JSON.stringify(value);
      await AsyncStorage.setItem(key, jsonValue);
      return { success: true };
    } catch (error) {
      console.error(`AsyncStorage.set error for key "${key}":`, error);
      return { success: false, error };
    }
  },

  async get(key, defaultValue = null) {
    try {
      const value = await AsyncStorage.getItem(key);
      if (value === null) return defaultValue;
      
      try {
        return JSON.parse(value);
      } catch {
        return value;  // Return as string if not valid JSON
      }
    } catch (error) {
      console.error(`AsyncStorage.get error for key "${key}":`, error);
      return defaultValue;
    }
  },

  async remove(key) {
    try {
      await AsyncStorage.removeItem(key);
      return { success: true };
    } catch (error) {
      console.error(`AsyncStorage.remove error for key "${key}":`, error);
      return { success: false, error };
    }
  },

  async getMultiple(keys) {
    try {
      const pairs = await AsyncStorage.multiGet(keys);
      const result = {};
      pairs.forEach(([key, value]) => {
        if (value !== null) {
          try {
            result[key] = JSON.parse(value);
          } catch {
            result[key] = value;
          }
        }
      });
      return result;
    } catch (error) {
      console.error('AsyncStorage.getMultiple error:', error);
      return {};
    }
  },
};

// ใช้งาน
const { success } = await storage.set('@user', { name: 'สมชาย', age: 30 });
const user = await storage.get('@user', { name: 'Guest' });
```

---

## Workshop: Settings Screen with Persistence

### โครงสร้างแอป

```
SettingsApp/
├── App.js
├── contexts/
│   └── SettingsContext.js
├── hooks/
│   └── useSettings.js
├── screens/
│   ├── HomeScreen.js
│   └── SettingsScreen.js
└── utils/
    └── storage.js
```

### storage.js utility

```javascript
// utils/storage.js
import AsyncStorage from '@react-native-async-storage/async-storage';

const KEYS = {
  SETTINGS: '@app_settings',
  USER_PROFILE: '@user_profile',
  RECENT_SEARCHES: '@recent_searches',
  ONBOARDING_DONE: '@onboarding_done',
};

export const StorageKeys = KEYS;

export const StorageService = {
  // Settings
  async getSettings() {
    try {
      const data = await AsyncStorage.getItem(KEYS.SETTINGS);
      if (!data) return null;
      return JSON.parse(data);
    } catch {
      return null;
    }
  },

  async saveSettings(settings) {
    try {
      await AsyncStorage.setItem(KEYS.SETTINGS, JSON.stringify(settings));
      return true;
    } catch {
      return false;
    }
  },

  // User Profile
  async getUserProfile() {
    try {
      const data = await AsyncStorage.getItem(KEYS.USER_PROFILE);
      return data ? JSON.parse(data) : null;
    } catch {
      return null;
    }
  },

  async saveUserProfile(profile) {
    try {
      await AsyncStorage.setItem(KEYS.USER_PROFILE, JSON.stringify(profile));
      return true;
    } catch {
      return false;
    }
  },

  // Recent Searches
  async getRecentSearches() {
    try {
      const data = await AsyncStorage.getItem(KEYS.RECENT_SEARCHES);
      return data ? JSON.parse(data) : [];
    } catch {
      return [];
    }
  },

  async addRecentSearch(query) {
    try {
      const searches = await this.getRecentSearches();
      const filtered = searches.filter(s => s !== query);
      const updated = [query, ...filtered].slice(0, 10);
      await AsyncStorage.setItem(KEYS.RECENT_SEARCHES, JSON.stringify(updated));
      return updated;
    } catch {
      return [];
    }
  },

  async clearRecentSearches() {
    try {
      await AsyncStorage.removeItem(KEYS.RECENT_SEARCHES);
      return true;
    } catch {
      return false;
    }
  },

  // Onboarding
  async isOnboardingDone() {
    try {
      const value = await AsyncStorage.getItem(KEYS.ONBOARDING_DONE);
      return value === 'true';
    } catch {
      return false;
    }
  },

  async markOnboardingDone() {
    try {
      await AsyncStorage.setItem(KEYS.ONBOARDING_DONE, 'true');
      return true;
    } catch {
      return false;
    }
  },

  // Clear all app data
  async clearAll() {
    try {
      const keys = Object.values(KEYS);
      await AsyncStorage.multiRemove(keys);
      return true;
    } catch {
      return false;
    }
  },
};
```

### Settings Context

```javascript
// contexts/SettingsContext.js
import React, { createContext, useContext, useState, useEffect } from 'react';
import { StorageService } from '../utils/storage';

const defaultSettings = {
  theme: 'light',         // 'light' | 'dark' | 'system'
  language: 'th',         // 'th' | 'en'
  fontSize: 'medium',     // 'small' | 'medium' | 'large'
  notifications: true,
  emailNotifications: false,
  soundEnabled: true,
  hapticFeedback: true,
  dataUsage: 'wifi_only', // 'always' | 'wifi_only'
  autoPlay: false,
  privacyMode: false,
};

const SettingsContext = createContext(null);

export function SettingsProvider({ children }) {
  const [settings, setSettings] = useState(defaultSettings);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadSettings();
  }, []);

  const loadSettings = async () => {
    try {
      const saved = await StorageService.getSettings();
      if (saved) {
        setSettings({ ...defaultSettings, ...saved });
      }
    } catch (error) {
      console.error('Failed to load settings:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateSetting = async (key, value) => {
    const newSettings = { ...settings, [key]: value };
    setSettings(newSettings);
    await StorageService.saveSettings(newSettings);
  };

  const updateSettings = async (updates) => {
    const newSettings = { ...settings, ...updates };
    setSettings(newSettings);
    await StorageService.saveSettings(newSettings);
  };

  const resetSettings = async () => {
    setSettings(defaultSettings);
    await StorageService.saveSettings(defaultSettings);
  };

  return (
    <SettingsContext.Provider value={{
      settings,
      loading,
      updateSetting,
      updateSettings,
      resetSettings,
    }}>
      {children}
    </SettingsContext.Provider>
  );
}

export const useSettings = () => {
  const context = useContext(SettingsContext);
  if (!context) throw new Error('useSettings must be inside SettingsProvider');
  return context;
};
```

### SettingsScreen

```javascript
// screens/SettingsScreen.js
import React, { useState } from 'react';
import {
  View, Text, StyleSheet, ScrollView,
  Switch, TouchableOpacity, Alert, Modal,
  ActivityIndicator
} from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { useSettings } from '../contexts/SettingsContext';

function SettingSection({ title, children }) {
  return (
    <View style={styles.section}>
      <Text style={styles.sectionTitle}>{title}</Text>
      <View style={styles.sectionContent}>
        {children}
      </View>
    </View>
  );
}

function SettingRow({ icon, title, subtitle, children, onPress, dangerous }) {
  const Wrapper = onPress ? TouchableOpacity : View;
  
  return (
    <Wrapper
      style={styles.row}
      onPress={onPress}
      activeOpacity={0.7}
    >
      <View style={[styles.rowIcon, dangerous && styles.dangerousIcon]}>
        <Ionicons 
          name={icon} 
          size={20} 
          color={dangerous ? '#E53935' : '#6200EE'} 
        />
      </View>
      <View style={styles.rowText}>
        <Text style={[styles.rowTitle, dangerous && styles.dangerousText]}>
          {title}
        </Text>
        {subtitle && <Text style={styles.rowSubtitle}>{subtitle}</Text>}
      </View>
      {children}
    </Wrapper>
  );
}

function ThemeModal({ visible, current, onSelect, onClose }) {
  const options = [
    { value: 'light', label: 'สว่าง', icon: 'sunny' },
    { value: 'dark', label: 'มืด', icon: 'moon' },
    { value: 'system', label: 'ตามระบบ', icon: 'phone-portrait' },
  ];

  return (
    <Modal visible={visible} transparent animationType="slide">
      <View style={styles.modalOverlay}>
        <View style={styles.modal}>
          <Text style={styles.modalTitle}>เลือก Theme</Text>
          {options.map(opt => (
            <TouchableOpacity
              key={opt.value}
              style={styles.modalOption}
              onPress={() => { onSelect(opt.value); onClose(); }}
            >
              <Ionicons name={opt.icon} size={22} color="#6200EE" />
              <Text style={styles.modalOptionText}>{opt.label}</Text>
              {current === opt.value && (
                <Ionicons name="checkmark" size={22} color="#6200EE" style={{ marginLeft: 'auto' }} />
              )}
            </TouchableOpacity>
          ))}
          <TouchableOpacity style={styles.modalCancel} onPress={onClose}>
            <Text style={styles.modalCancelText}>ยกเลิก</Text>
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );
}

export default function SettingsScreen() {
  const { settings, loading, updateSetting, resetSettings } = useSettings();
  const [themeModalVisible, setThemeModalVisible] = useState(false);
  const [saving, setSaving] = useState(false);

  const themeLabels = { light: 'สว่าง', dark: 'มืด', system: 'ตามระบบ' };
  const fontSizeLabels = { small: 'เล็ก', medium: 'กลาง', large: 'ใหญ่' };

  const handleResetSettings = () => {
    Alert.alert(
      'รีเซ็ตการตั้งค่า',
      'คุณต้องการรีเซ็ตการตั้งค่าทั้งหมดหรือไม่?',
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'รีเซ็ต',
          style: 'destructive',
          onPress: async () => {
            setSaving(true);
            await resetSettings();
            setSaving(false);
            Alert.alert('สำเร็จ', 'รีเซ็ตการตั้งค่าแล้ว');
          },
        },
      ]
    );
  };

  if (loading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
        <Text style={styles.loadingText}>กำลังโหลดการตั้งค่า...</Text>
      </View>
    );
  }

  return (
    <>
      <ScrollView style={styles.container}>
        {/* การแสดงผล */}
        <SettingSection title="การแสดงผล">
          <SettingRow
            icon="color-palette-outline"
            title="ธีม"
            subtitle={themeLabels[settings.theme]}
            onPress={() => setThemeModalVisible(true)}
          >
            <Ionicons name="chevron-forward" size={18} color="#CCC" />
          </SettingRow>

          <SettingRow
            icon="text"
            title="ขนาดตัวอักษร"
            subtitle={fontSizeLabels[settings.fontSize]}
          >
            <View style={styles.fontSizeButtons}>
              {['small', 'medium', 'large'].map(size => (
                <TouchableOpacity
                  key={size}
                  style={[
                    styles.fontBtn,
                    settings.fontSize === size && styles.activeFontBtn
                  ]}
                  onPress={() => updateSetting('fontSize', size)}
                >
                  <Text style={[
                    styles.fontBtnText,
                    { fontSize: size === 'small' ? 11 : size === 'medium' ? 14 : 17 },
                    settings.fontSize === size && styles.activeFontBtnText
                  ]}>
                    ก
                  </Text>
                </TouchableOpacity>
              ))}
            </View>
          </SettingRow>
        </SettingSection>

        {/* การแจ้งเตือน */}
        <SettingSection title="การแจ้งเตือน">
          <SettingRow
            icon="notifications-outline"
            title="การแจ้งเตือน"
            subtitle="รับการแจ้งเตือนจากแอป"
          >
            <Switch
              value={settings.notifications}
              onValueChange={(v) => updateSetting('notifications', v)}
              trackColor={{ false: '#DDD', true: '#6200EE' }}
            />
          </SettingRow>

          <SettingRow
            icon="mail-outline"
            title="อีเมลแจ้งเตือน"
            subtitle="รับอีเมลสรุปรายสัปดาห์"
          >
            <Switch
              value={settings.emailNotifications}
              onValueChange={(v) => updateSetting('emailNotifications', v)}
              trackColor={{ false: '#DDD', true: '#6200EE' }}
              disabled={!settings.notifications}
            />
          </SettingRow>
        </SettingSection>

        {/* เสียงและการสัมผัส */}
        <SettingSection title="เสียงและการสัมผัส">
          <SettingRow
            icon="volume-medium-outline"
            title="เสียง"
          >
            <Switch
              value={settings.soundEnabled}
              onValueChange={(v) => updateSetting('soundEnabled', v)}
              trackColor={{ false: '#DDD', true: '#6200EE' }}
            />
          </SettingRow>

          <SettingRow
            icon="phone-portrait-outline"
            title="Haptic Feedback"
            subtitle="สั่นเมื่อกดปุ่ม"
          >
            <Switch
              value={settings.hapticFeedback}
              onValueChange={(v) => updateSetting('hapticFeedback', v)}
              trackColor={{ false: '#DDD', true: '#6200EE' }}
            />
          </SettingRow>
        </SettingSection>

        {/* ความเป็นส่วนตัว */}
        <SettingSection title="ความเป็นส่วนตัว">
          <SettingRow
            icon="eye-off-outline"
            title="Privacy Mode"
            subtitle="ซ่อนข้อมูลส่วนตัว"
          >
            <Switch
              value={settings.privacyMode}
              onValueChange={(v) => updateSetting('privacyMode', v)}
              trackColor={{ false: '#DDD', true: '#6200EE' }}
            />
          </SettingRow>
        </SettingSection>

        {/* การรีเซ็ต */}
        <SettingSection title="">
          <SettingRow
            icon="refresh-outline"
            title="รีเซ็ตการตั้งค่า"
            subtitle="คืนค่าเริ่มต้นทั้งหมด"
            onPress={handleResetSettings}
            dangerous
          />
        </SettingSection>

        <View style={{ height: 40 }} />
      </ScrollView>

      {/* Theme Modal */}
      <ThemeModal
        visible={themeModalVisible}
        current={settings.theme}
        onSelect={(theme) => updateSetting('theme', theme)}
        onClose={() => setThemeModalVisible(false)}
      />

      {/* Saving Indicator */}
      {saving && (
        <View style={styles.savingOverlay}>
          <ActivityIndicator color="#FFF" />
          <Text style={styles.savingText}>กำลังบันทึก...</Text>
        </View>
      )}
    </>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  loadingText: { marginTop: 12, color: '#666' },
  section: { marginBottom: 8 },
  sectionTitle: {
    fontSize: 13,
    fontWeight: '600',
    color: '#888',
    paddingHorizontal: 16,
    paddingTop: 20,
    paddingBottom: 8,
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
  sectionContent: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    marginHorizontal: 16,
    overflow: 'hidden',
  },
  row: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 14,
    borderBottomWidth: 1,
    borderBottomColor: '#F5F5F5',
  },
  rowIcon: {
    width: 36,
    height: 36,
    borderRadius: 8,
    backgroundColor: '#EDE7F6',
    alignItems: 'center',
    justifyContent: 'center',
    marginRight: 12,
  },
  dangerousIcon: { backgroundColor: '#FFEBEE' },
  rowText: { flex: 1 },
  rowTitle: { fontSize: 15, color: '#212121' },
  dangerousText: { color: '#E53935' },
  rowSubtitle: { fontSize: 12, color: '#999', marginTop: 2 },
  fontSizeButtons: { flexDirection: 'row', gap: 6 },
  fontBtn: {
    width: 36,
    height: 36,
    borderRadius: 8,
    backgroundColor: '#F5F5F5',
    alignItems: 'center',
    justifyContent: 'center',
    borderWidth: 1,
    borderColor: '#EEE',
  },
  activeFontBtn: { backgroundColor: '#6200EE', borderColor: '#6200EE' },
  fontBtnText: { color: '#555' },
  activeFontBtnText: { color: '#FFF' },
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'flex-end',
  },
  modal: {
    backgroundColor: '#FFF',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    padding: 20,
    paddingBottom: 40,
  },
  modalTitle: { fontSize: 18, fontWeight: 'bold', marginBottom: 16, textAlign: 'center' },
  modalOption: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 14,
    borderRadius: 10,
    marginBottom: 4,
  },
  modalOptionText: { fontSize: 16, marginLeft: 12 },
  modalCancel: {
    backgroundColor: '#F5F5F5',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginTop: 8,
  },
  modalCancelText: { fontSize: 16, color: '#E53935' },
  savingOverlay: {
    position: 'absolute',
    bottom: 40,
    left: '50%',
    transform: [{ translateX: -70 }],
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: 'rgba(0,0,0,0.8)',
    paddingHorizontal: 20,
    paddingVertical: 10,
    borderRadius: 20,
  },
  savingText: { color: '#FFF', marginLeft: 8 },
});
```

### App.js

```javascript
// App.js
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { SettingsProvider } from './contexts/SettingsContext';
import HomeScreen from './screens/HomeScreen';
import SettingsScreen from './screens/SettingsScreen';

const Stack = createNativeStackNavigator();

export default function App() {
  return (
    <SettingsProvider>
      <NavigationContainer>
        <Stack.Navigator screenOptions={{
          headerStyle: { backgroundColor: '#6200EE' },
          headerTintColor: '#FFF',
        }}>
          <Stack.Screen name="Home" component={HomeScreen} options={{ title: 'แอปตัวอย่าง' }} />
          <Stack.Screen name="Settings" component={SettingsScreen} options={{ title: 'ตั้งค่า' }} />
        </Stack.Navigator>
      </NavigationContainer>
    </SettingsProvider>
  );
}
```

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ prefix เช่น '@myapp_' สำหรับทุก key
- สร้าง constants สำหรับ key names
- สร้าง utility/service layer สำหรับ AsyncStorage
- Handle null values เสมอ (getItem คืน null ถ้าไม่พบ)
- ใช้ multiGet/multiSet เมื่อต้องการดึง/บันทึกหลายค่า

❌ DON'T:
- ไม่เก็บข้อมูล sensitive เช่น password, credit card
- ไม่เก็บข้อมูลขนาดใหญ่ (>100KB)
- ไม่เรียก AsyncStorage ใน render phase (ใช้ useEffect)
- ไม่ลืม try/catch เพราะ AsyncStorage อาจ fail

🔒 ถ้าต้องการเก็บ sensitive data:
npm install react-native-encrypted-storage
```

---

## สรุป

AsyncStorage เป็นเครื่องมือสำคัญสำหรับ persistence ใน React Native:

1. **setItem/getItem** - บันทึกและดึงข้อมูล
2. **removeItem** - ลบข้อมูล
3. **multiSet/multiGet** - จัดการหลายค่าพร้อมกัน
4. **getAllKeys/clear** - จัดการทุก key
5. **Error Handling** - จัดการ error อย่างถูกต้อง
6. **Workshop** - Settings screen ที่บันทึกการตั้งค่าได้จริง
