# Part 023: Touchable Components และ Pressable

## บทนำ

การตอบสนองต่อการกดของผู้ใช้เป็นหัวใจสำคัญของ mobile app ทุกตัว React Native มี component หลายตัวสำหรับจัดการ touch interactions แต่ละตัวมีลักษณะการแสดงผลและการใช้งานที่แตกต่างกัน ในบทนี้เราจะเรียนรู้จากพื้นฐานไปจนถึงการสร้าง button library ที่ใช้งานจริงได้

## สารบัญ

1. TouchableOpacity
2. TouchableHighlight
3. TouchableNativeFeedback
4. Pressable (ตัวใหม่แนะนำ)
5. Haptic Feedback
6. Workshop: Button Components Library

---

## 1. TouchableOpacity

`TouchableOpacity` เป็น component ที่นิยมใช้มากที่สุด เมื่อกดจะลด opacity ของ component ทำให้เห็นว่ากดได้

### Props หลัก

| Prop | Type | Default | ความหมาย |
|------|------|---------|-----------|
| onPress | function | - | เมื่อกด |
| onLongPress | function | - | เมื่อกดค้าง |
| onPressIn | function | - | เมื่อเริ่มกด |
| onPressOut | function | - | เมื่อปล่อยนิ้ว |
| activeOpacity | number | 0.2 | ค่า opacity เมื่อกด |
| disabled | boolean | false | ปิดใช้งาน |
| delayLongPress | number | 500 | ms ก่อน onLongPress |

```jsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
} from 'react-native';

// ตัวอย่างพื้นฐาน
const TouchableOpacityExample = () => {
  const [pressCount, setPressCount] = useState(0);
  const [longPressed, setLongPressed] = useState(false);

  return (
    <View style={styles.container}>
      {/* ปุ่มพื้นฐาน */}
      <TouchableOpacity
        style={styles.button}
        onPress={() => setPressCount(prev => prev + 1)}
        activeOpacity={0.7}
      >
        <Text style={styles.buttonText}>กดฉัน ({pressCount})</Text>
      </TouchableOpacity>

      {/* กดค้าง */}
      <TouchableOpacity
        style={[styles.button, styles.longPressBtn]}
        onLongPress={() => setLongPressed(true)}
        onPressOut={() => setLongPressed(false)}
        delayLongPress={600}
      >
        <Text style={styles.buttonText}>
          {longPressed ? 'กำลังกดค้าง...' : 'กดค้างเพื่อดูผล'}
        </Text>
      </TouchableOpacity>

      {/* ปิดใช้งาน */}
      <TouchableOpacity
        style={[styles.button, styles.disabledBtn]}
        onPress={() => {}}
        disabled={true}
      >
        <Text style={styles.buttonText}>ปุ่มที่ปิดใช้งาน</Text>
      </TouchableOpacity>

      {/* opacity ต่างๆ */}
      {[0.1, 0.3, 0.5, 0.7, 0.9].map((opacity) => (
        <TouchableOpacity
          key={opacity}
          style={styles.opacityBtn}
          activeOpacity={opacity}
        >
          <Text>activeOpacity = {opacity}</Text>
        </TouchableOpacity>
      ))}
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, gap: 12 },
  button: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  longPressBtn: { backgroundColor: '#5856D6' },
  disabledBtn: { backgroundColor: '#ccc' },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  opacityBtn: {
    backgroundColor: '#f0f0f0',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
});
```

### Card ที่กดได้

```jsx
const ClickableCard = ({ title, description, onPress }) => {
  return (
    <TouchableOpacity
      style={cardStyles.card}
      onPress={onPress}
      activeOpacity={0.8}
    >
      <View style={cardStyles.cardContent}>
        <Text style={cardStyles.cardTitle}>{title}</Text>
        <Text style={cardStyles.cardDesc}>{description}</Text>
      </View>
      <Text style={cardStyles.arrow}>›</Text>
    </TouchableOpacity>
  );
};

const cardStyles = StyleSheet.create({
  card: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
  },
  cardContent: { flex: 1 },
  cardTitle: { fontSize: 16, fontWeight: '600', marginBottom: 4 },
  cardDesc: { color: '#666', fontSize: 14 },
  arrow: { fontSize: 24, color: '#ccc', fontWeight: '300' },
});
```

