# Part 025: Fonts และ Typography

## บทนำ

Typography เป็นส่วนสำคัญที่สุดใน UI design ถ้าเลือก font และขนาดถูกต้องจะทำให้ app ดูมีคุณภาพ น่าเชื่อถือ และอ่านง่าย ในบทนี้เราจะเรียนรู้ตั้งแต่การใช้ system fonts ไปจนถึงการโหลด custom fonts และการสร้าง typography system ที่สม่ำเสมอ

## สารบัญ

1. System Fonts
2. Custom Fonts (Google Fonts)
3. Font Weights และ Styles
4. Text Hierarchy
5. Line height, Letter spacing
6. Workshop: Typography System

---

## 1. System Fonts

React Native ใช้ system font โดย default ซึ่งแตกต่างกันบน iOS และ Android

### System Fonts ที่มีอยู่

**iOS:** San Francisco (SF Pro) - Apple ออกแบบมาอ่านง่าย

**Android:** Roboto - Google ออกแบบสำหรับ Android

```jsx
import React from 'react';
import { View, Text, Platform, StyleSheet } from 'react-native';

// System font ค่า default
const SystemFontDemo = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.text}>
        ข้อความนี้ใช้ system font ของอุปกรณ์
      </Text>
      <Text style={{ fontFamily: Platform.select({
        ios: 'System',
        android: 'Roboto',
      })}}>
        Text with platform font
      </Text>
    </View>
  );
};

// iOS Fonts ที่ติดตั้งมาแล้ว
const IOSFonts = () => (
  <View>
    <Text style={{ fontFamily: 'San Francisco' }}>San Francisco</Text>
    <Text style={{ fontFamily: 'Arial' }}>Arial</Text>
    <Text style={{ fontFamily: 'Helvetica Neue' }}>Helvetica Neue</Text>
    <Text style={{ fontFamily: 'Georgia' }}>Georgia</Text>
    <Text style={{ fontFamily: 'Times New Roman' }}>Times New Roman</Text>
    <Text style={{ fontFamily: 'Courier New' }}>Courier New</Text>
    <Text style={{ fontFamily: 'Verdana' }}>Verdana</Text>
    <Text style={{ fontFamily: 'Trebuchet MS' }}>Trebuchet MS</Text>
  </View>
);

// Android Fonts ที่ติดตั้งมาแล้ว
const AndroidFonts = () => (
  <View>
    <Text style={{ fontFamily: 'Roboto' }}>Roboto</Text>
    <Text style={{ fontFamily: 'Roboto-Light' }}>Roboto Light</Text>
    <Text style={{ fontFamily: 'Roboto-Thin' }}>Roboto Thin</Text>
    <Text style={{ fontFamily: 'Roboto-Bold' }}>Roboto Bold</Text>
    <Text style={{ fontFamily: 'sans-serif' }}>sans-serif</Text>
    <Text style={{ fontFamily: 'serif' }}>serif</Text>
    <Text style={{ fontFamily: 'monospace' }}>monospace</Text>
  </View>
);

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, gap: 12 },
  text: { fontSize: 16, color: '#333' },
});
```

---

## 2. Custom Fonts (Google Fonts)

### ติดตั้ง expo-font

```bash
npx expo install expo-font
```

### วิธีที่ 1: โหลด Local Font Files

```bash
# วางไฟล์ font ไว้ที่
assets/fonts/
  ├── Inter-Regular.ttf
  ├── Inter-Medium.ttf
  ├── Inter-SemiBold.ttf
  ├── Inter-Bold.ttf
  └── Inter-ExtraBold.ttf
```

