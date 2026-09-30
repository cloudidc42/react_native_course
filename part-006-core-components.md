# Part 006: Core Components: View, Text, Image, TextInput, Button

## สารบัญ
1. [View Component](#view-component)
2. [Text Component](#text-component)
3. [Image Component](#image-component)
4. [TextInput Component](#textinput-component)
5. [Button Component](#button-component)
6. [Pressable](#pressable)
7. [TouchableOpacity](#touchableopacity)
8. [TouchableHighlight](#touchablehighlight)
9. [ScrollView](#scrollview)
10. [SafeAreaView](#safeareaview)
11. [Workshop: สร้าง UI Layout](#workshop)

---

## View Component

View เป็น component พื้นฐานที่สุด เทียบได้กับ `<div>` ใน HTML

### การใช้งานพื้นฐาน

```tsx
import { View } from 'react-native';

// View พื้นฐาน
<View />

// View ที่มี children
<View>
  <Text>Hello</Text>
</View>

// View ที่มี style
<View style={{ backgroundColor: '#007AFF', padding: 16 }}>
  <Text style={{ color: '#fff' }}>Hello</Text>
</View>
```

### Props ที่สำคัญของ View

```tsx
<View
  // Layout
  style={styles.container}

  // Accessibility
  accessible={true}
  accessibilityLabel="Container"
  accessibilityRole="none"
  accessibilityHint="ช่วยเหลือสำหรับ screen reader"

  // Testing
  testID="container-view"
  nativeID="native-container"

  // Pointer Events - ควบคุมว่า element นี้รับ touch event ไหม
  // 'box-none': View เอง ไม่รับ, children รับ
  // 'none': ทั้ง View และ children ไม่รับ
  // 'box-only': View รับ, children ไม่รับ
  // 'auto': ปกติ (default)
  pointerEvents="auto"

  // Collision Avoidance
  collapsable={false}   // ป้องกัน Android จาก optimize

  // Clipping
  removeClippedSubviews={true}  // Performance ใน long list
/>
```

### View Patterns ที่ใช้บ่อย

```tsx
// 1. Container
const Container = ({ children }) => (
  <View style={{ flex: 1, backgroundColor: '#fff' }}>
    {children}
  </View>
);

// 2. Row
const Row = ({ children, gap = 0 }) => (
  <View style={{ flexDirection: 'row', gap }}>
    {children}
  </View>
);

// 3. Center
const Center = ({ children }) => (
  <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
    {children}
  </View>
);

// 4. Divider
const Divider = ({ color = '#e0e0e0', thickness = 1, margin = 16 }) => (
  <View
    style={{
      height: thickness,
      backgroundColor: color,
      marginVertical: margin,
    }}
  />
);

// 5. Spacer
const Spacer = ({ size = 16 }) => (
  <View style={{ height: size }} />
);

// 6. Card
const Card = ({ children, style }) => (
  <View
    style={[
      {
        backgroundColor: '#fff',
        borderRadius: 12,
        padding: 16,
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 4,
        elevation: 3,
      },
      style,
    ]}
  >
    {children}
  </View>
);
```

---

## Text Component

Text ใช้สำหรับแสดงข้อความ ใน React Native ทุก text ต้องอยู่ใน `<Text>`

### ข้อสำคัญ

```tsx
// ❌ ไม่ได้ - Text ต้องอยู่ใน <Text>
<View>
  Hello World
</View>

// ✅ ถูกต้อง
<View>
  <Text>Hello World</Text>
</View>
```

### Props ที่สำคัญ

```tsx
<Text
  // Style
  style={styles.text}

  // จำนวนบรรทัดสูงสุด
  numberOfLines={2}           // ตัดที่ 2 บรรทัด
  ellipsizeMode="tail"        // 'head' | 'middle' | 'tail' | 'clip'

  // Interaction
  onPress={() => console.log('pressed')}
  onLongPress={() => console.log('long pressed')}
  onPressIn={() => console.log('press in')}
  onPressOut={() => console.log('press out')}

  // Selectability
  selectable={true}           // ให้ user select text ได้
  selectTextOnFocus={true}    // select ทั้งหมดเมื่อ focus

  // Accessibility
  accessible={true}
  accessibilityLabel="กดเพื่ออ่านเพิ่มเติม"
  accessibilityRole="link"

  // Android specific
  dataDetectorType="all"      // Auto detect links, phone numbers

  // Nesting allowed
  allowFontScaling={true}     // ปรับขนาดตาม system font

  // Testing
  testID="text-element"
>
  ข้อความ
</Text>
```

### Text Styles ที่สำคัญ

```tsx
const styles = StyleSheet.create({
  // Font
  text1: {
    fontFamily: 'Prompt-Regular',    // ชื่อ font
    fontSize: 16,                    // ขนาด
    fontWeight: 'bold',              // 'normal' | 'bold' | '100' - '900'
    fontStyle: 'italic',             // 'normal' | 'italic'
    letterSpacing: 0.5,              // ระยะห่างตัวอักษร
    lineHeight: 24,                  // ความสูงของบรรทัด
  },

  // Color & Decoration
  text2: {
    color: '#333',
    textDecorationLine: 'underline', // 'none' | 'underline' | 'line-through' | 'underline line-through'
    textDecorationColor: '#007AFF',
    textDecorationStyle: 'solid',    // 'solid' | 'double' | 'dotted' | 'dashed'
  },

  // Alignment
  text3: {
    textAlign: 'center',             // 'left' | 'right' | 'center' | 'justify'
    textAlignVertical: 'center',     // Android only
  },

  // Transform
  text4: {
    textTransform: 'uppercase',      // 'none' | 'uppercase' | 'lowercase' | 'capitalize'
  },

  // Shadow (iOS)
  text5: {
    textShadowColor: 'rgba(0,0,0,0.3)',
    textShadowOffset: { width: 1, height: 1 },
    textShadowRadius: 3,
  },
});
```

### Nested Text (Inline Styling)

```tsx
const RichText = () => (
  <Text style={styles.base}>
    ยินดีต้อนรับ,{' '}
    <Text style={styles.name}>สมชาย</Text>
    {' '}คุณมี{' '}
    <Text style={styles.highlight}>5 ข้อความใหม่</Text>
    {' '}รอคุณอยู่
  </Text>
);

const styles = StyleSheet.create({
  base: { fontSize: 16, color: '#333' },
  name: { fontWeight: 'bold', color: '#007AFF' },
  highlight: { color: '#FF3B30' },
});
```

### Custom Font

```bash
# 1. วางไฟล์ font ใน src/assets/fonts/
# 2. สร้าง react-native.config.js

# react-native.config.js
module.exports = {
  assets: ['./src/assets/fonts'],
};

# 3. Link fonts
npx react-native-asset

# หรือ (เก่า)
npx react-native link
```

```tsx
// 4. ใช้งาน
const styles = StyleSheet.create({
  promptText: {
    fontFamily: 'Prompt-Regular',
    fontSize: 16,
  },
  promptBold: {
    fontFamily: 'Prompt-Bold',
    fontSize: 16,
  },
});
```

---

## Image Component

Image ใช้สำหรับแสดงรูปภาพ

### Local Images

```tsx
import { Image } from 'react-native';

// Static require
<Image source={require('./assets/images/logo.png')} />

// กำหนด size (จำเป็น)
<Image
  source={require('./assets/logo.png')}
  style={{ width: 200, height: 100 }}
/>
```

### Remote Images (URL)

```tsx
// Remote image ต้องกำหนด width และ height
<Image
  source={{ uri: 'https://example.com/image.jpg' }}
  style={{ width: 200, height: 200 }}
/>

// พร้อม headers
<Image
  source={{
    uri: 'https://example.com/image.jpg',
    headers: { Authorization: 'Bearer token123' },
    cache: 'force-cache',  // 'default' | 'reload' | 'force-cache' | 'only-if-cached'
  }}
  style={{ width: 200, height: 200 }}
/>
```

### Props ที่สำคัญ

```tsx
<Image
  source={source}
  
  style={{ width: 200, height: 200 }}
  
  // Resize Mode
  resizeMode="cover"
  // 'cover': ครอบคลุมพื้นที่ ตัดส่วนเกิน
  // 'contain': แสดงทั้งภาพ มี padding ถ้าจำเป็น
  // 'stretch': ยืดให้เต็มพื้นที่ (อาจเสียสัดส่วน)
  // 'repeat': ทำซ้ำ
  // 'center': แสดงตรงกลาง ขนาดเดิม

  // Placeholder
  defaultSource={require('./placeholder.png')}  // iOS only

  // Loading
  loadingIndicatorSource={require('./loading.gif')}  // Android only

  // Fade
  fadeDuration={300}   // Android: fade in duration

  // Events
  onLoad={() => console.log('โหลดสำเร็จ')}
  onError={(error) => console.error('โหลดไม่ได้:', error)}
  onLoadStart={() => console.log('เริ่มโหลด')}
  onLoadEnd={() => console.log('โหลดเสร็จสิ้น')}
  
  // Blur
  blurRadius={2}  // Blur effect

  // Accessibility
  accessible={true}
  accessibilityLabel="รูปภาพสินค้า"
/>
```

### Image ที่มี Fallback/Placeholder

```tsx
import React, { useState } from 'react';
import { Image, View, ActivityIndicator, StyleSheet } from 'react-native';

interface SmartImageProps {
  uri: string;
  width: number;
  height: number;
  placeholder?: string;
}

const SmartImage: React.FC<SmartImageProps> = ({
  uri,
  width,
  height,
  placeholder,
}) => {
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(false);

  return (
    <View style={{ width, height }}>
      {/* Loading indicator */}
      {loading && (
        <View style={[styles.overlay, { width, height }]}>
          <ActivityIndicator color="#007AFF" />
        </View>
      )}

      {/* Error fallback */}
      {error ? (
        <View style={[styles.errorContainer, { width, height }]}>
          <Image
            source={require('./assets/placeholder.png')}
            style={{ width, height }}
            resizeMode="contain"
          />
        </View>
      ) : (
        <Image
          source={{ uri }}
          style={{ width, height }}
          resizeMode="cover"
          onLoadStart={() => setLoading(true)}
          onLoad={() => setLoading(false)}
          onError={() => {
            setLoading(false);
            setError(true);
          }}
        />
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  overlay: {
    position: 'absolute',
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f0f0f0',
    zIndex: 1,
  },
  errorContainer: {
    backgroundColor: '#f0f0f0',
    justifyContent: 'center',
    alignItems: 'center',
  },
});
```

### ImageBackground

```tsx
import { ImageBackground, Text, View } from 'react-native';

const HeroSection = () => (
  <ImageBackground
    source={{ uri: 'https://example.com/hero.jpg' }}
    style={styles.hero}
    resizeMode="cover"
  >
    {/* Overlay */}
    <View style={styles.overlay}>
      <Text style={styles.title}>ชื่อแอปพลิเคชัน</Text>
      <Text style={styles.subtitle}>คำบรรยาย</Text>
    </View>
  </ImageBackground>
);

const styles = StyleSheet.create({
  hero: {
    width: '100%',
    height: 250,
    justifyContent: 'flex-end',
  },
  overlay: {
    backgroundColor: 'rgba(0,0,0,0.5)',
    padding: 20,
  },
  title: {
    color: '#fff',
    fontSize: 28,
    fontWeight: 'bold',
  },
  subtitle: {
    color: 'rgba(255,255,255,0.8)',
    fontSize: 16,
    marginTop: 4,
  },
});
```

---

## TextInput Component

TextInput ใช้สำหรับรับ input จากผู้ใช้

### การใช้งานพื้นฐาน

```tsx
import React, { useState } from 'react';
import { TextInput, View, Text } from 'react-native';

const BasicInput = () => {
  const [value, setValue] = useState('');

  return (
    <View>
      <TextInput
        value={value}
        onChangeText={setValue}
        placeholder="พิมพ์ที่นี่..."
        style={{
          borderWidth: 1,
          borderColor: '#ccc',
          borderRadius: 8,
          padding: 12,
          fontSize: 16,
        }}
      />
      <Text>คุณพิมพ์: {value}</Text>
    </View>
  );
};
```

### Props ที่สำคัญ

```tsx
<TextInput
  // Value
  value={value}
  onChangeText={(text) => setValue(text)}
  defaultValue="ค่าเริ่มต้น"      // Uncontrolled
  
  // Placeholder
  placeholder="Email address"
  placeholderTextColor="#999"

  // Keyboard Type
  keyboardType="default"
  // 'default' | 'numeric' | 'email-address' | 'phone-pad'
  // 'decimal-pad' | 'number-pad' | 'ascii-capable'
  // 'numbers-and-punctuation' | 'url' | 'name-phone-pad'
  // 'twitter' | 'web-search' (iOS only)
  // 'visible-password' (Android only)

  // Return Key
  returnKeyType="done"
  // 'done' | 'go' | 'next' | 'search' | 'send'
  // 'default' | 'emergency-call' | 'google' | 'join' | 'route' | 'yahoo'

  // Auto Behaviors
  autoCapitalize="none"
  // 'none' | 'sentences' | 'words' | 'characters'
  
  autoCorrect={false}
  autoComplete="email"
  // 'off' | 'username' | 'password' | 'email' | 'name'
  // 'tel' | 'street-address' | 'postal-code' | 'cc-number'

  // Security
  secureTextEntry={true}          // Password field

  // Multi-line
  multiline={true}
  numberOfLines={4}               // Android only
  textAlignVertical="top"         // Android: align text to top

  // Length
  maxLength={100}

  // Focus
  autoFocus={true}                // Focus เมื่อ mount
  selectTextOnFocus={true}        // Select all เมื่อ focus
  
  // Events
  onFocus={() => console.log('focused')}
  onBlur={() => console.log('blurred')}
  onSubmitEditing={() => console.log('submitted')}
  onKeyPress={({ nativeEvent: { key } }) => console.log('key:', key)}
  onSelectionChange={({ nativeEvent: { selection } }) => {
    console.log('selection:', selection);
  }}

  // Ref (สำหรับ programmatic control)
  ref={inputRef}

  // Style
  style={styles.input}
  editable={true}                 // false = read only
  
  // iOS specific
  clearButtonMode="while-editing"  // 'never' | 'while-editing' | 'unless-editing' | 'always'
  enablesReturnKeyAutomatically={true}  // Disable return key when empty
  spellCheck={false}
  textContentType="emailAddress"  // iOS autofill
  // 'none' | 'URL' | 'addressCity' | 'addressCityAndState' | 'addressState'
  // 'countryName' | 'creditCardNumber' | 'emailAddress' | 'familyName'
  // 'fullStreetAddress' | 'givenName' | 'jobTitle' | 'location' | 'middleName'
  // 'name' | 'namePrefix' | 'nameSuffix' | 'nickname' | 'organizationName'
  // 'postalCode' | 'streetAddressLine1' | 'streetAddressLine2' | 'sublocality'
  // 'telephoneNumber' | 'username' | 'password' | 'newPassword' | 'oneTimeCode'
/>
```

### Custom Input Component

```tsx
import React, { useState, useRef } from 'react';
import {
  View,
  Text,
  TextInput,
  TextInputProps,
  StyleSheet,
  TouchableOpacity,
  Animated,
} from 'react-native';

interface InputFieldProps extends TextInputProps {
  label: string;
  error?: string;
  hint?: string;
  rightIcon?: React.ReactNode;
  leftIcon?: React.ReactNode;
  required?: boolean;
}

const InputField: React.FC<InputFieldProps> = ({
  label,
  error,
  hint,
  rightIcon,
  leftIcon,
  required = false,
  ...props
}) => {
  const [isFocused, setIsFocused] = useState(false);
  const inputRef = useRef<TextInput>(null);

  const borderColor = error
    ? '#FF3B30'
    : isFocused
    ? '#007AFF'
    : '#D1D5DB';

  return (
    <View style={styles.wrapper}>
      {/* Label */}
      <View style={styles.labelRow}>
        <Text style={styles.label}>{label}</Text>
        {required && <Text style={styles.required}>*</Text>}
      </View>

      {/* Input Container */}
      <TouchableOpacity
        activeOpacity={1}
        onPress={() => inputRef.current?.focus()}
        style={[
          styles.inputContainer,
          { borderColor },
          isFocused && styles.focused,
        ]}
      >
        {leftIcon && <View style={styles.iconLeft}>{leftIcon}</View>}
        
        <TextInput
          ref={inputRef}
          style={[
            styles.input,
            leftIcon && styles.inputWithLeftIcon,
            rightIcon && styles.inputWithRightIcon,
          ]}
          onFocus={() => setIsFocused(true)}
          onBlur={() => setIsFocused(false)}
          placeholderTextColor="#9CA3AF"
          {...props}
        />
        
        {rightIcon && <View style={styles.iconRight}>{rightIcon}</View>}
      </TouchableOpacity>

      {/* Error / Hint */}
      {error ? (
        <Text style={styles.errorText}>{error}</Text>
      ) : hint ? (
        <Text style={styles.hintText}>{hint}</Text>
      ) : null}
    </View>
  );
};

const styles = StyleSheet.create({
  wrapper: {
    marginBottom: 16,
  },
  labelRow: {
    flexDirection: 'row',
    marginBottom: 6,
  },
  label: {
    fontSize: 14,
    fontWeight: '500',
    color: '#374151',
  },
  required: {
    color: '#FF3B30',
    marginLeft: 2,
  },
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    borderWidth: 1.5,
    borderRadius: 8,
    backgroundColor: '#fff',
    paddingHorizontal: 12,
  },
  focused: {
    shadowColor: '#007AFF',
    shadowOffset: { width: 0, height: 0 },
    shadowOpacity: 0.2,
    shadowRadius: 4,
    elevation: 2,
  },
  input: {
    flex: 1,
    paddingVertical: 11,
    fontSize: 15,
    color: '#111827',
  },
  inputWithLeftIcon: {
    paddingLeft: 8,
  },
  inputWithRightIcon: {
    paddingRight: 8,
  },
  iconLeft: {
    marginRight: 4,
  },
  iconRight: {
    marginLeft: 4,
  },
  errorText: {
    marginTop: 4,
    fontSize: 12,
    color: '#FF3B30',
  },
  hintText: {
    marginTop: 4,
    fontSize: 12,
    color: '#6B7280',
  },
});

export default InputField;
```

---

## Button Component

Button ใน React Native ค่อนข้าง basic และมีข้อจำกัด

```tsx
import { Button } from 'react-native';

// แบบ basic (ไม่ค่อยนิยม)
<Button
  title="กดปุ่ม"
  onPress={() => console.log('กด!')}
  color="#007AFF"      // สีปุ่ม (iOS: text color, Android: bg color)
  disabled={false}
  accessibilityLabel="ปุ่มกด"
/>
```

> ⚠️ **ข้อจำกัดของ Button:** ไม่สามารถ customize style ได้มาก แนะนำใช้ `TouchableOpacity` หรือ `Pressable` แทน

---

## Pressable

Pressable คือ component ใหม่ที่แนะนำ สามารถ customize ได้มากที่สุด

```tsx
import { Pressable, Text } from 'react-native';

// แบบง่าย
<Pressable onPress={() => console.log('กด!')}>
  <Text>กดที่นี่</Text>
</Pressable>

// กับ style ที่เปลี่ยนตาม state
<Pressable
  style={({ pressed }) => [
    styles.button,
    pressed && styles.buttonPressed,  // style เมื่อกด
  ]}
  onPress={handlePress}
>
  {({ pressed }) => (
    <Text style={[styles.label, pressed && styles.labelPressed]}>
      {pressed ? 'กำลังกด...' : 'กดปุ่ม'}
    </Text>
  )}
</Pressable>

// Props ทั้งหมด
<Pressable
  onPress={handlePress}
  onLongPress={() => console.log('กดค้าง')}
  onPressIn={() => console.log('เริ่มกด')}
  onPressOut={() => console.log('ปล่อย')}
  delayLongPress={500}     // ms ก่อน long press
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}  // เพิ่ม touch area
  disabled={false}
  android_ripple={{         // Android ripple effect
    color: '#007AFF',
    borderless: false,
    radius: 50,
  }}
>
  <Text>กด</Text>
</Pressable>
```

### Custom Button ด้วย Pressable

```tsx
import React from 'react';
import { Pressable, Text, StyleSheet, ViewStyle } from 'react-native';

interface PressableButtonProps {
  label: string;
  onPress: () => void;
  disabled?: boolean;
  style?: ViewStyle;
}

const PressableButton: React.FC<PressableButtonProps> = ({
  label,
  onPress,
  disabled = false,
  style,
}) => (
  <Pressable
    onPress={onPress}
    disabled={disabled}
    style={({ pressed }) => [
      styles.button,
      pressed && styles.pressed,
      disabled && styles.disabled,
      style,
    ]}
    android_ripple={{ color: 'rgba(255,255,255,0.3)' }}
  >
    {({ pressed }) => (
      <Text style={[styles.label, pressed && styles.labelPressed]}>
        {label}
      </Text>
    )}
  </Pressable>
);

const styles = StyleSheet.create({
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 20,
    paddingVertical: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  pressed: {
    backgroundColor: '#0056B3',
    transform: [{ scale: 0.98 }],
  },
  disabled: {
    backgroundColor: '#BFDBFE',
  },
  label: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  labelPressed: {
    opacity: 0.8,
  },
});

export default PressableButton;
```

---

## TouchableOpacity

TouchableOpacity ลด opacity เมื่อกด เป็นที่นิยมมาก

```tsx
import { TouchableOpacity, Text } from 'react-native';

<TouchableOpacity
  onPress={() => console.log('กด')}
  activeOpacity={0.7}    // 0.0 - 1.0, ต่ำกว่า = โปร่งใสมากกว่า
  disabled={false}
  onLongPress={() => {}}
  delayLongPress={500}
  hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}
>
  <Text>กดปุ่ม</Text>
</TouchableOpacity>
```

### เปรียบเทียบ Touchable Components

| | TouchableOpacity | TouchableHighlight | Pressable |
|--|----------------|-------------------|-----------|
| Feedback | Fade opacity | เปลี่ยนสี underlay | Custom style |
| Flexibility | สูง | ปานกลาง | สูงมาก |
| Android Ripple | ไม่มี | ไม่มี | มี |
| แนะนำ | ✅ ใช้ได้ | ❌ เก่า | ✅ ใหม่ที่สุด |

---

## TouchableHighlight

แสดง highlight เมื่อกด (iOS-like)

```tsx
import { TouchableHighlight, Text, View } from 'react-native';

<TouchableHighlight
  onPress={handlePress}
  underlayColor="rgba(0,122,255,0.1)"   // สีที่แสดงเมื่อกด
  activeOpacity={0.8}
  style={styles.button}
>
  <View>
    <Text>กดปุ่ม</Text>
  </View>
</TouchableHighlight>
```

---

## ScrollView

ScrollView สำหรับ content ที่ยาวเกิน screen

```tsx
import { ScrollView, View, Text } from 'react-native';

<ScrollView
  // Content container style
  contentContainerStyle={styles.contentContainer}

  // Direction
  horizontal={false}           // แนวตั้ง (default)
  // horizontal={true}         // แนวนอน

  // Show scrollbar
  showsVerticalScrollIndicator={false}
  showsHorizontalScrollIndicator={false}

  // Bounce (iOS)
  bounces={true}               // Bounce เมื่อถึงขอบ
  alwaysBounceVertical={false}

  // Scroll
  scrollEnabled={true}
  scrollEventThrottle={16}     // ms ระหว่าง scroll events
  onScroll={({ nativeEvent }) => {
    console.log('Offset Y:', nativeEvent.contentOffset.y);
  }}
  onScrollBeginDrag={() => {}}
  onScrollEndDrag={() => {}}
  onMomentumScrollBegin={() => {}}
  onMomentumScrollEnd={() => {}}

  // Keyboard
  keyboardDismissMode="on-drag"  // 'none' | 'on-drag' | 'interactive'
  keyboardShouldPersistTaps="handled"  // 'never' | 'always' | 'handled'

  // Paging
  pagingEnabled={false}        // Snap to page

  // Refresh
  refreshControl={
    <RefreshControl
      refreshing={refreshing}
      onRefresh={onRefresh}
      colors={['#007AFF']}     // Android
      tintColor="#007AFF"      // iOS
    />
  }

  // Performance
  removeClippedSubviews={false}  // ใช้ระวัง!

  // Snap
  snapToInterval={200}         // Snap ทุกๆ 200px
  decelerationRate="fast"      // 'normal' | 'fast' | number

  // Ref
  ref={scrollViewRef}
>
  {/* Content */}
</ScrollView>
```

### Scroll to position

```tsx
import React, { useRef } from 'react';
import { ScrollView, Button } from 'react-native';

const ScrollExample = () => {
  const scrollRef = useRef<ScrollView>(null);

  const scrollToTop = () => {
    scrollRef.current?.scrollTo({ x: 0, y: 0, animated: true });
  };

  const scrollToBottom = () => {
    scrollRef.current?.scrollToEnd({ animated: true });
  };

  const scrollToPosition = () => {
    scrollRef.current?.scrollTo({ x: 0, y: 300, animated: true });
  };

  return (
    <View style={{ flex: 1 }}>
      <Button title="Scroll to Top" onPress={scrollToTop} />
      <Button title="Scroll to Bottom" onPress={scrollToBottom} />
      
      <ScrollView ref={scrollRef}>
        {/* Content */}
      </ScrollView>
    </View>
  );
};
```

---

## SafeAreaView

SafeAreaView จัดการกับ notch, status bar, home indicator โดยอัตโนมัติ

```tsx
import { SafeAreaView } from 'react-native';

// แบบ built-in (ใช้ได้แต่จำกัด)
<SafeAreaView style={{ flex: 1 }}>
  {/* content */}
</SafeAreaView>

// แนะนำ: ใช้ react-native-safe-area-context
import { SafeAreaProvider, SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';

// ห่อ App ด้วย SafeAreaProvider
const App = () => (
  <SafeAreaProvider>
    <AppContent />
  </SafeAreaProvider>
);

// ใช้ SafeAreaView
const Screen = () => (
  <SafeAreaView style={{ flex: 1 }} edges={['top', 'bottom']}>
    {/* content */}
  </SafeAreaView>
);

// หรือใช้ Insets โดยตรง
const Header = () => {
  const insets = useSafeAreaInsets();
  
  return (
    <View
      style={{
        paddingTop: insets.top,
        paddingHorizontal: 16,
        backgroundColor: '#007AFF',
      }}
    >
      <Text>Header</Text>
    </View>
  );
};
```

---

## Workshop: สร้าง UI Layout

### Workshop 6.1: Login Screen

```tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  Image,
  StyleSheet,
  SafeAreaView,
  KeyboardAvoidingView,
  Platform,
  ScrollView,
} from 'react-native';

const LoginScreen = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [showPassword, setShowPassword] = useState(false);
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    if (!email || !password) {
      alert('กรุณากรอก Email และ Password');
      return;
    }

    setLoading(true);
    // Simulate API call
    setTimeout(() => {
      setLoading(false);
      alert('เข้าสู่ระบบสำเร็จ!');
    }, 1500);
  };

  return (
    <SafeAreaView style={styles.safeArea}>
      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      >
        <ScrollView
          contentContainerStyle={styles.container}
          keyboardShouldPersistTaps="handled"
        >
          {/* Logo */}
          <View style={styles.logoContainer}>
            <View style={styles.logo}>
              <Text style={styles.logoText}>RN</Text>
            </View>
            <Text style={styles.appName}>React Native App</Text>
            <Text style={styles.tagline}>เข้าสู่ระบบเพื่อดำเนินการต่อ</Text>
          </View>

          {/* Form */}
          <View style={styles.form}>
            {/* Email */}
            <View style={styles.inputGroup}>
              <Text style={styles.label}>อีเมล</Text>
              <TextInput
                value={email}
                onChangeText={setEmail}
                placeholder="กรอกอีเมลของคุณ"
                placeholderTextColor="#9CA3AF"
                keyboardType="email-address"
                autoCapitalize="none"
                autoComplete="email"
                style={styles.input}
              />
            </View>

            {/* Password */}
            <View style={styles.inputGroup}>
              <Text style={styles.label}>รหัสผ่าน</Text>
              <View style={styles.passwordContainer}>
                <TextInput
                  value={password}
                  onChangeText={setPassword}
                  placeholder="กรอกรหัสผ่าน"
                  placeholderTextColor="#9CA3AF"
                  secureTextEntry={!showPassword}
                  style={[styles.input, styles.passwordInput]}
                />
                <TouchableOpacity
                  style={styles.showPasswordBtn}
                  onPress={() => setShowPassword(!showPassword)}
                >
                  <Text style={styles.showPasswordText}>
                    {showPassword ? 'ซ่อน' : 'แสดง'}
                  </Text>
                </TouchableOpacity>
              </View>
            </View>

            {/* Forgot Password */}
            <TouchableOpacity style={styles.forgotPassword}>
              <Text style={styles.forgotPasswordText}>ลืมรหัสผ่าน?</Text>
            </TouchableOpacity>

            {/* Login Button */}
            <TouchableOpacity
              style={[styles.loginButton, loading && styles.loginButtonLoading]}
              onPress={handleLogin}
              disabled={loading}
              activeOpacity={0.8}
            >
              <Text style={styles.loginButtonText}>
                {loading ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ'}
              </Text>
            </TouchableOpacity>

            {/* Divider */}
            <View style={styles.divider}>
              <View style={styles.dividerLine} />
              <Text style={styles.dividerText}>หรือ</Text>
              <View style={styles.dividerLine} />
            </View>

            {/* Social Login */}
            <View style={styles.socialButtons}>
              <TouchableOpacity style={styles.socialButton}>
                <Text style={styles.socialButtonText}>Google</Text>
              </TouchableOpacity>
              <TouchableOpacity style={styles.socialButton}>
                <Text style={styles.socialButtonText}>Facebook</Text>
              </TouchableOpacity>
            </View>
          </View>

          {/* Register Link */}
          <View style={styles.registerContainer}>
            <Text style={styles.registerText}>ยังไม่มีบัญชี? </Text>
            <TouchableOpacity>
              <Text style={styles.registerLink}>สมัครสมาชิก</Text>
            </TouchableOpacity>
          </View>
        </ScrollView>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  safeArea: { flex: 1, backgroundColor: '#fff' },
  container: {
    flexGrow: 1,
    padding: 24,
    justifyContent: 'center',
  },
  logoContainer: {
    alignItems: 'center',
    marginBottom: 40,
  },
  logo: {
    width: 72,
    height: 72,
    borderRadius: 20,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
    marginBottom: 16,
  },
  logoText: { color: '#fff', fontSize: 28, fontWeight: 'bold' },
  appName: { fontSize: 24, fontWeight: 'bold', color: '#111', marginBottom: 4 },
  tagline: { fontSize: 14, color: '#6B7280' },
  form: { gap: 0 },
  inputGroup: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '500', color: '#374151', marginBottom: 6 },
  input: {
    borderWidth: 1.5,
    borderColor: '#D1D5DB',
    borderRadius: 8,
    paddingHorizontal: 14,
    paddingVertical: 12,
    fontSize: 15,
    color: '#111',
    backgroundColor: '#FAFAFA',
  },
  passwordContainer: { position: 'relative' },
  passwordInput: { paddingRight: 64 },
  showPasswordBtn: {
    position: 'absolute',
    right: 12,
    top: 0,
    bottom: 0,
    justifyContent: 'center',
  },
  showPasswordText: { fontSize: 13, color: '#007AFF', fontWeight: '500' },
  forgotPassword: { alignSelf: 'flex-end', marginBottom: 20 },
  forgotPasswordText: { fontSize: 13, color: '#007AFF' },
  loginButton: {
    backgroundColor: '#007AFF',
    borderRadius: 10,
    paddingVertical: 14,
    alignItems: 'center',
  },
  loginButtonLoading: { opacity: 0.7 },
  loginButtonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  divider: { flexDirection: 'row', alignItems: 'center', marginVertical: 24 },
  dividerLine: { flex: 1, height: 1, backgroundColor: '#E5E7EB' },
  dividerText: { marginHorizontal: 16, color: '#9CA3AF', fontSize: 13 },
  socialButtons: { flexDirection: 'row', gap: 12 },
  socialButton: {
    flex: 1,
    borderWidth: 1.5,
    borderColor: '#D1D5DB',
    borderRadius: 10,
    paddingVertical: 12,
    alignItems: 'center',
  },
  socialButtonText: { fontSize: 14, color: '#374151', fontWeight: '500' },
  registerContainer: {
    flexDirection: 'row',
    justifyContent: 'center',
    marginTop: 32,
  },
  registerText: { fontSize: 14, color: '#6B7280' },
  registerLink: { fontSize: 14, color: '#007AFF', fontWeight: '600' },
});

export default LoginScreen;
```

### Workshop 6.2: Product Detail Screen

```tsx
import React, { useState } from 'react';
import {
  View, Text, Image, ScrollView, TouchableOpacity,
  StyleSheet, SafeAreaView,
} from 'react-native';

const ProductDetailScreen = () => {
  const [quantity, setQuantity] = useState(1);
  const [selectedSize, setSelectedSize] = useState('M');
  const [selectedColor, setSelectedColor] = useState('#FF6B6B');

  const sizes = ['XS', 'S', 'M', 'L', 'XL', 'XXL'];
  const colors = ['#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#333'];

  const price = 1290;
  const originalPrice = 1890;

  return (
    <SafeAreaView style={styles.container}>
      <ScrollView showsVerticalScrollIndicator={false}>
        {/* Product Image */}
        <Image
          source={{ uri: 'https://via.placeholder.com/400x400' }}
          style={styles.productImage}
          resizeMode="cover"
        />

        {/* Product Info */}
        <View style={styles.content}>
          {/* Title & Price */}
          <View style={styles.titleRow}>
            <Text style={styles.title}>เสื้อ Premium Cotton</Text>
            <View style={styles.priceContainer}>
              <Text style={styles.price}>฿{price.toLocaleString()}</Text>
              <Text style={styles.originalPrice}>฿{originalPrice.toLocaleString()}</Text>
              <View style={styles.discountBadge}>
                <Text style={styles.discountText}>
                  -{Math.round((1 - price/originalPrice) * 100)}%
                </Text>
              </View>
            </View>
          </View>

          {/* Rating */}
          <View style={styles.ratingRow}>
            <Text style={styles.stars}>★★★★½</Text>
            <Text style={styles.ratingText}>4.5 (128 รีวิว)</Text>
          </View>

          {/* Color Selection */}
          <View style={styles.section}>
            <Text style={styles.sectionTitle}>สี</Text>
            <View style={styles.colorOptions}>
              {colors.map(color => (
                <TouchableOpacity
                  key={color}
                  style={[
                    styles.colorOption,
                    { backgroundColor: color },
                    selectedColor === color && styles.colorSelected,
                  ]}
                  onPress={() => setSelectedColor(color)}
                />
              ))}
            </View>
          </View>

          {/* Size Selection */}
          <View style={styles.section}>
            <Text style={styles.sectionTitle}>ไซส์</Text>
            <View style={styles.sizeOptions}>
              {sizes.map(size => (
                <TouchableOpacity
                  key={size}
                  style={[
                    styles.sizeOption,
                    selectedSize === size && styles.sizeSelected,
                  ]}
                  onPress={() => setSelectedSize(size)}
                >
                  <Text
                    style={[
                      styles.sizeText,
                      selectedSize === size && styles.sizeTextSelected,
                    ]}
                  >
                    {size}
                  </Text>
                </TouchableOpacity>
              ))}
            </View>
          </View>

          {/* Quantity */}
          <View style={styles.section}>
            <Text style={styles.sectionTitle}>จำนวน</Text>
            <View style={styles.quantityRow}>
              <TouchableOpacity
                style={styles.qtyButton}
                onPress={() => setQuantity(Math.max(1, quantity - 1))}
              >
                <Text style={styles.qtyButtonText}>−</Text>
              </TouchableOpacity>
              <Text style={styles.quantity}>{quantity}</Text>
              <TouchableOpacity
                style={styles.qtyButton}
                onPress={() => setQuantity(quantity + 1)}
              >
                <Text style={styles.qtyButtonText}>+</Text>
              </TouchableOpacity>
            </View>
          </View>

          {/* Description */}
          <View style={styles.section}>
            <Text style={styles.sectionTitle}>รายละเอียด</Text>
            <Text style={styles.description}>
              เสื้อ Premium Cotton 100% ผ้านุ่มสบาย ระบายอากาศได้ดี
              เหมาะสำหรับสวมใส่ทุกวัน มีให้เลือกหลากหลายสี
              ทนทาน ซักได้ง่าย ไม่ยับง่าย
            </Text>
          </View>
        </View>
      </ScrollView>

      {/* Bottom Action */}
      <View style={styles.bottomAction}>
        <View style={styles.totalPrice}>
          <Text style={styles.totalLabel}>ยอดรวม</Text>
          <Text style={styles.totalValue}>
            ฿{(price * quantity).toLocaleString()}
          </Text>
        </View>
        <TouchableOpacity style={styles.addToCartButton}>
          <Text style={styles.addToCartText}>เพิ่มลงตะกร้า</Text>
        </TouchableOpacity>
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  productImage: { width: '100%', height: 350 },
  content: { padding: 20 },
  titleRow: { marginBottom: 12 },
  title: { fontSize: 22, fontWeight: 'bold', color: '#111', marginBottom: 8 },
  priceContainer: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  price: { fontSize: 24, fontWeight: 'bold', color: '#007AFF' },
  originalPrice: {
    fontSize: 16, color: '#999',
    textDecorationLine: 'line-through',
  },
  discountBadge: {
    backgroundColor: '#FF3B30',
    paddingHorizontal: 8, paddingVertical: 2,
    borderRadius: 4,
  },
  discountText: { color: '#fff', fontSize: 12, fontWeight: '700' },
  ratingRow: { flexDirection: 'row', alignItems: 'center', gap: 6, marginBottom: 20 },
  stars: { fontSize: 16, color: '#FF9500' },
  ratingText: { fontSize: 14, color: '#666' },
  section: { marginBottom: 20 },
  sectionTitle: { fontSize: 16, fontWeight: '600', color: '#333', marginBottom: 12 },
  colorOptions: { flexDirection: 'row', gap: 10 },
  colorOption: { width: 32, height: 32, borderRadius: 16 },
  colorSelected: { borderWidth: 3, borderColor: '#fff', shadowColor: '#000', shadowOpacity: 0.3, shadowRadius: 4, elevation: 4 },
  sizeOptions: { flexDirection: 'row', flexWrap: 'wrap', gap: 8 },
  sizeOption: {
    width: 52, height: 44,
    borderWidth: 1.5, borderColor: '#D1D5DB',
    borderRadius: 8, justifyContent: 'center', alignItems: 'center',
  },
  sizeSelected: { borderColor: '#007AFF', backgroundColor: '#007AFF' },
  sizeText: { fontSize: 14, fontWeight: '500', color: '#333' },
  sizeTextSelected: { color: '#fff' },
  quantityRow: { flexDirection: 'row', alignItems: 'center', gap: 16 },
  qtyButton: {
    width: 40, height: 40,
    borderWidth: 1.5, borderColor: '#D1D5DB',
    borderRadius: 8, justifyContent: 'center', alignItems: 'center',
  },
  qtyButtonText: { fontSize: 20, color: '#333' },
  quantity: { fontSize: 18, fontWeight: '600', minWidth: 32, textAlign: 'center' },
  description: { fontSize: 14, color: '#666', lineHeight: 22 },
  bottomAction: {
    flexDirection: 'row',
    padding: 16,
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    backgroundColor: '#fff',
    alignItems: 'center',
    gap: 16,
  },
  totalPrice: { flex: 1 },
  totalLabel: { fontSize: 12, color: '#999' },
  totalValue: { fontSize: 20, fontWeight: 'bold', color: '#111' },
  addToCartButton: {
    flex: 2,
    backgroundColor: '#007AFF',
    paddingVertical: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  addToCartText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});

export default ProductDetailScreen;
```

---

## Tips และ Best Practices

### 1. KeyboardAvoidingView สำหรับ Form

```tsx
import { KeyboardAvoidingView, Platform } from 'react-native';

<KeyboardAvoidingView
  style={{ flex: 1 }}
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
>
  {/* Form content */}
</KeyboardAvoidingView>
```

### 2. Image Optimization

```tsx
// ✅ ระบุ width/height ให้ชัดเจน
<Image
  source={{ uri: url }}
  style={{ width: 200, height: 200 }}
  resizeMode="cover"
/>

// ✅ ใช้ WebP format (เล็กกว่า PNG 26%)
// source={{ uri: 'image.webp' }}

// ✅ ใช้ @2x, @3x สำหรับ local images
// logo.png, logo@2x.png, logo@3x.png
```

### 3. Text Truncation

```tsx
// ✅ ตัดข้อความเมื่อยาวเกิน
<Text numberOfLines={2} ellipsizeMode="tail">
  ข้อความยาวมากๆ ที่อาจจะยาวเกินกว่าที่จะแสดงได้
</Text>
```

---

## สรุป Part 006

### ได้เรียนรู้

1. **View** - container พื้นฐาน เทียบกับ div
2. **Text** - แสดงข้อความ ทุก text ต้องอยู่ใน Text
3. **Image** - local, remote, ImageBackground
4. **TextInput** - รับ input, keyboard types, custom styling
5. **Button** - basic และ limited
6. **Pressable** - flexible, ripple, style function
7. **TouchableOpacity** - นิยมใช้, activeOpacity
8. **ScrollView** - scrollable container
9. **SafeAreaView** - จัดการ notch และ safe areas

---

**ต่อไป → Part 007: StyleSheet และ Flexbox Layout**