---

## 2. TouchableHighlight

`TouchableHighlight` แสดง highlight สีเมื่อกด ดีสำหรับ list items

### ข้อแตกต่างจาก TouchableOpacity
- ต้องมี child component เดียวเท่านั้น
- แสดง underlay color เมื่อกด
- เหมาะกับ list items ที่มีพื้นหลัง

```jsx
import { TouchableHighlight } from 'react-native';

const TouchableHighlightExample = () => {
  const menuItems = ['หน้าหลัก', 'โปรไฟล์', 'การตั้งค่า', 'ออกจากระบบ'];

  return (
    <View>
      {menuItems.map((item, index) => (
        <TouchableHighlight
          key={index}
          style={highlightStyles.item}
          onPress={() => console.log(`กด: ${item}`)}
          underlayColor="#f0f0f0"      // สีที่แสดงเมื่อกด
          activeOpacity={0.9}
        >
          <View style={highlightStyles.itemContent}>
            <Text style={highlightStyles.itemText}>{item}</Text>
            <Text style={highlightStyles.chevron}>›</Text>
          </View>
        </TouchableHighlight>
      ))}
    </View>
  );
};

const highlightStyles = StyleSheet.create({
  item: {
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  itemContent: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
  },
  itemText: { fontSize: 16, color: '#333' },
  chevron: { fontSize: 20, color: '#ccc' },
});
```

---

## 3. TouchableNativeFeedback (Android)

`TouchableNativeFeedback` ใช้ native ripple effect บน Android

```jsx
import { TouchableNativeFeedback, Platform, View } from 'react-native';

const NativeFeedbackButton = ({ onPress, children }) => {
  // ใช้เฉพาะ Android
  if (Platform.OS === 'android') {
    return (
      <TouchableNativeFeedback
        onPress={onPress}
        background={TouchableNativeFeedback.Ripple('#00000020', false)}
        useForeground={true}
      >
        <View style={nativeStyles.button}>
          {children}
        </View>
      </TouchableNativeFeedback>
    );
  }

  // iOS ใช้ TouchableOpacity แทน
  return (
    <TouchableOpacity onPress={onPress} activeOpacity={0.7}>
      <View style={nativeStyles.button}>
        {children}
      </View>
    </TouchableOpacity>
  );
};

const CrossPlatformButton = ({ onPress, title }) => {
  const Inner = () => (
    <View style={nativeStyles.inner}>
      <Text style={nativeStyles.title}>{title}</Text>
    </View>
  );

  if (Platform.OS === 'android') {
    return (
      <View style={nativeStyles.container}>
        <TouchableNativeFeedback
          onPress={onPress}
          background={TouchableNativeFeedback.Ripple('#fff', false)}
        >
          <Inner />
        </TouchableNativeFeedback>
      </View>
    );
  }

  return (
    <TouchableOpacity
      style={nativeStyles.container}
      onPress={onPress}
      activeOpacity={0.8}
    >
      <Inner />
    </TouchableOpacity>
  );
};

const nativeStyles = StyleSheet.create({
  container: {
    borderRadius: 8,
    overflow: 'hidden',
    backgroundColor: '#007AFF',
  },
  button: { padding: 16 },
  inner: { padding: 16, alignItems: 'center' },
  title: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

## 4. Pressable (ตัวใหม่แนะนำ)

`Pressable` เป็น component ใหม่ที่มีความยืดหยุ่นมากกว่า Touchable components เก่า มี state-based styling และ hit slop

### ทำไมต้องใช้ Pressable
- รองรับ state ที่ละเอียดกว่า (pressed, focused, hovered)
- Hit slop ที่ flexible มากกว่า
- Style ตาม state ได้โดยตรง
- API ที่สะอาดกว่า

```jsx
import React from 'react';
import { Pressable, Text, View, StyleSheet } from 'react-native';