```jsx
import React from 'react';
import { View, Text } from 'react-native';
import { useFonts } from 'expo-font';
import * as SplashScreen from 'expo-splash-screen';

// ป้องกัน splash screen ซ่อนก่อน font โหลดเสร็จ
SplashScreen.preventAutoHideAsync();

export default function App() {
  const [fontsLoaded, fontError] = useFonts({
    'Inter-Regular': require('./assets/fonts/Inter-Regular.ttf'),
    'Inter-Medium': require('./assets/fonts/Inter-Medium.ttf'),
    'Inter-SemiBold': require('./assets/fonts/Inter-SemiBold.ttf'),
    'Inter-Bold': require('./assets/fonts/Inter-Bold.ttf'),
    'Inter-ExtraBold': require('./assets/fonts/Inter-ExtraBold.ttf'),
  });

  React.useEffect(() => {
    if (fontsLoaded || fontError) {
      SplashScreen.hideAsync();
    }
  }, [fontsLoaded, fontError]);

  if (!fontsLoaded && !fontError) {
    return null; // หรือ loading component
  }

  return (
    <View style={{ flex: 1, padding: 20 }}>
      <Text style={{ fontFamily: 'Inter-Regular', fontSize: 16 }}>
        Inter Regular
      </Text>
      <Text style={{ fontFamily: 'Inter-Medium', fontSize: 16 }}>
        Inter Medium
      </Text>
      <Text style={{ fontFamily: 'Inter-SemiBold', fontSize: 16 }}>
        Inter SemiBold
      </Text>
      <Text style={{ fontFamily: 'Inter-Bold', fontSize: 16 }}>
        Inter Bold
      </Text>
      <Text style={{ fontFamily: 'Inter-ExtraBold', fontSize: 16 }}>
        Inter Extra Bold
      </Text>
    </View>
  );
}
```

### วิธีที่ 2: Google Fonts (expo-google-fonts)

```bash
# ติดตั้ง font ที่ต้องการ
npx expo install @expo-google-fonts/inter
npx expo install @expo-google-fonts/playfair-display
npx expo install @expo-google-fonts/noto-sans-thai
```

```jsx
import { useFonts, Inter_400Regular, Inter_600SemiBold, Inter_700Bold } from '@expo-google-fonts/inter';
import { PlayfairDisplay_700Bold } from '@expo-google-fonts/playfair-display';
import { NotoSansThai_400Regular, NotoSansThai_700Bold } from '@expo-google-fonts/noto-sans-thai';

export default function App() {
  const [fontsLoaded] = useFonts({
    Inter_400Regular,
    Inter_600SemiBold,
    Inter_700Bold,
    PlayfairDisplay_700Bold,
    NotoSansThai_400Regular,
    NotoSansThai_700Bold,
  });

  if (!fontsLoaded) return null;

  return (
    <View style={{ flex: 1, padding: 20, gap: 12 }}>
      {/* ภาษาอังกฤษ */}
      <Text style={{ fontFamily: 'Inter_400Regular', fontSize: 16 }}>
        The quick brown fox
      </Text>
      <Text style={{ fontFamily: 'Inter_700Bold', fontSize: 24 }}>
        Bold Heading
      </Text>
      <Text style={{ fontFamily: 'PlayfairDisplay_700Bold', fontSize: 28 }}>
        Elegant Display
      </Text>
      
      {/* ภาษาไทย */}
      <Text style={{ fontFamily: 'NotoSansThai_400Regular', fontSize: 16 }}>
        ข้อความภาษาไทยที่สวยงาม
      </Text>
      <Text style={{ fontFamily: 'NotoSansThai_700Bold', fontSize: 20 }}>
        หัวข้อภาษาไทย
      </Text>
    </View>
  );
}
```

### Font ภาษาไทยที่แนะนำ

```bash
# Font ภาษาไทยที่ดี
npx expo install @expo-google-fonts/noto-sans-thai
npx expo install @expo-google-fonts/kanit
npx expo install @expo-google-fonts/sarabun
npx expo install @expo-google-fonts/prompt
npx expo install @expo-google-fonts/mitr
```

```jsx
import { Kanit_400Regular, Kanit_700Bold } from '@expo-google-fonts/kanit';
import { Sarabun_400Regular, Sarabun_600SemiBold } from '@expo-google-fonts/sarabun';
import { Prompt_300Light, Prompt_400Regular, Prompt_500Medium, Prompt_700Bold } from '@expo-google-fonts/prompt';

// Kanit - modern, clean
// Sarabun - อ่านง่าย เป็นทางการ
// Prompt - สวย เหมาะกับ display
// Mitr - สำหรับ heading

const ThaiTypography = () => {
  return (
    <View style={{ gap: 16 }}>
      <Text style={{ fontFamily: 'Kanit_700Bold', fontSize: 24 }}>
        ฟอนต์ Kanit Bold - สวยทันสมัย
      </Text>
      <Text style={{ fontFamily: 'Sarabun_400Regular', fontSize: 16 }}>
        ฟอนต์ Sarabun Regular - อ่านง่าย เหมาะกับ body text
      </Text>
      <Text style={{ fontFamily: 'Prompt_500Medium', fontSize: 18 }}>
        ฟอนต์ Prompt Medium - สะอาดสวยงาม
      </Text>
    </View>
  );
};
```

---

## 3. Font Weights และ Styles

### Font Weight

