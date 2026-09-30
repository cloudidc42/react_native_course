# Part 027: Platform-specific Code

## บทนำ

React Native ช่วยให้เราเขียน code ครั้งเดียวใช้ได้ทั้ง iOS และ Android แต่บางครั้งเราต้องการพฤติกรรมหรือ UI ที่แตกต่างกันตาม platform เพื่อให้ app ดู native บนแต่ละ platform ในบทนี้เราจะเรียนรู้วิธีต่างๆ ในการจัดการ platform-specific code

## สารบัญ

1. Platform.OS
2. Platform.select()
3. Platform-specific Files (.ios.js, .android.js)
4. StatusBar (Platform Differences)
5. Safe Areas
6. Workshop: Platform Adaptive UI

---

## 1. Platform.OS

`Platform.OS` ให้ค่า string ของ platform ปัจจุบัน: `'ios'`, `'android'`, `'web'`, `'windows'`, `'macos'`

```jsx
import React from 'react';
import { Platform, View, Text, StyleSheet } from 'react-native';

// ตรวจสอบ platform
const PlatformCheck = () => {
  console.log('Platform:', Platform.OS);
  console.log('Version:', Platform.Version);  // iOS: number, Android: number
  console.log('Is iOS 14+:', Platform.OS === 'ios' && Platform.Version >= 14);

  return (
    <View style={styles.container}>
      <Text style={styles.label}>Platform</Text>
      <Text style={styles.value}>{Platform.OS}</Text>

      <Text style={styles.label}>Version</Text>
      <Text style={styles.value}>{Platform.Version}</Text>

      {Platform.OS === 'ios' && (
        <View style={styles.iosBadge}>
          <Text style={styles.badgeText}>🍎 iOS {Platform.Version}</Text>
        </View>
      )}

      {Platform.OS === 'android' && (
        <View style={styles.androidBadge}>
          <Text style={styles.badgeText}>🤖 Android API {Platform.Version}</Text>
        </View>
      )}
    </View>
  );
};

// Conditional rendering
const PlatformSpecificContent = () => {
  if (Platform.OS === 'ios') {
    return (
      <View style={{ padding: 20 }}>
        <Text>iOS-only feature</Text>
        {/* Face ID, Apple Pay, etc. */}
      </View>
    );
  }

  if (Platform.OS === 'android') {
    return (
      <View style={{ padding: 20 }}>
        <Text>Android-only feature</Text>
        {/* Back button, Fingerprint, etc. */}
      </View>
    );
  }

  return null;
};

// Style แยกตาม platform
const PlatformStyles = () => {
  return (
    <View style={[
      styles.base,
      Platform.OS === 'ios' ? styles.iosStyle : styles.androidStyle,
    ]}>
      <Text>Platform-specific styling</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, gap: 8 },
  label: { fontSize: 12, color: '#666', textTransform: 'uppercase', letterSpacing: 1 },
  value: { fontSize: 20, fontWeight: 'bold', marginBottom: 8 },
  iosBadge: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    alignSelf: 'flex-start',
  },
  androidBadge: {
    backgroundColor: '#34A853',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    alignSelf: 'flex-start',
  },
  badgeText: { color: '#fff', fontWeight: '600' },
  base: { padding: 16, borderRadius: 8 },
  iosStyle: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    backgroundColor: '#fff',
  },
  androidStyle: {
    elevation: 4,
    backgroundColor: '#fff',
  },
});
```

### Platform Version Checks

```jsx
import { Platform } from 'react-native';

// iOS Version Checks
const isIOS14Plus = Platform.OS === 'ios' && Platform.Version >= 14;
const isIOS15Plus = Platform.OS === 'ios' && Platform.Version >= 15;
const isIOS16Plus = Platform.OS === 'ios' && Platform.Version >= 16;

// Android API Level Checks
const isAndroid10Plus = Platform.OS === 'android' && Platform.Version >= 29;
const isAndroid12Plus = Platform.OS === 'android' && Platform.Version >= 31;
const isAndroid13Plus = Platform.OS === 'android' && Platform.Version >= 33;

// ตัวอย่างการใช้
const BlurEffect = () => {
  if (Platform.OS === 'ios' && Platform.Version >= 14) {
    // ใช้ BlurView
    return <BlurView intensity={50} style={{ flex: 1 }} />;
  }
  // Fallback สำหรับ version เก่า
  return <View style={{ flex: 1, backgroundColor: 'rgba(255,255,255,0.8)' }} />;
};
```