// ตัวอย่างพื้นฐาน
const PressableBasic = () => {
  return (
    <Pressable
      onPress={() => console.log('กด!')}
      onLongPress={() => console.log('กดค้าง!')}
      onPressIn={() => console.log('เริ่มกด')}
      onPressOut={() => console.log('ปล่อย')}
      // Style ตาม state
      style={({ pressed }) => [
        pressStyles.button,
        pressed && pressStyles.buttonPressed,
      ]}
      // Hit slop: พื้นที่กดได้รอบๆ component
      hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
      // ระยะเวลาก่อนถือว่า long press
      delayLongPress={500}
      // ปิดใช้งาน
      disabled={false}
    >
      {({ pressed }) => (
        <Text style={[pressStyles.text, pressed && { color: '#fff' }]}>
          {pressed ? 'กำลังกด...' : 'กดฉัน'}
        </Text>
      )}
    </Pressable>
  );
};

// Pressable Button ที่สมบูรณ์
const PressableButton = ({
  title,
  onPress,
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false,
  icon,
}) => {
  const variants = {
    primary: { bg: '#007AFF', text: '#fff', border: 'transparent' },
    secondary: { bg: '#fff', text: '#007AFF', border: '#007AFF' },
    danger: { bg: '#FF3B30', text: '#fff', border: 'transparent' },
    ghost: { bg: 'transparent', text: '#007AFF', border: 'transparent' },
  };

  const sizes = {
    small: { padding: 8, fontSize: 14, borderRadius: 8 },
    medium: { padding: 14, fontSize: 16, borderRadius: 10 },
    large: { padding: 18, fontSize: 18, borderRadius: 12 },
  };

  const variantStyle = variants[variant];
  const sizeStyle = sizes[size];

  return (
    <Pressable
      onPress={onPress}
      disabled={disabled || loading}
      style={({ pressed }) => [
        {
          backgroundColor: variantStyle.bg,
          borderColor: variantStyle.border,
          borderWidth: variantStyle.border !== 'transparent' ? 1.5 : 0,
          padding: sizeStyle.padding,
          borderRadius: sizeStyle.borderRadius,
          flexDirection: 'row',
          alignItems: 'center',
          justifyContent: 'center',
          gap: 8,
          opacity: disabled ? 0.5 : pressed ? 0.8 : 1,
          transform: [{ scale: pressed ? 0.97 : 1 }],
        },
      ]}
    >
      {loading ? (
        <ActivityIndicator color={variantStyle.text} size="small" />
      ) : (
        <>
          {icon && <Text style={{ fontSize: sizeStyle.fontSize }}>{icon}</Text>}
          <Text
            style={{
              color: variantStyle.text,
              fontSize: sizeStyle.fontSize,
              fontWeight: '600',
            }}
          >
            {title}
          </Text>
        </>
      )}
    </Pressable>
  );
};

// Icon Button
const IconButton = ({ icon, onPress, color = '#007AFF', size = 44 }) => {
  return (
    <Pressable
      onPress={onPress}
      style={({ pressed }) => ({
        width: size,
        height: size,
        borderRadius: size / 2,
        backgroundColor: pressed ? `${color}20` : 'transparent',
        justifyContent: 'center',
        alignItems: 'center',
      })}
      hitSlop={8}
    >
      <Text style={{ fontSize: size * 0.5, color }}>{icon}</Text>
    </Pressable>
  );
};

// Toggle Button
const ToggleButton = ({ label, isActive, onToggle }) => {
  return (
    <Pressable
      onPress={onToggle}
      style={({ pressed }) => [
        toggleStyles.btn,
        isActive && toggleStyles.activeBtn,
        pressed && { opacity: 0.8 },
      ]}
    >
      <Text style={[toggleStyles.text, isActive && toggleStyles.activeText]}>
        {label}
      </Text>
    </Pressable>
  );
};