```jsx
const FontWeightDemo = () => {
  const weights = [
    { weight: '100', name: 'Thin' },
    { weight: '200', name: 'ExtraLight' },
    { weight: '300', name: 'Light' },
    { weight: '400', name: 'Regular' },
    { weight: '500', name: 'Medium' },
    { weight: '600', name: 'SemiBold' },
    { weight: '700', name: 'Bold' },
    { weight: '800', name: 'ExtraBold' },
    { weight: '900', name: 'Black' },
  ];

  return (
    <View style={{ gap: 8 }}>
      {weights.map(({ weight, name }) => (
        <Text key={weight} style={{ fontSize: 18, fontWeight: weight }}>
          {weight} - {name}
        </Text>
      ))}
    </View>
  );
};

// หมายเหตุ: Android รองรับเฉพาะ 'normal' (400) และ 'bold' (700) สำหรับ system font
// สำหรับ weights อื่นๆ ต้องใช้ custom font ที่มี weight นั้นๆ

// ตัวอย่างที่ถูกต้องสำหรับ Android
const AndroidSafeWeights = () => (
  <View>
    {/* ✅ ใช้ custom font */}
    <Text style={{ fontFamily: 'Inter_300Light' }}>Light with Inter</Text>
    <Text style={{ fontFamily: 'Inter_500Medium' }}>Medium with Inter</Text>
    
    {/* ✅ ใช้ normal/bold ซึ่ง system รองรับ */}
    <Text style={{ fontWeight: 'normal' }}>Normal (400)</Text>
    <Text style={{ fontWeight: 'bold' }}>Bold (700)</Text>
  </View>
);
```

### Font Style

```jsx
const FontStyleDemo = () => (
  <View style={{ gap: 8 }}>
    <Text style={{ fontStyle: 'normal' }}>Normal text</Text>
    <Text style={{ fontStyle: 'italic' }}>Italic text - ข้อความเอียง</Text>
    
    {/* Combine weight + style */}
    <Text style={{ fontWeight: 'bold', fontStyle: 'italic' }}>
      Bold Italic
    </Text>
  </View>
);
```

### Text Decoration

```jsx
const TextDecorationDemo = () => (
  <View style={{ gap: 8 }}>
    <Text style={{ textDecorationLine: 'underline' }}>
      Underline - ขีดเส้นใต้
    </Text>
    <Text style={{ textDecorationLine: 'line-through' }}>
      Strike through - ขีดทับ
    </Text>
    <Text style={{ textDecorationLine: 'underline line-through' }}>
      Both
    </Text>
    <Text style={{
      textDecorationLine: 'underline',
      textDecorationColor: '#FF3B30',
      textDecorationStyle: 'dotted',
    }}>
      Dotted red underline
    </Text>
  </View>
);
```

---

## 4. Text Hierarchy

การสร้าง text hierarchy ที่ดีทำให้เนื้อหาอ่านง่ายและมีลำดับความสำคัญชัดเจน

### Type Scale