---

## 2. Platform.select()

`Platform.select()` ให้ค่าที่แตกต่างกันตาม platform ในรูปแบบ object

```jsx
import { Platform } from 'react-native';

// Pattern พื้นฐาน
const value = Platform.select({
  ios: 'iOS value',
  android: 'Android value',
  default: 'Other platform',
});

// ใช้กับ Styles
const ThemedStyles = StyleSheet.create({
  container: {
    backgroundColor: '#fff',
    ...Platform.select({
      ios: {
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 8,
      },
      android: {
        elevation: 4,
      },
    }),
  },

  header: {
    backgroundColor: '#007AFF',
    paddingTop: Platform.select({
      ios: 44,
      android: 0,
    }),
    height: Platform.select({
      ios: 88,
      android: 56,
    }),
  },

  text: {
    fontFamily: Platform.select({
      ios: 'San Francisco',
      android: 'Roboto',
    }),
    fontSize: Platform.select({
      ios: 17,
      android: 16,
    }),
  },
});

// ใช้กับ Components
const PlatformComponent = Platform.select({
  ios: IOSComponent,
  android: AndroidComponent,
});

// ใช้กับ Props
const PlatformProps = {
  onPress: Platform.select({
    ios: handleIOSPress,
    android: handleAndroidPress,
  }),
  style: Platform.select({
    ios: iosStyle,
    android: androidStyle,
    default: defaultStyle,
  }),
};

// ตัวอย่างจริง: Navigation Header
const NavigationHeader = ({ title, onBack }) => {
  return (
    <View style={[
      headerStyles.container,
      Platform.select({
        ios: headerStyles.iosContainer,
        android: headerStyles.androidContainer,
      }),
    ]}>
      {/* Back button แบบ platform */}
      {onBack && (
        <TouchableOpacity
          style={headerStyles.backBtn}
          onPress={onBack}
        >
          {Platform.select({
            ios: (
              <View style={{ flexDirection: 'row', alignItems: 'center', gap: 4 }}>
                <Ionicons name="chevron-back" size={28} color="#007AFF" />
                <Text style={{ color: '#007AFF', fontSize: 17 }}>Back</Text>
              </View>
            ),
            android: (
              <MaterialIcons name="arrow-back" size={24} color="#333" />
            ),
          })}
        </TouchableOpacity>
      )}

      {/* Title */}
      <Text style={[
        headerStyles.title,
        Platform.select({
          ios: headerStyles.iosTitle,
          android: headerStyles.androidTitle,
        }),
      ]}>
        {title}
      </Text>
    </View>
  );
};

const headerStyles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  iosContainer: {
    justifyContent: 'center',
    height: 44,
    paddingHorizontal: 16,
  },
  androidContainer: {
    justifyContent: 'flex-start',
    height: 56,
    paddingHorizontal: 16,
    backgroundColor: '#fff',
    elevation: 4,
  },
  backBtn: {},
  title: {
    fontWeight: '600',
  },
  iosTitle: {
    position: 'absolute',
    left: 0,
    right: 0,
    textAlign: 'center',
    fontSize: 17,
  },
  androidTitle: {
    flex: 1,
    fontSize: 20,
    marginLeft: 16,
  },
});
```

---

## 3. Platform-specific Files

React Native รองรับการสร้างไฟล์แยกตาม platform โดยอัตโนมัติ

### รูปแบบการตั้งชื่อไฟล์

```
Button.ios.js      # สำหรับ iOS
Button.android.js  # สำหรับ Android
Button.js          # Fallback / default
```

### ตัวอย่าง: Alert Component

```jsx
// Alert.ios.js
import { Alert } from 'react-native';

const showAlert = (title, message, buttons) => {
  Alert.alert(title, message, buttons);
};

// สำหรับ iOS เพิ่ม prompt support
const showInputAlert = (title, message, onSubmit) => {
  Alert.prompt(
    title,
    message,
    (text) => onSubmit(text),
    'plain-text'
  );
};

export default { showAlert, showInputAlert };
```