const pressStyles = StyleSheet.create({
  button: {
    backgroundColor: '#f0f0f0',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  buttonPressed: { backgroundColor: '#007AFF' },
  text: { fontSize: 16, color: '#333', fontWeight: '600' },
});

const toggleStyles = StyleSheet.create({
  btn: {
    paddingVertical: 8,
    paddingHorizontal: 16,
    borderRadius: 20,
    backgroundColor: '#f0f0f0',
    borderWidth: 1,
    borderColor: '#e0e0e0',
  },
  activeBtn: {
    backgroundColor: '#007AFF',
    borderColor: '#007AFF',
  },
  text: { color: '#666', fontSize: 14 },
  activeText: { color: '#fff', fontWeight: '600' },
});
```

---

## 5. Haptic Feedback

Haptic Feedback ทำให้ผู้ใช้รู้สึกว่า app ตอบสนอง เหมือน app ระดับ premium

### ติดตั้ง expo-haptics

```bash
npx expo install expo-haptics
```

```jsx
import * as Haptics from 'expo-haptics';

// ประเภทของ Haptics
const HapticsExample = () => {
  // Light impact - สำหรับ tap ทั่วไป
  const lightTap = () => {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  };

  // Medium impact - สำหรับ selection
  const mediumTap = () => {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
  };

  // Heavy impact - สำหรับ action สำคัญ
  const heavyTap = () => {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);
  };

  // Selection feedback
  const selectionFeedback = () => {
    Haptics.selectionAsync();
  };

  // Notification feedback
  const successFeedback = () => {
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
  };

  const errorFeedback = () => {
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
  };

  const warningFeedback = () => {
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
  };

  return (
    <View style={{ gap: 12, padding: 20 }}>
      <Text style={{ fontSize: 18, fontWeight: 'bold', marginBottom: 8 }}>
        Impact Feedback
      </Text>

      <Pressable style={hapticStyles.btn} onPress={lightTap}>
        <Text>Light Impact</Text>
      </Pressable>

      <Pressable style={hapticStyles.btn} onPress={mediumTap}>
        <Text>Medium Impact</Text>
      </Pressable>

      <Pressable style={hapticStyles.btn} onPress={heavyTap}>
        <Text>Heavy Impact</Text>
      </Pressable>

      <Text style={{ fontSize: 18, fontWeight: 'bold', marginTop: 8, marginBottom: 8 }}>
        Notification Feedback
      </Text>

      <Pressable style={[hapticStyles.btn, hapticStyles.success]} onPress={successFeedback}>
        <Text style={{ color: '#fff' }}>✓ Success</Text>
      </Pressable>

      <Pressable style={[hapticStyles.btn, hapticStyles.error]} onPress={errorFeedback}>
        <Text style={{ color: '#fff' }}>✕ Error</Text>
      </Pressable>

      <Pressable style={[hapticStyles.btn, hapticStyles.warning]} onPress={warningFeedback}>
        <Text>⚠️ Warning</Text>
      </Pressable>
    </View>
  );
};

// Hook สำหรับใช้ Haptics ง่ายขึ้น
const useHaptics = () => {
  const tap = useCallback(() => {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  }, []);

  const select = useCallback(() => {
    Haptics.selectionAsync();
  }, []);

  const success = useCallback(() => {
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
  }, []);

  const error = useCallback(() => {
    Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
  }, []);

  return { tap, select, success, error };
};

// ใช้งาน
const MyButton = () => {
  const haptics = useHaptics();

  return (
    <Pressable
      onPress={() => {
        haptics.tap();
        // ทำงานอื่นๆ
      }}
    >
      <Text>กดพร้อม Haptics</Text>
    </Pressable>
  );
};