```jsx
// Type Scale ตาม Material Design 3
const TypeScale = {
  // Display - ใหญ่มาก สำหรับ hero text
  displayLarge: { fontSize: 57, fontWeight: '400', lineHeight: 64, letterSpacing: -0.25 },
  displayMedium: { fontSize: 45, fontWeight: '400', lineHeight: 52 },
  displaySmall: { fontSize: 36, fontWeight: '400', lineHeight: 44 },

  // Headline - สำหรับ section headers
  headlineLarge: { fontSize: 32, fontWeight: '400', lineHeight: 40 },
  headlineMedium: { fontSize: 28, fontWeight: '400', lineHeight: 36 },
  headlineSmall: { fontSize: 24, fontWeight: '400', lineHeight: 32 },

  // Title - สำหรับ card titles, nav titles
  titleLarge: { fontSize: 22, fontWeight: '400', lineHeight: 28 },
  titleMedium: { fontSize: 16, fontWeight: '500', lineHeight: 24, letterSpacing: 0.15 },
  titleSmall: { fontSize: 14, fontWeight: '500', lineHeight: 20, letterSpacing: 0.1 },

  // Body - สำหรับ content text
  bodyLarge: { fontSize: 16, fontWeight: '400', lineHeight: 24, letterSpacing: 0.5 },
  bodyMedium: { fontSize: 14, fontWeight: '400', lineHeight: 20, letterSpacing: 0.25 },
  bodySmall: { fontSize: 12, fontWeight: '400', lineHeight: 16, letterSpacing: 0.4 },

  // Label - สำหรับ buttons, tabs
  labelLarge: { fontSize: 14, fontWeight: '500', lineHeight: 20, letterSpacing: 0.1 },
  labelMedium: { fontSize: 12, fontWeight: '500', lineHeight: 16, letterSpacing: 0.5 },
  labelSmall: { fontSize: 11, fontWeight: '500', lineHeight: 16, letterSpacing: 0.5 },
};

// ใช้ใน component
const ArticleLayout = () => (
  <View style={{ padding: 20, gap: 12 }}>
    <Text style={[TypeScale.displaySmall, { color: '#1a1a1a' }]}>
      บทความหลัก
    </Text>
    <Text style={[TypeScale.titleMedium, { color: '#666' }]}>
      ผู้เขียน: สมชาย ไทย • 5 ม.ค. 2025
    </Text>
    <Text style={[TypeScale.bodyLarge, { color: '#333' }]}>
      ย่อหน้าแรกของบทความซึ่งมีเนื้อหาสำคัญที่ผู้อ่านควรทราบ
      และมีความยาวพอสมควรเพื่อให้ดูเหมือน content จริง
    </Text>
    <Text style={[TypeScale.headlineSmall, { color: '#1a1a1a' }]}>
      หัวข้อย่อย
    </Text>
    <Text style={[TypeScale.bodyMedium, { color: '#555' }]}>
      เนื้อหาในส่วนย่อย ที่มีขนาดเล็กลงเล็กน้อย
    </Text>
    <Text style={[TypeScale.labelSmall, { color: '#999' }]}>
      อ่านแล้ว 1,234 ครั้ง
    </Text>
  </View>
);
```

### iOS-style Type Scale

```jsx
// Apple's SF Pro text styles (iOS Human Interface Guidelines)
const iOSTypeScale = {
  largeTitle: { fontSize: 34, fontWeight: '700', lineHeight: 41 },
  title1: { fontSize: 28, fontWeight: '700', lineHeight: 34 },
  title2: { fontSize: 22, fontWeight: '700', lineHeight: 28 },
  title3: { fontSize: 20, fontWeight: '600', lineHeight: 25 },
  headline: { fontSize: 17, fontWeight: '600', lineHeight: 22 },
  body: { fontSize: 17, fontWeight: '400', lineHeight: 22 },
  callout: { fontSize: 16, fontWeight: '400', lineHeight: 21 },
  subheadline: { fontSize: 15, fontWeight: '400', lineHeight: 20 },
  footnote: { fontSize: 13, fontWeight: '400', lineHeight: 18 },
  caption1: { fontSize: 12, fontWeight: '400', lineHeight: 16 },
  caption2: { fontSize: 11, fontWeight: '400', lineHeight: 13 },
};

const IOSStyleLayout = () => (
  <View style={{ padding: 20, gap: 8 }}>
    <Text style={iOSTypeScale.largeTitle}>Large Title</Text>
    <Text style={iOSTypeScale.title1}>Title 1</Text>
    <Text style={iOSTypeScale.title2}>Title 2</Text>
    <Text style={iOSTypeScale.title3}>Title 3</Text>
    <Text style={iOSTypeScale.headline}>Headline</Text>
    <Text style={iOSTypeScale.body}>Body text ข้อความปกติ</Text>
    <Text style={iOSTypeScale.callout}>Callout ข้อความโดด</Text>
    <Text style={iOSTypeScale.subheadline}>Subheadline</Text>
    <Text style={iOSTypeScale.footnote}>Footnote หมายเหตุ</Text>
    <Text style={iOSTypeScale.caption1}>Caption 1</Text>
    <Text style={iOSTypeScale.caption2}>Caption 2 เล็กมาก</Text>
  </View>
);
```

---

## 5. Line height, Letter spacing