```jsx
// Alert.android.js
import { Alert } from 'react-native';

const showAlert = (title, message, buttons) => {
  Alert.alert(title, message, buttons, { cancelable: true });
};

// Android ไม่มี prompt - ต้องใช้ Modal แทน
const showInputAlert = (title, message, onSubmit) => {
  // implement custom modal
};

export default { showAlert, showInputAlert };
```

```jsx
// ใช้งาน - import โดยไม่ระบุ platform
import PlatformAlert from './Alert';

// React Native จะเลือก Alert.ios.js หรือ Alert.android.js ให้อัตโนมัติ
PlatformAlert.showAlert('Test', 'Hello!', [{ text: 'OK' }]);
```

### ตัวอย่าง: DatePicker

```jsx
// DatePicker.ios.js
import React from 'react';
import DateTimePicker from '@react-native-community/datetimepicker';

const DatePicker = ({ date, onChange }) => {
  return (
    <DateTimePicker
      value={date}
      mode="date"
      display="spinner"    // iOS: inline, spinner, compact
      onChange={onChange}
    />
  );
};

export default DatePicker;
```

```jsx
// DatePicker.android.js
import React, { useState } from 'react';
import { TouchableOpacity, Text } from 'react-native';
import DateTimePicker from '@react-native-community/datetimepicker';

const DatePicker = ({ date, onChange }) => {
  const [show, setShow] = useState(false);

  const handleChange = (event, selectedDate) => {
    setShow(false);
    if (selectedDate) onChange(event, selectedDate);
  };

  return (
    <>
      <TouchableOpacity onPress={() => setShow(true)}>
        <Text>{date.toLocaleDateString('th-TH')}</Text>
      </TouchableOpacity>
      {show && (
        <DateTimePicker
          value={date}
          mode="date"
          display="default"   // Android: default, spinner, calendar
          onChange={handleChange}
        />
      )}
    </>
  );
};

export default DatePicker;
```

### ตัวอย่าง: Haptics

```jsx
// HapticFeedback.ios.js
import * as Haptics from 'expo-haptics';

export const tap = () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
export const select = () => Haptics.selectionAsync();
export const success = () => Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
export const error = () => Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
```

```jsx
// HapticFeedback.android.js
import * as Haptics from 'expo-haptics';

// Android haptics แตกต่างเล็กน้อย
export const tap = () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
export const select = () => Haptics.selectionAsync();
export const success = () => Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
export const error = () => Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
```

---

## 4. StatusBar (Platform Differences)

StatusBar มีพฤติกรรมแตกต่างกันมากระหว่าง iOS และ Android

```jsx
import { StatusBar, Platform, View } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

// StatusBar height ที่แตกต่างกัน
const STATUS_BAR_HEIGHT = Platform.select({
  ios: 0,           // iOS จัดการด้วย SafeAreaView
  android: StatusBar.currentHeight || 24,
});

// iOS StatusBar
const IOSStatusBar = () => {
  return (
    <StatusBar
      barStyle="dark-content"    // 'default' | 'light-content' | 'dark-content'
      hidden={false}
      animated={true}
      // translucent ไม่มีผลบน iOS
    />
  );
};

// Android StatusBar
const AndroidStatusBar = () => {
  return (
    <StatusBar
      barStyle="dark-content"
      backgroundColor="#fff"         // สีพื้นหลัง status bar
      translucent={false}            // true: content อยู่ใต้ status bar
      hidden={false}
      animated={true}
    />
  );
};

// Translucent status bar บน Android (Full Bleed)
const TranslucentLayout = () => {
  const insets = useSafeAreaInsets();

  return (
    <View style={{ flex: 1 }}>
      <StatusBar translucent backgroundColor="transparent" barStyle="light-content" />
      
      {/* Header ที่ extend ใต้ status bar */}
      <View style={{
        height: 200 + insets.top,
        backgroundColor: '#007AFF',
        paddingTop: insets.top,
      }}>
        <Text style={{ color: '#fff', padding: 16, fontSize: 24, fontWeight: 'bold' }}>
          Full Bleed Header
        </Text>
      </View>

      {/* Content */}
      <View style={{ flex: 1, padding: 16 }}>
        <Text>Content below header</Text>
      </View>
    </View>
  );
};

// Platform-adaptive status bar
const AdaptiveStatusBar = ({ backgroundColor = '#fff', isDark = false }) => {
  if (Platform.OS === 'android') {
    return (
      <StatusBar
        backgroundColor={backgroundColor}
        barStyle={isDark ? 'light-content' : 'dark-content'}
        translucent={false}
      />
    );
  }

  // iOS
  return (
    <StatusBar
      barStyle={isDark ? 'light-content' : 'dark-content'}
      animated
    />
  );
};
```