const hapticStyles = StyleSheet.create({
  btn: {
    backgroundColor: '#f0f0f0',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  success: { backgroundColor: '#34C759' },
  error: { backgroundColor: '#FF3B30' },
  warning: { backgroundColor: '#FF9500' },
});
```

---

## 6. Workshop: Button Components Library

สร้าง Button Library ที่ครบครันและใช้งานได้จริง

```jsx
import React, { useState, useCallback, useRef } from 'react';
import {
  View,
  Text,
  Pressable,
  ActivityIndicator,
  Animated,
  StyleSheet,
  TouchableOpacity,
  Platform,
} from 'react-native';
import * as Haptics from 'expo-haptics';

// ============================================
// 1. Base Button System
// ============================================

const BUTTON_VARIANTS = {
  primary: {
    background: '#007AFF',
    text: '#ffffff',
    border: null,
    pressedBg: '#0056CC',
  },
  secondary: {
    background: '#ffffff',
    text: '#007AFF',
    border: '#007AFF',
    pressedBg: '#F0F8FF',
  },
  danger: {
    background: '#FF3B30',
    text: '#ffffff',
    border: null,
    pressedBg: '#CC2E25',
  },
  success: {
    background: '#34C759',
    text: '#ffffff',
    border: null,
    pressedBg: '#28A347',
  },
  ghost: {
    background: 'transparent',
    text: '#007AFF',
    border: null,
    pressedBg: '#F0F8FF',
  },
  dark: {
    background: '#1C1C1E',
    text: '#ffffff',
    border: null,
    pressedBg: '#2C2C2E',
  },
};

const BUTTON_SIZES = {
  xs: { height: 32, paddingH: 12, fontSize: 12, borderRadius: 8, iconSize: 14 },
  sm: { height: 40, paddingH: 16, fontSize: 14, borderRadius: 10, iconSize: 16 },
  md: { height: 48, paddingH: 20, fontSize: 16, borderRadius: 12, iconSize: 18 },
  lg: { height: 56, paddingH: 24, fontSize: 18, borderRadius: 14, iconSize: 20 },
  xl: { height: 64, paddingH: 28, fontSize: 20, borderRadius: 16, iconSize: 22 },
};

const Button = ({
  title,
  onPress,
  variant = 'primary',
  size = 'md',
  disabled = false,
  loading = false,
  fullWidth = false,
  leftIcon,
  rightIcon,
  haptic = true,
  style,
  textStyle,
}) => {
  const variantStyle = BUTTON_VARIANTS[variant] || BUTTON_VARIANTS.primary;
  const sizeStyle = BUTTON_SIZES[size] || BUTTON_SIZES.md;
  const scaleAnim = useRef(new Animated.Value(1)).current;

  const handlePressIn = useCallback(() => {
    Animated.spring(scaleAnim, {
      toValue: 0.96,
      useNativeDriver: true,
      speed: 50,
      bounciness: 0,
    }).start();
  }, []);

  const handlePressOut = useCallback(() => {
    Animated.spring(scaleAnim, {
      toValue: 1,
      useNativeDriver: true,
      speed: 50,
      bounciness: 2,
    }).start();
  }, []);

  const handlePress = useCallback(() => {
    if (haptic) {
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    }
    onPress?.();
  }, [haptic, onPress]);

  return (
    <Animated.View
      style={[
        { transform: [{ scale: scaleAnim }] },
        fullWidth && { width: '100%' },
      ]}
    >
      <Pressable
        onPress={handlePress}
        onPressIn={handlePressIn}
        onPressOut={handlePressOut}
        disabled={disabled || loading}
        style={({ pressed }) => [
          btnLibStyles.base,
          {
            height: sizeStyle.height,
            paddingHorizontal: sizeStyle.paddingH,
            borderRadius: sizeStyle.borderRadius,
            backgroundColor: pressed
              ? variantStyle.pressedBg
              : variantStyle.background,
            borderWidth: variantStyle.border ? 1.5 : 0,
            borderColor: variantStyle.border || 'transparent',
            opacity: disabled ? 0.5 : 1,
          },
          fullWidth && { width: '100%' },
          style,
        ]}
      >
        {loading ? (
          <ActivityIndicator
            color={variantStyle.text}
            size={sizeStyle.iconSize}
          />
        ) : (
          <View style={btnLibStyles.content}>
            {leftIcon && (
              <Text style={{ fontSize: sizeStyle.iconSize, marginRight: 6 }}>
                {leftIcon}
              </Text>
            )}
            <Text
              style={[
                btnLibStyles.label,
                {
                  color: variantStyle.text,
                  fontSize: sizeStyle.fontSize,
                },
                textStyle,
              ]}
            >
              {title}
            </Text>
            {rightIcon && (
              <Text style={{ fontSize: sizeStyle.iconSize, marginLeft: 6 }}>
                {rightIcon}
              </Text>
            )}
          </View>
        )}
      </Pressable>
    </Animated.View>
  );
};

// ============================================
// 2. Icon Button
// ============================================

const IconButton = ({
  icon,
  onPress,
  variant = 'ghost',
  size = 'md',
  disabled = false,
  badge,
  haptic = true,
}) => {
  const variantStyle = BUTTON_VARIANTS[variant];
  const buttonSize = { xs: 32, sm: 36, md: 44, lg: 52, xl: 60 }[size];
  const iconSize = { xs: 14, sm: 16, md: 20, lg: 24, xl: 28 }[size];

  const handlePress = useCallback(() => {
    if (haptic) Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    onPress?.();
  }, [haptic, onPress]);

  return (
    <Pressable
      onPress={handlePress}
      disabled={disabled}
      style={({ pressed }) => ({
        width: buttonSize,
        height: buttonSize,
        borderRadius: buttonSize / 2,
        backgroundColor: pressed
          ? variantStyle.pressedBg
          : variantStyle.background,
        borderWidth: variantStyle.border ? 1 : 0,
        borderColor: variantStyle.border || 'transparent',
        justifyContent: 'center',
        alignItems: 'center',
        opacity: disabled ? 0.5 : 1,
      })}
      hitSlop={8}
    >
      <Text style={{ fontSize: iconSize }}>{icon}</Text>
      {badge !== undefined && (
        <View style={[btnLibStyles.badge, { backgroundColor: '#FF3B30' }]}>
          <Text style={btnLibStyles.badgeText}>
            {badge > 99 ? '99+' : badge}
          </Text>
        </View>
      )}
    </Pressable>
  );
};

// ============================================
// 3. Floating Action Button
// ============================================

const FAB = ({
  icon = '+',
  onPress,
  color = '#007AFF',
  size = 56,
  extended = false,
  label,
}) => {
  const scaleAnim = useRef(new Animated.Value(1)).current;

  return (
    <Animated.View
      style={[
        btnLibStyles.fab,
        {
          width: extended ? 'auto' : size,
          height: size,
          borderRadius: size / 2,
          backgroundColor: color,
          transform: [{ scale: scaleAnim }],
        },
      ]}
    >
      <Pressable
        onPress={() => {
          Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
          onPress?.();
        }}
        onPressIn={() => {
          Animated.spring(scaleAnim, {
            toValue: 0.92,
            useNativeDriver: true,
          }).start();
        }}
        onPressOut={() => {
          Animated.spring(scaleAnim, {
            toValue: 1,
            useNativeDriver: true,
          }).start();
        }}
        style={{
          width: extended ? 'auto' : size,
          height: size,
          justifyContent: 'center',
          alignItems: 'center',
          flexDirection: 'row',
          gap: 8,
          paddingHorizontal: extended ? 20 : 0,
        }}
      >
        <Text style={{ fontSize: 24, color: '#fff' }}>{icon}</Text>
        {extended && label && (
          <Text style={{ color: '#fff', fontSize: 16, fontWeight: '600' }}>
            {label}
          </Text>
        )}
      </Pressable>
    </Animated.View>
  );
};

// ============================================
// 4. Chip / Tag Button
// ============================================

const Chip = ({ label, selected, onPress, icon, removable, onRemove }) => {
  return (
    <Pressable
      onPress={onPress}
      style={({ pressed }) => [
        chipStyles.chip,
        selected && chipStyles.selectedChip,
        pressed && { opacity: 0.8 },
      ]}
    >
      {icon && (
        <Text style={{ fontSize: 14, marginRight: 4 }}>{icon}</Text>
      )}
      <Text style={[chipStyles.label, selected && chipStyles.selectedLabel]}>
        {label}
      </Text>
      {removable && (
        <Pressable
          onPress={onRemove}
          style={chipStyles.removeBtn}
          hitSlop={4}
        >
          <Text style={{ fontSize: 12, color: selected ? '#fff' : '#666' }}>✕</Text>
        </Pressable>
      )}
    </Pressable>
  );
};

// ============================================
// 5. Button Group
// ============================================

const ButtonGroup = ({ options, selectedIndex, onSelect }) => {
  return (
    <View style={groupStyles.container}>
      {options.map((option, index) => (
        <Pressable
          key={index}
          onPress={() => {
            Haptics.selectionAsync();
            onSelect(index);
          }}
          style={({ pressed }) => [
            groupStyles.item,
            index === 0 && groupStyles.firstItem,
            index === options.length - 1 && groupStyles.lastItem,
            selectedIndex === index && groupStyles.selectedItem,
            pressed && { opacity: 0.8 },
          ]}
        >
          <Text
            style={[
              groupStyles.itemText,
              selectedIndex === index && groupStyles.selectedText,
            ]}
          >
            {option}
          </Text>
        </Pressable>
      ))}
    </View>
  );
};

// ============================================
// 6. Demo Showcase
// ============================================

const ButtonShowcase = () => {
  const [loading, setLoading] = useState(false);
  const [selected, setSelected] = useState(0);
  const [chips, setChips] = useState(['React', 'Native', 'iOS', 'Android']);

  const handleLoadingPress = () => {
    setLoading(true);
    setTimeout(() => setLoading(false), 2000);
  };

  return (
    <ScrollView contentContainerStyle={{ padding: 20, gap: 24 }}>
      {/* Variants */}
      <Section title="Variants">
        <Button title="Primary" variant="primary" />
        <Button title="Secondary" variant="secondary" />
        <Button title="Danger" variant="danger" onPress={() => {
          Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
        }} />
        <Button title="Success" variant="success" />
        <Button title="Ghost" variant="ghost" />
      </Section>

      {/* Sizes */}
      <Section title="Sizes">
        <Button title="Extra Small" size="xs" />
        <Button title="Small" size="sm" />
        <Button title="Medium" size="md" />
        <Button title="Large" size="lg" />
        <Button title="Extra Large" size="xl" />
      </Section>

      {/* With Icons */}
      <Section title="With Icons">
        <Button title="ค้นหา" leftIcon="🔍" />
        <Button title="บันทึก" rightIcon="💾" />
        <Button title="แชร์" leftIcon="📤" variant="secondary" />
        <Button title="ลบ" leftIcon="🗑️" variant="danger" />
      </Section>

      {/* States */}
      <Section title="States">
        <Button title="Loading..." loading={loading} onPress={handleLoadingPress} />
        <Button title="Disabled" disabled={true} />
        <Button title="Full Width" fullWidth={true} />
      </Section>

      {/* Icon Buttons */}
      <Section title="Icon Buttons" row>
        <IconButton icon="❤️" variant="ghost" />
        <IconButton icon="🔔" variant="ghost" badge={5} />
        <IconButton icon="⚙️" variant="ghost" />
        <IconButton icon="🏠" variant="primary" />
      </Section>

      {/* Button Group */}
      <Section title="Button Group">
        <ButtonGroup
          options={['รายวัน', 'รายสัปดาห์', 'รายเดือน']}
          selectedIndex={selected}
          onSelect={setSelected}
        />
      </Section>

      {/* Chips */}
      <Section title="Chips" row>
        {chips.map((chip, index) => (
          <Chip
            key={chip}
            label={chip}
            selected={index === 0}
            onPress={() => {}}
            removable
            onRemove={() => setChips(prev => prev.filter((_, i) => i !== index))}
          />
        ))}
      </Section>
    </ScrollView>
  );
};

// Helper Section Component
const Section = ({ title, children, row = false }) => (
  <View>
    <Text style={{ fontSize: 16, fontWeight: '600', color: '#666', marginBottom: 12 }}>
      {title}
    </Text>
    <View style={[{ gap: 10 }, row && { flexDirection: 'row', flexWrap: 'wrap' }]}>
      {children}
    </View>
  </View>
);

// Styles
const btnLibStyles = StyleSheet.create({
  base: {
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
  },
  content: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
  },
  label: { fontWeight: '600' },
  badge: {
    position: 'absolute',
    top: -2,
    right: -2,
    minWidth: 18,
    height: 18,
    borderRadius: 9,
    justifyContent: 'center',
    alignItems: 'center',
    paddingHorizontal: 4,
  },
  badgeText: { color: '#fff', fontSize: 10, fontWeight: 'bold' },
  fab: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.3,
    shadowRadius: 8,
    elevation: 8,
  },
});