```jsx
const SpacingDemo = () => (
  <View style={{ gap: 24, padding: 20 }}>
    {/* Line Height */}
    <View>
      <Text style={{ fontSize: 14, fontWeight: '600', color: '#666', marginBottom: 8 }}>
        Line Height
      </Text>
      
      <Text style={{ fontSize: 16, lineHeight: 16, marginBottom: 12, backgroundColor: '#f0f8ff' }}>
        lineHeight: 16 (ชิดกัน) Lorem ipsum dolor sit amet consectetur adipiscing elit
      </Text>
      
      <Text style={{ fontSize: 16, lineHeight: 24, marginBottom: 12, backgroundColor: '#f0f8ff' }}>
        lineHeight: 24 (ปกติ) Lorem ipsum dolor sit amet consectetur adipiscing elit
      </Text>
      
      <Text style={{ fontSize: 16, lineHeight: 32, backgroundColor: '#f0f8ff' }}>
        lineHeight: 32 (กว้าง) Lorem ipsum dolor sit amet consectetur adipiscing elit
      </Text>
    </View>

    {/* Letter Spacing */}
    <View>
      <Text style={{ fontSize: 14, fontWeight: '600', color: '#666', marginBottom: 8 }}>
        Letter Spacing
      </Text>
      
      <Text style={{ fontSize: 16, letterSpacing: -1 }}>
        letterSpacing: -1 (ชิดกัน)
      </Text>
      <Text style={{ fontSize: 16, letterSpacing: 0 }}>
        letterSpacing: 0 (ปกติ)
      </Text>
      <Text style={{ fontSize: 16, letterSpacing: 1 }}>
        letterSpacing: 1 (ห่างขึ้นนิดหน่อย)
      </Text>
      <Text style={{ fontSize: 16, letterSpacing: 3 }}>
        letterSpacing: 3 (ห่างมาก)
      </Text>
      <Text style={{ fontSize: 12, letterSpacing: 2, textTransform: 'uppercase' }}>
        SPACED UPPERCASE LABEL
      </Text>
    </View>
  </View>
);

// การใช้ lineHeight กับภาษาไทย
const ThaiLineHeight = () => (
  <View style={{ padding: 20 }}>
    {/* ภาษาไทยต้องการ lineHeight ที่มากกว่าเพื่อให้วรรณยุกต์แสดงได้ครบ */}
    <Text style={{
      fontSize: 16,
      lineHeight: 28,   // ประมาณ 1.75x ของ fontSize สำหรับภาษาไทย
      color: '#333',
    }}>
      ข้อความภาษาไทยต้องการ line height มากกว่าเล็กน้อย
      เพื่อให้วรรณยุกต์และสระบนล่างแสดงผลได้ครบถ้วน
      ไม่ถูกตัดหรือทับซ้อนกัน
    </Text>
  </View>
);
```

---

## 6. Workshop: Typography System