---

## 5. Safe Areas

```jsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { Platform } from 'react-native';

// Custom hook สำหรับ platform-specific safe areas
const usePlatformSafeAreas = () => {
  const insets = useSafeAreaInsets();

  return {
    top: insets.top,
    bottom: insets.bottom,
    left: insets.left,
    right: insets.right,

    // Helper values
    headerHeight: Platform.select({
      ios: 44 + insets.top,
      android: 56,
    }),
    tabBarHeight: Platform.select({
      ios: 49 + insets.bottom,
      android: 56,
    }),
  };
};

// Custom Header ที่รองรับ safe area
const PlatformHeader = ({ title, rightButton, onBack, transparent = false }) => {
  const { top, headerHeight } = usePlatformSafeAreas();

  return (
    <View style={[
      headerStyles.header,
      {
        height: headerHeight,
        paddingTop: top,
        backgroundColor: transparent ? 'transparent' : '#fff',
      },
      Platform.OS === 'android' && !transparent && { elevation: 4 },
      Platform.OS === 'ios' && !transparent && {
        borderBottomWidth: 0.5,
        borderBottomColor: '#e0e0e0',
      },
    ]}>
      {/* Left: Back Button */}
      <View style={headerStyles.left}>
        {onBack && (
          <TouchableOpacity onPress={onBack} style={headerStyles.backButton} hitSlop={8}>
            {Platform.OS === 'ios' ? (
              <Ionicons name="chevron-back" size={28} color="#007AFF" />
            ) : (
              <MaterialIcons name="arrow-back" size={24} color="#333" />
            )}
          </TouchableOpacity>
        )}
      </View>

      {/* Center: Title */}
      <View style={headerStyles.center}>
        <Text style={[
          headerStyles.title,
          Platform.OS === 'android' && headerStyles.androidTitle,
        ]}>
          {title}
        </Text>
      </View>

      {/* Right: Action */}
      <View style={headerStyles.right}>
        {rightButton}
      </View>
    </View>
  );
};

const headerStyles = StyleSheet.create({
  header: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    paddingHorizontal: 16,
    paddingBottom: Platform.OS === 'ios' ? 8 : 0,
    justifyContent: 'space-between',
  },
  left: {
    width: 80,
    alignItems: 'flex-start',
    justifyContent: 'center',
    height: Platform.OS === 'android' ? 56 : 44,
  },
  center: {
    flex: 1,
    alignItems: Platform.OS === 'ios' ? 'center' : 'flex-start',
    justifyContent: 'center',
    height: Platform.OS === 'android' ? 56 : 44,
  },
  right: {
    width: 80,
    alignItems: 'flex-end',
    justifyContent: 'center',
    height: Platform.OS === 'android' ? 56 : 44,
  },
  title: {
    fontSize: Platform.OS === 'ios' ? 17 : 20,
    fontWeight: '600',
    color: '#1a1a1a',
  },
  androidTitle: {
    fontSize: 20,
    marginLeft: 16,
  },
  backButton: {},
});
```

---

## 6. Workshop: Platform Adaptive UI

```jsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  Switch,
  Platform,
  StyleSheet,
  ScrollView,
  SafeAreaView,
  StatusBar,
  Dimensions,
} from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

// ============================================
// Platform Constants
// ============================================

const PLATFORM = {
  isIOS: Platform.OS === 'ios',
  isAndroid: Platform.OS === 'android',
  version: Platform.Version,
  isNewIPhone: Platform.OS === 'ios' && Platform.Version >= 11,
};

// ============================================
// Platform-adaptive List Item
// ============================================

const ListItem = ({
  title,
  subtitle,
  icon,
  rightElement,
  onPress,
  destructive = false,
  showArrow = true,
}) => {
  const isIOS = Platform.OS === 'ios';

  if (isIOS) {
    return (
      <TouchableOpacity
        style={iosListStyles.item}
        onPress={onPress}
        activeOpacity={0.5}
      >
        {icon && (
          <View style={iosListStyles.iconContainer}>
            <Text style={{ fontSize: 20 }}>{icon}</Text>
          </View>
        )}
        <View style={iosListStyles.content}>
          <Text style={[
            iosListStyles.title,
            destructive && { color: '#FF3B30' },
          ]}>
            {title}
          </Text>
          {subtitle && (
            <Text style={iosListStyles.subtitle}>{subtitle}</Text>
          )}
        </View>
        {rightElement || (showArrow && (
          <Text style={iosListStyles.arrow}>›</Text>
        ))}
      </TouchableOpacity>
    );
  }

  // Android Material Design style
  return (
    <TouchableOpacity
      style={androidListStyles.item}
      onPress={onPress}
      activeOpacity={0.7}
    >
      {icon && (
        <Text style={{ fontSize: 22, marginRight: 16 }}>{icon}</Text>
      )}
      <View style={androidListStyles.content}>
        <Text style={[
          androidListStyles.title,
          destructive && { color: '#B00020' },
        ]}>
          {title}
        </Text>
        {subtitle && (
          <Text style={androidListStyles.subtitle}>{subtitle}</Text>
        )}
      </View>
      {rightElement}
    </TouchableOpacity>
  );
};

const iosListStyles = StyleSheet.create({
  item: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 12,
    paddingHorizontal: 16,
    backgroundColor: '#fff',
    minHeight: 44,
  },
  iconContainer: {
    width: 36,
    height: 36,
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
    backgroundColor: '#f0f0f0',
  },
  content: { flex: 1 },
  title: { fontSize: 17, color: '#1a1a1a' },
  subtitle: { fontSize: 13, color: '#8E8E93', marginTop: 2 },
  arrow: { fontSize: 20, color: '#C7C7CC' },
});

const androidListStyles = StyleSheet.create({
  item: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 16,
    paddingHorizontal: 16,
    backgroundColor: '#fff',
    minHeight: 56,
  },
  content: { flex: 1 },
  title: { fontSize: 16, color: '#1a1a1a' },
  subtitle: { fontSize: 14, color: '#757575', marginTop: 2 },
});

// ============================================
// Platform-adaptive Section
// ============================================

const Section = ({ title, children, footer }) => {
  const isIOS = Platform.OS === 'ios';

  if (isIOS) {
    return (
      <View style={iosSectionStyles.section}>
        {title && (
          <Text style={iosSectionStyles.header}>{title.toUpperCase()}</Text>
        )}
        <View style={iosSectionStyles.content}>
          {React.Children.map(children, (child, i) => (
            <>
              {child}
              {i < React.Children.count(children) - 1 && (
                <View style={iosSectionStyles.separator} />
              )}
            </>
          ))}
        </View>
        {footer && (
          <Text style={iosSectionStyles.footer}>{footer}</Text>
        )}
      </View>
    );
  }

  // Android
  return (
    <View style={androidSectionStyles.section}>
      {title && (
        <Text style={androidSectionStyles.header}>{title}</Text>
      )}
      {children}
      {footer && (
        <Text style={androidSectionStyles.footer}>{footer}</Text>
      )}
    </View>
  );
};

const iosSectionStyles = StyleSheet.create({
  section: { marginTop: 24, paddingHorizontal: 16 },
  header: {
    fontSize: 13,
    color: '#6D6D72',
    marginBottom: 8,
    paddingLeft: 4,
  },
  content: {
    backgroundColor: '#fff',
    borderRadius: 10,
    overflow: 'hidden',
    borderWidth: 0.5,
    borderColor: '#C6C6C8',
  },
  separator: {
    height: StyleSheet.hairlineWidth,
    backgroundColor: '#C6C6C8',
    marginLeft: 16,
  },
  footer: {
    fontSize: 13,
    color: '#6D6D72',
    marginTop: 8,
    paddingHorizontal: 4,
    lineHeight: 18,
  },
});

const androidSectionStyles = StyleSheet.create({
  section: { marginTop: 8 },
  header: {
    fontSize: 14,
    fontWeight: '600',
    color: '#007AFF',
    paddingVertical: 12,
    paddingHorizontal: 16,
  },
  footer: {
    fontSize: 12,
    color: '#757575',
    paddingHorizontal: 16,
    paddingBottom: 8,
  },
});

// ============================================
// Platform-adaptive Dialog
// ============================================

const showPlatformAlert = (title, message, buttons) => {
  if (Platform.OS === 'ios') {
    // iOS: สร้าง native alert ที่ดูแบบ iOS
    Alert.alert(title, message, buttons, { cancelable: false });
  } else {
    // Android: cancelable ได้
    Alert.alert(title, message, buttons, { cancelable: true });
  }
};

// ============================================
// Platform Adaptive App
// ============================================

const PlatformAdaptiveApp = () => {
  const [notifications, setNotifications] = useState(true);
  const [location, setLocation] = useState(false);
  const [darkMode, setDarkMode] = useState(false);
  const insets = useSafeAreaInsets();
  const isIOS = Platform.OS === 'ios';

  const handleLogout = () => {
    showPlatformAlert(
      'ออกจากระบบ',
      'คุณแน่ใจหรือไม่ว่าต้องการออกจากระบบ?',
      [
        { text: 'ยกเลิก', style: 'cancel' },
        { text: 'ออกจากระบบ', style: 'destructive', onPress: () => {} },
      ]
    );
  };

  return (
    <View style={[
      appStyles.container,
      { paddingTop: isIOS ? insets.top : 0 },
    ]}>
      {/* Status Bar */}
      <StatusBar
        barStyle="dark-content"
        backgroundColor={isIOS ? undefined : '#fff'}
        translucent={false}
      />

      {/* Platform-specific Header */}
      <View style={[
        appStyles.header,
        isIOS ? appStyles.iosHeader : appStyles.androidHeader,
      ]}>
        <Text style={[
          appStyles.headerTitle,
          !isIOS && appStyles.androidHeaderTitle,
        ]}>
          การตั้งค่า
        </Text>
      </View>

      <ScrollView contentContainerStyle={{ paddingBottom: 32 + insets.bottom }}>
        {/* Profile Section */}
        <View style={isIOS ? iosSectionStyles.section : {}}>
          <View style={[
            isIOS ? profileStyles.iosCard : profileStyles.androidCard,
          ]}>
            <View style={profileStyles.avatar}>
              <Text style={{ color: '#fff', fontSize: 28, fontWeight: 'bold' }}>SC</Text>
            </View>
            <View style={{ flex: 1 }}>
              <Text style={profileStyles.name}>สมชาย ไทย</Text>
              <Text style={profileStyles.email}>somchai@example.com</Text>
            </View>
            <Text style={{ fontSize: 20, color: '#C7C7CC' }}>›</Text>
          </View>
        </View>

        {/* Notifications Section */}
        <Section
          title="การแจ้งเตือน"
          footer="เปิดการแจ้งเตือนเพื่อรับข่าวสารล่าสุด"
        >
          <ListItem
            title="การแจ้งเตือน"
            icon="🔔"
            rightElement={
              <Switch
                value={notifications}
                onValueChange={setNotifications}
                trackColor={isIOS
                  ? { false: '#e0e0e0', true: '#34C759' }
                  : { false: '#e0e0e0', true: '#4CAF50' }
                }
                thumbColor={isIOS ? '#fff' : (notifications ? '#fff' : '#fff')}
                ios_backgroundColor="#e0e0e0"
              />
            }
          />
          <ListItem
            title="ตำแหน่งที่ตั้ง"
            icon="📍"
            subtitle="อนุญาตการเข้าถึงตำแหน่ง"
            rightElement={
              <Switch
                value={location}
                onValueChange={setLocation}
                trackColor={{ false: '#e0e0e0', true: isIOS ? '#34C759' : '#4CAF50' }}
                thumbColor="#fff"
                ios_backgroundColor="#e0e0e0"
              />
            }
          />
        </Section>

        {/* Appearance Section */}
        <Section title="หน้าตา">
          <ListItem
            title="โหมดมืด"
            icon="🌙"
            rightElement={
              <Switch
                value={darkMode}
                onValueChange={setDarkMode}
                trackColor={{ false: '#e0e0e0', true: isIOS ? '#007AFF' : '#2196F3' }}
                thumbColor="#fff"
                ios_backgroundColor="#e0e0e0"
              />
            }
          />
          <ListItem
            title="ขนาดตัวอักษร"
            icon="🔤"
            subtitle="ปานกลาง"
            onPress={() => {}}
          />
          <ListItem
            title="ภาษา"
            icon="🌐"
            subtitle="ภาษาไทย"
            onPress={() => {}}
          />
        </Section>

        {/* Account Section */}
        <Section title="บัญชี">
          <ListItem title="โปรไฟล์" icon="👤" onPress={() => {}} />
          <ListItem title="ความเป็นส่วนตัว" icon="🔒" onPress={() => {}} />
          <ListItem title="ความปลอดภัย" icon="🛡️" onPress={() => {}} />
        </Section>

        {/* Danger Section */}
        <Section>
          <ListItem
            title="ออกจากระบบ"
            icon="🚪"
            destructive
            showArrow={false}
            onPress={handleLogout}
          />
        </Section>

        {/* Platform Info */}
        <View style={appStyles.platformInfo}>
          <Text style={appStyles.platformText}>
            {isIOS ? '🍎' : '🤖'} {Platform.OS} {Platform.Version}
          </Text>
        </View>
      </ScrollView>
    </View>
  );
};

const appStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: Platform.OS === 'ios' ? '#F2F2F7' : '#fff' },
  header: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    paddingHorizontal: 16,
  },
  iosHeader: {
    height: 44,
    justifyContent: 'center',
    borderBottomWidth: 0.5,
    borderBottomColor: '#C6C6C8',
    backgroundColor: '#F2F2F7',
    marginBottom: 8,
  },
  androidHeader: {
    height: 56,
    alignItems: 'center',
    backgroundColor: '#fff',
    elevation: 4,
    marginBottom: 0,
  },
  headerTitle: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#1a1a1a',
  },
  androidHeaderTitle: {
    fontSize: 20,
    fontWeight: '600',
  },
  platformInfo: {
    padding: 16,
    alignItems: 'center',
    marginTop: 16,
  },
  platformText: {
    color: '#999',
    fontSize: 13,
  },
});

const profileStyles = StyleSheet.create({
  iosCard: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    borderRadius: 10,
    borderWidth: 0.5,
    borderColor: '#C6C6C8',
    gap: 12,
  },
  androidCard: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    elevation: 2,
    marginHorizontal: 0,
    gap: 12,
  },
  avatar: {
    width: 60,
    height: 60,
    borderRadius: 30,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
  name: { fontSize: 18, fontWeight: '600', color: '#1a1a1a' },
  email: { fontSize: 14, color: '#8E8E93', marginTop: 2 },
});

export default PlatformAdaptiveApp;
```