const chipStyles = StyleSheet.create({
  chip: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 6,
    paddingHorizontal: 12,
    borderRadius: 20,
    borderWidth: 1,
    borderColor: '#e0e0e0',
    backgroundColor: '#f9f9f9',
  },
  selectedChip: {
    backgroundColor: '#007AFF',
    borderColor: '#007AFF',
  },
  label: { fontSize: 14, color: '#333' },
  selectedLabel: { color: '#fff', fontWeight: '600' },
  removeBtn: { marginLeft: 6 },
});

const groupStyles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    borderRadius: 10,
    borderWidth: 1,
    borderColor: '#007AFF',
    overflow: 'hidden',
  },
  item: {
    flex: 1,
    paddingVertical: 10,
    alignItems: 'center',
    backgroundColor: '#fff',
    borderRightWidth: 1,
    borderRightColor: '#007AFF',
  },
  firstItem: {},
  lastItem: { borderRightWidth: 0 },
  selectedItem: { backgroundColor: '#007AFF' },
  itemText: { fontSize: 14, color: '#007AFF' },
  selectedText: { color: '#fff', fontWeight: '600' },
});

export {
  Button,
  IconButton,
  FAB,
  Chip,
  ButtonGroup,
  ButtonShowcase,
};

export default ButtonShowcase;
```

---

## Tips และ Best Practices

### 1. ใช้ Pressable สำหรับ custom components ใหม่

```jsx
// ✅ แนะนำ
<Pressable style={({ pressed }) => [style, pressed && pressedStyle]}>