```jsx
import React, { useState } from 'react';
import {
  View,
  Text,
  ScrollView,
  TouchableOpacity,
  StyleSheet,
  SafeAreaView,
} from 'react-native';
import {
  useFonts,
  Inter_300Light,
  Inter_400Regular,
  Inter_500Medium,
  Inter_600SemiBold,
  Inter_700Bold,
} from '@expo-google-fonts/inter';
import {
  Prompt_400Regular,
  Prompt_500Medium,
  Prompt_700Bold,
} from '@expo-google-fonts/prompt';
import * as SplashScreen from 'expo-splash-screen';

SplashScreen.preventAutoHideAsync();

// ============================================
// Typography Token System
// ============================================

const FONTS = {
  // Body fonts (อ่านง่าย)
  regular: 'Inter_400Regular',
  medium: 'Inter_500Medium',
  semibold: 'Inter_600SemiBold',
  bold: 'Inter_700Bold',
  light: 'Inter_300Light',

  // Display fonts (สำหรับ heading ภาษาไทย)
  thaiRegular: 'Prompt_400Regular',
  thaiMedium: 'Prompt_500Medium',
  thaiBold: 'Prompt_700Bold',
};

const FONT_SIZES = {
  xs: 11,
  sm: 13,
  base: 16,
  md: 16,
  lg: 18,
  xl: 20,
  '2xl': 24,
  '3xl': 30,
  '4xl': 36,
  '5xl': 48,
  '6xl': 60,
};

const LINE_HEIGHTS = {
  none: 1,
  tight: 1.25,
  snug: 1.375,
  normal: 1.5,
  relaxed: 1.625,
  loose: 2,
};

const LETTER_SPACINGS = {
  tighter: -0.8,
  tight: -0.4,
  normal: 0,
  wide: 0.4,
  wider: 0.8,
  widest: 1.6,
};

// Helper function สร้าง text style
const createTextStyle = ({
  size = 'md',
  weight = 'regular',
  lineHeight = 'normal',
  letterSpacing = 'normal',
  color = '#1a1a1a',
  italic = false,
  uppercase = false,
} = {}) => ({
  fontFamily: FONTS[weight] || FONTS.regular,
  fontSize: FONT_SIZES[size] || FONT_SIZES.md,
  lineHeight: (FONT_SIZES[size] || 16) * LINE_HEIGHTS[lineHeight],
  letterSpacing: LETTER_SPACINGS[letterSpacing] || 0,
  color,
  ...(italic && { fontStyle: 'italic' }),
  ...(uppercase && { textTransform: 'uppercase' }),
});

// Typography Presets
const Typography = {
  // Display
  heroTitle: createTextStyle({ size: '5xl', weight: 'bold', lineHeight: 'tight', letterSpacing: 'tighter' }),
  displayTitle: createTextStyle({ size: '4xl', weight: 'bold', lineHeight: 'tight' }),

  // Headings
  h1: createTextStyle({ size: '3xl', weight: 'bold', lineHeight: 'snug' }),
  h2: createTextStyle({ size: '2xl', weight: 'bold', lineHeight: 'snug' }),
  h3: createTextStyle({ size: 'xl', weight: 'semibold', lineHeight: 'snug' }),
  h4: createTextStyle({ size: 'lg', weight: 'semibold', lineHeight: 'normal' }),
  h5: createTextStyle({ size: 'md', weight: 'semibold', lineHeight: 'normal' }),
  h6: createTextStyle({ size: 'sm', weight: 'semibold', lineHeight: 'normal' }),

  // Body
  bodyLg: createTextStyle({ size: 'lg', weight: 'regular', lineHeight: 'relaxed' }),
  body: createTextStyle({ size: 'md', weight: 'regular', lineHeight: 'relaxed' }),
  bodySm: createTextStyle({ size: 'sm', weight: 'regular', lineHeight: 'relaxed' }),

  // UI
  label: createTextStyle({ size: 'sm', weight: 'medium', letterSpacing: 'wide' }),
  caption: createTextStyle({ size: 'xs', weight: 'regular', color: '#666' }),
  overline: createTextStyle({ size: 'xs', weight: 'semibold', letterSpacing: 'widest', uppercase: true }),
  button: createTextStyle({ size: 'md', weight: 'semibold' }),
  link: createTextStyle({ size: 'md', weight: 'regular', color: '#007AFF' }),

  // Code
  code: {
    fontFamily: Platform.select({ ios: 'Courier New', android: 'monospace' }),
    fontSize: 14,
    lineHeight: 22,
    color: '#d63384',
    backgroundColor: '#f8f9fa',
  },
};

// Thai Typography Presets
const TypographyTH = {
  h1: { ...Typography.h1, fontFamily: FONTS.thaiBold },
  h2: { ...Typography.h2, fontFamily: FONTS.thaiBold },
  h3: { ...Typography.h3, fontFamily: FONTS.thaiMedium },
  body: {
    fontFamily: FONTS.thaiRegular,
    fontSize: FONT_SIZES.md,
    lineHeight: FONT_SIZES.md * 1.75, // ภาษาไทยต้องการ line height มากกว่า
    color: '#1a1a1a',
  },
};

// ============================================
// Typography Components
// ============================================

const Heading = ({ level = 1, children, style, ...props }) => {
  const textStyles = {
    1: Typography.h1,
    2: Typography.h2,
    3: Typography.h3,
    4: Typography.h4,
    5: Typography.h5,
    6: Typography.h6,
  };

  return (
    <Text style={[textStyles[level], style]} {...props}>
      {children}
    </Text>
  );
};

const Body = ({ size = 'md', children, style, ...props }) => {
  const textStyles = {
    lg: Typography.bodyLg,
    md: Typography.body,
    sm: Typography.bodySm,
  };

  return (
    <Text style={[textStyles[size], style]} {...props}>
      {children}
    </Text>
  );
};

const Caption = ({ children, style, ...props }) => (
  <Text style={[Typography.caption, style]} {...props}>
    {children}
  </Text>
);

const Label = ({ children, style, ...props }) => (
  <Text style={[Typography.label, style]} {...props}>
    {children}
  </Text>
);

const Code = ({ children, style }) => (
  <Text style={[Typography.code, style]}>{children}</Text>
);

// ============================================
// Typography Showcase
// ============================================

const TypographyShowcase = () => {
  const [activeTab, setActiveTab] = useState('scale');

  const tabs = [
    { id: 'scale', label: 'Scale' },
    { id: 'thai', label: 'ภาษาไทย' },
    { id: 'usage', label: 'ตัวอย่าง' },
  ];

  const ScaleTab = () => (
    <ScrollView contentContainerStyle={{ padding: 20, gap: 16 }}>
      <Section title="Display">
        <Text style={Typography.heroTitle}>Hero</Text>
        <Text style={Typography.displayTitle}>Display</Text>
      </Section>

      <Section title="Headings">
        <Text style={Typography.h1}>Heading 1</Text>
        <Text style={Typography.h2}>Heading 2</Text>
        <Text style={Typography.h3}>Heading 3</Text>
        <Text style={Typography.h4}>Heading 4</Text>
        <Text style={Typography.h5}>Heading 5</Text>
        <Text style={Typography.h6}>Heading 6</Text>
      </Section>

      <Section title="Body">
        <Text style={Typography.bodyLg}>Body Large - ข้อความขนาดใหญ่</Text>
        <Text style={Typography.body}>Body - ข้อความปกติ</Text>
        <Text style={Typography.bodySm}>Body Small - ข้อความเล็ก</Text>
      </Section>

      <Section title="UI Elements">
        <Text style={Typography.label}>Label Element</Text>
        <Text style={Typography.caption}>Caption - ข้อความเล็กมาก</Text>
        <Text style={Typography.overline}>Overline Category</Text>
        <Text style={Typography.button}>Button Text</Text>
        <Text style={Typography.link}>Link Text</Text>
        <Code>const code = 'example';</Code>
      </Section>
    </ScrollView>
  );

  const ThaiTab = () => (
    <ScrollView contentContainerStyle={{ padding: 20, gap: 20 }}>
      <Text style={[TypographyTH.h1, { marginBottom: 8 }]}>
        หัวข้อระดับ 1
      </Text>
      <Text style={[TypographyTH.h2, { marginBottom: 8 }]}>
        หัวข้อระดับ 2 สำหรับส่วนย่อย
      </Text>
      <Text style={[TypographyTH.h3, { marginBottom: 8 }]}>
        หัวข้อระดับ 3 สำหรับรายละเอียด
      </Text>
      <Text style={TypographyTH.body}>
        นี่คือข้อความเนื้อหาภาษาไทยที่มี line height เหมาะสม
        ทำให้อ่านง่ายและไม่ดูอึดอัด วรรณยุกต์และสระทุกตัวแสดงผลได้สมบูรณ์
        โดยไม่ถูกตัดทอนหรือทับซ้อนกับบรรทัดอื่น
      </Text>
    </ScrollView>
  );

  const UsageTab = () => (
    <ScrollView contentContainerStyle={{ gap: 24, padding: 16 }}>
      {/* Article Card */}
      <View style={usageStyles.card}>
        <Caption>เทคโนโลยี • 5 นาที</Caption>
        <Heading level={2} style={{ marginTop: 8, marginBottom: 8 }}>
          React Native 0.73: อะไรใหม่บ้าง?
        </Heading>
        <Body>
          React Native 0.73 มาพร้อมกับ improvements หลายอย่าง
          รวมถึง Bridgeless mode ที่เร็วขึ้นอย่างเห็นได้ชัด
        </Body>
        <View style={usageStyles.meta}>
          <Caption>โดย สมชาย ไทย</Caption>
          <Caption>1,234 views</Caption>
        </View>
      </View>

      {/* Profile Card */}
      <View style={usageStyles.card}>
        <View style={usageStyles.profileHeader}>
          <View style={usageStyles.avatar}>
            <Heading level={4} style={{ color: '#fff' }}>SC</Heading>
          </View>
          <View>
            <Heading level={4}>สมชาย ไทย</Heading>
            <Caption>Senior Developer @ Acme Corp</Caption>
          </View>
        </View>
        <Body size="sm" style={{ marginTop: 12 }}>
          นักพัฒนา React Native ที่มีประสบการณ์มากกว่า 5 ปี
        </Body>
        <View style={usageStyles.stats}>
          {[
            { label: 'โปรเจกต์', value: '24' },
            { label: 'ผู้ติดตาม', value: '1.2K' },
            { label: 'กำลังติดตาม', value: '342' },
          ].map((stat) => (
            <View key={stat.label} style={{ alignItems: 'center' }}>
              <Heading level={3} style={{ color: '#007AFF' }}>{stat.value}</Heading>
              <Caption>{stat.label}</Caption>
            </View>
          ))}
        </View>
      </View>

      {/* Price Tag */}
      <View style={usageStyles.card}>
        <Label style={{ color: '#34C759' }}>ราคาพิเศษ</Label>
        <View style={usageStyles.priceRow}>
          <Text style={[Typography.heroTitle, { color: '#1a1a1a', fontSize: 40 }]}>
            ฿499
          </Text>
          <Text style={[Typography.bodyLg, { color: '#999', textDecorationLine: 'line-through' }]}>
            ฿999
          </Text>
        </View>
        <Caption>ประหยัด ฿500 (50%)</Caption>
      </View>
    </ScrollView>
  );

  return (
    <SafeAreaView style={{ flex: 1, backgroundColor: '#f5f5f5' }}>
      {/* Header */}
      <View style={showcaseStyles.header}>
        <Text style={Typography.h2}>Typography System</Text>
      </View>

      {/* Tabs */}
      <View style={showcaseStyles.tabs}>
        {tabs.map((tab) => (
          <TouchableOpacity
            key={tab.id}
            style={[
              showcaseStyles.tab,
              activeTab === tab.id && showcaseStyles.activeTab,
            ]}
            onPress={() => setActiveTab(tab.id)}
          >
            <Text style={[
              Typography.label,
              activeTab === tab.id && { color: '#007AFF' },
            ]}>
              {tab.label}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      {/* Content */}
      {activeTab === 'scale' && <ScaleTab />}
      {activeTab === 'thai' && <ThaiTab />}
      {activeTab === 'usage' && <UsageTab />}
    </SafeAreaView>
  );
};

const Section = ({ title, children }) => (
  <View style={{ marginBottom: 8 }}>
    <Text style={[Typography.overline, { color: '#999', marginBottom: 12 }]}>
      {title}
    </Text>
    <View style={{ gap: 6 }}>{children}</View>
  </View>
);

const showcaseStyles = StyleSheet.create({
  header: {
    padding: 16,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  tabs: {
    flexDirection: 'row',
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  tab: {
    flex: 1,
    paddingVertical: 12,
    alignItems: 'center',
    borderBottomWidth: 2,
    borderBottomColor: 'transparent',
  },
  activeTab: {
    borderBottomColor: '#007AFF',
  },
});

const usageStyles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
  },
  meta: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginTop: 12,
  },
  profileHeader: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 12,
  },
  avatar: {
    width: 52,
    height: 52,
    borderRadius: 26,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
  stats: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    marginTop: 16,
    paddingTop: 16,
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
  },
  priceRow: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    gap: 12,
    marginVertical: 8,
  },
});

// Main App with font loading
export default function TypographyApp() {
  const [fontsLoaded] = useFonts({
    Inter_300Light,
    Inter_400Regular,
    Inter_500Medium,
    Inter_600SemiBold,
    Inter_700Bold,
    Prompt_400Regular,
    Prompt_500Medium,
    Prompt_700Bold,
  });

  useEffect(() => {
    if (fontsLoaded) {
      SplashScreen.hideAsync();
    }
  }, [fontsLoaded]);

  if (!fontsLoaded) return null;

  return <TypographyShowcase />;
}

export { Typography, TypographyTH, FONTS, FONT_SIZES, Heading, Body, Caption, Label, Code };
```