---

## Tips และ Best Practices

### 1. ใช้ Platform.select() แทน if-else ใน styles
```jsx
// ✅ Clean
const style = Platform.select({
  ios: { shadowColor: '#000', shadowOpacity: 0.1 },
  android: { elevation: 4 },
});

// ⚠️ ยุ่งกว่า
const style = Platform.OS === 'ios'
  ? { shadowColor: '#000', shadowOpacity: 0.1 }
  : { elevation: 4 };
```

### 2. ตั้งชื่อไฟล์แยกสำหรับ business logic สำคัญ
```
src/
  services/
    storage.ios.js
    storage.android.js
  components/
    DatePicker.ios.js
    DatePicker.android.js
```

### 3. อย่าลืม Test บน device จริง
- Haptics ไม่ทำงานบน simulator
- Camera ไม่ทำงานบน emulator
- Push notifications ต้องใช้ real device

### 4. Android Back Button
```jsx
import { BackHandler } from 'react-native';

useEffect(() => {
  if (Platform.OS === 'android') {
    const subscription = BackHandler.addEventListener('hardwareBackPress', () => {
      // handle back press
      return true; // prevent default
    });
    return () => subscription.remove();
  }
}, []);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- `Platform.OS` สำหรับตรวจสอบ platform
- `Platform.select()` สำหรับ conditional values
- Platform-specific files (.ios.js, .android.js)
- StatusBar differences ระหว่าง iOS และ Android
- Safe areas และ custom headers
- Workshop: Platform Adaptive Settings UI

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Debugging เบื้องต้น