// ⚠️ เก่ากว่า แต่ยังใช้ได้
<TouchableOpacity activeOpacity={0.7}>
```

### 2. Hit Slop สำหรับปุ่มเล็ก

```jsx
// ✅ ปุ่มเล็กควรมี hit slop
<Pressable hitSlop={12}>
  <Text style={{ fontSize: 16 }}>×</Text>
</Pressable>
```

### 3. Haptic Feedback

```jsx
// ✅ เพิ่ม haptic เพื่อ UX ที่ดีขึ้น
const handlePress = () => {
  Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  // action
};
```

### 4. Accessibility

```jsx
<Pressable
  accessible={true}
  accessibilityRole="button"
  accessibilityLabel="บันทึกข้อมูล"
  accessibilityHint="แตะเพื่อบันทึกข้อมูลของคุณ"
  accessibilityState={{ disabled: isDisabled, busy: isLoading }}
>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- `TouchableOpacity` - ลด opacity เมื่อกด
- `TouchableHighlight` - แสดง highlight สีเมื่อกด
- `TouchableNativeFeedback` - native ripple effect บน Android
- `Pressable` - component ใหม่ที่ยืดหยุ่นกว่า
- Haptic Feedback สำหรับ physical response
- Workshop: Button Library ที่ครบครัน

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Icons และ Vector Icons