---

## Tips และ Best Practices

### 1. ภาษาไทยต้องการ Line Height พิเศษ
```jsx
// ✅ ภาษาไทย
const thaiBody = {
  lineHeight: fontSize * 1.7,  // 1.5-2x ของ fontSize
};

// ✅ ภาษาอังกฤษ
const englishBody = {
  lineHeight: fontSize * 1.5,  // 1.3-1.6x ของ fontSize
};
```

### 2. iOS vs Android Font Weight
```jsx
// ✅ ใช้ custom font สำหรับ weight ที่ต้องการ
// Android ไม่รองรับ font weight นอกจาก 400 และ 700 กับ system font
```

### 3. Dynamic Type (Accessibility)
```jsx
// React Native รองรับ Dynamic Type บน iOS ผ่าน allowFontScaling
<Text allowFontScaling={true}>   {/* default: true */}
<Text allowFontScaling={false}>  // ปิด dynamic type
```

### 4. Performance
```jsx
// ✅ Load fonts ครั้งเดียวที่ root
// ✅ ใช้ StyleSheet.create() ไม่ใช่ inline styles
// ✅ Memoize styles ที่ซับซ้อน
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- System fonts บน iOS และ Android
- การโหลด Google Fonts ด้วย `expo-google-fonts`
- Font weights, styles, และ text decorations
- การสร้าง Type Scale และ Text Hierarchy
- Line height และ Letter spacing โดยเฉพาะภาษาไทย
- Workshop: Typography System ที่ครบครัน

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Colors และ Themes
