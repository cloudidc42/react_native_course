# Part 026: Colors และ Themes

## บทนำ

การจัดการ colors และ themes เป็นสิ่งสำคัญในการสร้าง app ที่ดูสอดคล้องกัน (consistent) โดยเฉพาะเมื่อต้องรองรับทั้ง light mode และ dark mode ในบทนี้เราจะสร้าง theming system ที่ครบครัน

## สารบัญ

1. Color Palette
2. Dark Mode
3. ThemeContext
4. Styled Components
5. React Native Paper Themes
6. Workshop: Light/Dark Theme Toggle

---

## 1. Color Palette

### Design Token System

```jsx
// tokens/colors.js

// Primitive colors (raw values)
export const primitiveColors = {
  // Blues
  blue50: '#EFF6FF',
  blue100: '#DBEAFE',
  blue200: '#BFDBFE',
  blue300: '#93C5FD',
  blue400: '#60A5FA',
  blue500: '#3B82F6',
  blue600: '#2563EB',
  blue700: '#1D4ED8',
  blue800: '#1E40AF',
  blue900: '#1E3A8A',

  // Grays
  gray50: '#F9FAFB',
  gray100: '#F3F4F6',
  gray200: '#E5E7EB',
  gray300: '#D1D5DB',
  gray400: '#9CA3AF',
  gray500: '#6B7280',
  gray600: '#4B5563',
  gray700: '#374151',
  gray800: '#1F2937',
  gray900: '#111827',

  // Reds
  red50: '#FEF2F2',
  red100: '#FEE2E2',
  red500: '#EF4444',
  red600: '#DC2626',
  red700: '#B91C1C',

  // Greens
  green50: '#F0FDF4',
  green100: '#DCFCE7',
  green500: '#22C55E',
  green600: '#16A34A',

  // Yellows/Oranges
  yellow50: '#FEFCE8',
  yellow400: '#FACC15',
  yellow500: '#EAB308',
  orange500: '#F97316',

  // Purple
  purple500: '#A855F7',
  purple600: '#9333EA',

  // Pure
  white: '#FFFFFF',
  black: '#000000',
  transparent: 'transparent',
};

// iOS System Colors
export const iosColors = {
  blue: '#007AFF',
  green: '#34C759',
  indigo: '#5856D6',
  orange: '#FF9500',
  pink: '#FF2D55',
  purple: '#AF52DE',
  red: '#FF3B30',
  teal: '#5AC8FA',
  yellow: '#FFCC00',
  gray: '#8E8E93',
  gray2: '#AEAEB2',
  gray3: '#C7C7CC',
  gray4: '#D1D1D6',
  gray5: '#E5E5EA',
  gray6: '#F2F2F7',
};

// Semantic colors
export const semanticColors = {
  light: {
    primary: '#007AFF',
    primaryHover: '#0056CC',
    secondary: '#5856D6',
    success: '#34C759',
    warning: '#FF9500',
    danger: '#FF3B30',
    info: '#5AC8FA',

    textPrimary: '#000000',
    textSecondary: '#3C3C43',
    textTertiary: '#8E8E93',
    textDisabled: '#C7C7CC',
    textInverse: '#FFFFFF',
    textLink: '#007AFF',

    backgroundPrimary: '#FFFFFF',
    backgroundSecondary: '#F2F2F7',
    backgroundTertiary: '#FFFFFF',
    backgroundGrouped: '#F2F2F7',
    backgroundOverlay: 'rgba(0,0,0,0.4)',

    borderPrimary: '#C6C6C8',
    borderSecondary: '#E5E5EA',
    borderFocus: '#007AFF',

    shadowColor: '#000000',
    shadowOpacity: 0.1,
  },
  dark: {
    primary: '#0A84FF',
    primaryHover: '#0060D0',
    secondary: '#5E5CE6',
    success: '#30D158',
    warning: '#FF9F0A',
    danger: '#FF453A',
    info: '#64D2FF',

    textPrimary: '#FFFFFF',
    textSecondary: '#EBEBF5',
    textTertiary: '#8E8E93',
    textDisabled: '#48484A',
    textInverse: '#000000',
    textLink: '#0A84FF',

    backgroundPrimary: '#000000',
    backgroundSecondary: '#1C1C1E',
    backgroundTertiary: '#2C2C2E',
    backgroundGrouped: '#000000',
    backgroundOverlay: 'rgba(0,0,0,0.6)',

    borderPrimary: '#38383A',
    borderSecondary: '#2C2C2E',
    borderFocus: '#0A84FF',

    shadowColor: '#000000',
    shadowOpacity: 0.3,
  },
};
```

---

## 2. Dark Mode

React Native รองรับ dark mode ผ่าน `useColorScheme`

```jsx
import { useColorScheme, ColorSchemeName } from 'react-native';

// Hook พื้นฐาน
const BasicDarkMode = () => {
  const colorScheme = useColorScheme(); // 'light' | 'dark' | null

  const isDark = colorScheme === 'dark';

  return (
    <View style={{
      flex: 1,
      backgroundColor: isDark ? '#000' : '#fff',
    }}>
      <Text style={{
        color: isDark ? '#fff' : '#000',
        fontSize: 16,
      }}>
        โหมดปัจจุบัน: {colorScheme}
      </Text>
    </View>
  );
};

// ตัวอย่างที่ปรับตาม scheme
const ThemedCard = ({ title, description }) => {
  const colorScheme = useColorScheme();
  const isDark = colorScheme === 'dark';

  return (
    <View style={[
      cardStyles.card,
      isDark ? cardStyles.darkCard : cardStyles.lightCard,
    ]}>
      <Text style={[
        cardStyles.title,
        { color: isDark ? '#fff' : '#1a1a1a' },
      ]}>
        {title}
      </Text>
      <Text style={[
        cardStyles.description,
        { color: isDark ? '#aaa' : '#666' },
      ]}>
        {description}
      </Text>
    </View>
  );
};

const cardStyles = StyleSheet.create({
  card: {
    borderRadius: 12,
    padding: 16,
    marginBottom: 12,
  },
  lightCard: {
    backgroundColor: '#fff',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
  },
  darkCard: {
    backgroundColor: '#1C1C1E',
    borderWidth: 1,
    borderColor: '#38383A',
  },
  title: { fontSize: 18, fontWeight: '600', marginBottom: 8 },
  description: { fontSize: 14, lineHeight: 20 },
});
```

---

## 3. ThemeContext

สร้าง Theme system ที่ครบครันด้วย Context

```jsx
import React, {
  createContext,
  useContext,
  useState,
  useCallback,
  useMemo,
  useEffect,
} from 'react';
import { useColorScheme, Appearance } from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';

// ============================================
// Theme Definitions
// ============================================

const lightTheme = {
  name: 'light',
  colors: {
    primary: '#007AFF',
    primaryLight: '#4DA6FF',
    primaryDark: '#0056CC',

    secondary: '#5856D6',
    accent: '#FF9500',

    background: '#F2F2F7',
    surface: '#FFFFFF',
    surfaceVariant: '#F9F9F9',
    card: '#FFFFFF',

    text: '#000000',
    textSecondary: '#3C3C43',
    textTertiary: '#8E8E93',
    textPlaceholder: '#C7C7CC',
    textInverse: '#FFFFFF',

    border: '#C6C6C8',
    borderLight: '#E5E5EA',
    divider: '#E5E5EA',

    success: '#34C759',
    warning: '#FF9500',
    error: '#FF3B30',
    info: '#5AC8FA',

    overlay: 'rgba(0,0,0,0.5)',
    shadow: '#000000',
  },
  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
    '2xl': 48,
  },
  borderRadius: {
    sm: 6,
    md: 10,
    lg: 14,
    xl: 20,
    full: 9999,
  },
  shadows: {
    sm: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 1 },
      shadowOpacity: 0.05,
      shadowRadius: 2,
      elevation: 1,
    },
    md: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.08,
      shadowRadius: 4,
      elevation: 3,
    },
    lg: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 4 },
      shadowOpacity: 0.12,
      shadowRadius: 8,
      elevation: 6,
    },
    xl: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 8 },
      shadowOpacity: 0.15,
      shadowRadius: 16,
      elevation: 10,
    },
  },
};

const darkTheme = {
  ...lightTheme,
  name: 'dark',
  colors: {
    primary: '#0A84FF',
    primaryLight: '#5EB0FF',
    primaryDark: '#0060D0',

    secondary: '#5E5CE6',
    accent: '#FF9F0A',

    background: '#000000',
    surface: '#1C1C1E',
    surfaceVariant: '#2C2C2E',
    card: '#1C1C1E',

    text: '#FFFFFF',
    textSecondary: '#EBEBF5',
    textTertiary: '#8E8E93',
    textPlaceholder: '#48484A',
    textInverse: '#000000',

    border: '#38383A',
    borderLight: '#2C2C2E',
    divider: '#2C2C2E',

    success: '#30D158',
    warning: '#FF9F0A',
    error: '#FF453A',
    info: '#64D2FF',

    overlay: 'rgba(0,0,0,0.7)',
    shadow: '#000000',
  },
  shadows: {
    sm: { shadowColor: '#000', shadowOffset: { width: 0, height: 1 }, shadowOpacity: 0.2, shadowRadius: 2, elevation: 1 },
    md: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.3, shadowRadius: 4, elevation: 3 },
    lg: { shadowColor: '#000', shadowOffset: { width: 0, height: 4 }, shadowOpacity: 0.4, shadowRadius: 8, elevation: 6 },
    xl: { shadowColor: '#000', shadowOffset: { width: 0, height: 8 }, shadowOpacity: 0.5, shadowRadius: 16, elevation: 10 },
  },
};

// Custom themes
const sepiaTheme = {
  ...lightTheme,
  name: 'sepia',
  colors: {
    ...lightTheme.colors,
    background: '#F4ECD8',
    surface: '#FBF5E6',
    card: '#FBF5E6',
    text: '#3D2B1F',
    textSecondary: '#5C4033',
    border: '#C4A882',
    primary: '#8B5E3C',
  },
};

// ============================================
// Theme Context
// ============================================

const ThemeContext = createContext(null);

export const ThemeProvider = ({ children }) => {
  const systemColorScheme = useColorScheme();
  const [themeMode, setThemeMode] = useState('system'); // 'light' | 'dark' | 'system' | 'sepia'

  // โหลด theme preference จาก storage
  useEffect(() => {
    AsyncStorage.getItem('themeMode').then((saved) => {
      if (saved) setThemeMode(saved);
    });
  }, []);

  const setTheme = useCallback(async (mode) => {
    setThemeMode(mode);
    await AsyncStorage.setItem('themeMode', mode);
  }, []);

  const theme = useMemo(() => {
    if (themeMode === 'system') {
      return systemColorScheme === 'dark' ? darkTheme : lightTheme;
    }
    if (themeMode === 'dark') return darkTheme;
    if (themeMode === 'sepia') return sepiaTheme;
    return lightTheme;
  }, [themeMode, systemColorScheme]);

  const isDark = theme.name === 'dark';

  return (
    <ThemeContext.Provider value={{ theme, themeMode, setTheme, isDark }}>
      {children}
    </ThemeContext.Provider>
  );
};

// Hook
export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme ต้องใช้ภายใน ThemeProvider');
  }
  return context;
};

// ============================================
// Themed Components
// ============================================

const ThemedView = ({ style, ...props }) => {
  const { theme } = useTheme();
  return (
    <View
      style={[{ backgroundColor: theme.colors.background }, style]}
      {...props}
    />
  );
};

const ThemedText = ({ variant = 'primary', style, ...props }) => {
  const { theme } = useTheme();
  const colorMap = {
    primary: theme.colors.text,
    secondary: theme.colors.textSecondary,
    tertiary: theme.colors.textTertiary,
    inverse: theme.colors.textInverse,
    link: theme.colors.primary,
    error: theme.colors.error,
    success: theme.colors.success,
  };

  return (
    <Text style={[{ color: colorMap[variant] }, style]} {...props} />
  );
};

const ThemedCard = ({ style, ...props }) => {
  const { theme } = useTheme();
  return (
    <View
      style={[
        {
          backgroundColor: theme.colors.card,
          borderRadius: theme.borderRadius.lg,
          padding: theme.spacing.md,
          borderWidth: theme.name === 'dark' ? 1 : 0,
          borderColor: theme.colors.border,
        },
        theme.shadows.md,
        style,
      ]}
      {...props}
    />
  );
};
```

---

## 4. Styled Components ใน React Native

```bash
npm install styled-components
```

```jsx
import styled from 'styled-components/native';
import { useTheme } from './ThemeContext';

// ============================================
// Styled Components
// ============================================

// ใช้ theme จาก ThemeProvider
const StyledButton = styled.TouchableOpacity`
  background-color: ${({ theme }) => theme.colors.primary};
  padding: ${({ theme }) => theme.spacing.md}px;
  border-radius: ${({ theme }) => theme.borderRadius.md}px;
  align-items: center;
  opacity: ${({ disabled }) => disabled ? 0.5 : 1};
`;

const ButtonText = styled.Text`
  color: ${({ theme }) => theme.colors.textInverse};
  font-size: 16px;
  font-weight: 600;
`;

const Card = styled.View`
  background-color: ${({ theme }) => theme.colors.card};
  border-radius: ${({ theme }) => theme.borderRadius.lg}px;
  padding: ${({ theme }) => theme.spacing.md}px;
  margin-bottom: ${({ theme }) => theme.spacing.sm}px;
`;

const Title = styled.Text`
  color: ${({ theme }) => theme.colors.text};
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 8px;
`;

const Body = styled.Text`
  color: ${({ theme }) => theme.colors.textSecondary};
  font-size: 14px;
  line-height: 20px;
`;

// Variant props
const Badge = styled.View`
  background-color: ${({ theme, variant = 'primary' }) => {
    const colors = {
      primary: theme.colors.primary,
      success: theme.colors.success,
      warning: theme.colors.warning,
      danger: theme.colors.error,
    };
    return colors[variant] || theme.colors.primary;
  }};
  padding: 4px 10px;
  border-radius: 20px;
`;

const BadgeText = styled.Text`
  color: white;
  font-size: 12px;
  font-weight: 600;
`;

// ใช้งาน
const StyledComponentsDemo = () => {
  const { theme } = useTheme();

  return (
    <ThemeProvider theme={theme}>
      <ScrollView>
        <StyledButton onPress={() => {}}>
          <ButtonText>Styled Button</ButtonText>
        </StyledButton>

        <Card>
          <Title>Card Title</Title>
          <Body>Card body text ที่ใช้ styled-components</Body>
          <View style={{ flexDirection: 'row', gap: 8, marginTop: 12 }}>
            <Badge variant="primary"><BadgeText>Primary</BadgeText></Badge>
            <Badge variant="success"><BadgeText>Success</BadgeText></Badge>
            <Badge variant="danger"><BadgeText>Danger</BadgeText></Badge>
          </View>
        </Card>
      </ScrollView>
    </ThemeProvider>
  );
};
```

---

## 5. React Native Paper Themes

```bash
npx expo install react-native-paper
```

```jsx
import { Provider as PaperProvider, MD3DarkTheme, MD3LightTheme, adaptNavigationTheme } from 'react-native-paper';
import { Button, Card, Text, Chip, TextInput, FAB, Snackbar } from 'react-native-paper';

// สร้าง custom theme บน Material Design 3
const lightPaperTheme = {
  ...MD3LightTheme,
  colors: {
    ...MD3LightTheme.colors,
    primary: '#007AFF',
    secondary: '#5856D6',
    error: '#FF3B30',
    background: '#F2F2F7',
    surface: '#FFFFFF',
    onPrimary: '#FFFFFF',
    onSecondary: '#FFFFFF',
  },
};

const darkPaperTheme = {
  ...MD3DarkTheme,
  colors: {
    ...MD3DarkTheme.colors,
    primary: '#0A84FF',
    secondary: '#5E5CE6',
    error: '#FF453A',
    background: '#000000',
    surface: '#1C1C1E',
  },
};

// App Root
const App = () => {
  const { isDark } = useTheme();
  const theme = isDark ? darkPaperTheme : lightPaperTheme;

  return (
    <PaperProvider theme={theme}>
      <MainContent />
    </PaperProvider>
  );
};

// ใช้งาน Paper components
const PaperComponentsDemo = () => {
  const [text, setText] = useState('');
  const [snackbar, setSnackbar] = useState(false);

  return (
    <ScrollView style={{ flex: 1, padding: 16 }}>
      {/* Text Input */}
      <TextInput
        label="อีเมล"
        value={text}
        onChangeText={setText}
        mode="outlined"
        style={{ marginBottom: 16 }}
      />

      {/* Buttons */}
      <Button mode="contained" onPress={() => setSnackbar(true)}>
        Contained Button
      </Button>
      <Button mode="outlined" style={{ marginTop: 8 }}>
        Outlined Button
      </Button>
      <Button mode="text">Text Button</Button>

      {/* Card */}
      <Card style={{ marginTop: 16 }}>
        <Card.Cover source={{ uri: 'https://picsum.photos/400/200' }} />
        <Card.Content>
          <Text variant="titleLarge">Card Title</Text>
          <Text variant="bodyMedium">Card content text</Text>
        </Card.Content>
        <Card.Actions>
          <Button>Cancel</Button>
          <Button mode="contained">OK</Button>
        </Card.Actions>
      </Card>

      {/* Chips */}
      <View style={{ flexDirection: 'row', flexWrap: 'wrap', gap: 8, marginTop: 16 }}>
        <Chip icon="heart" onPress={() => {}}>ชอบ</Chip>
        <Chip icon="star" selected onPress={() => {}}>ดาว</Chip>
        <Chip mode="outlined" onPress={() => {}}>Outlined</Chip>
      </View>

      {/* Snackbar */}
      <Snackbar
        visible={snackbar}
        onDismiss={() => setSnackbar(false)}
        duration={3000}
        action={{ label: 'ปิด', onPress: () => setSnackbar(false) }}
      >
        บันทึกสำเร็จแล้ว!
      </Snackbar>
    </ScrollView>
  );
};
```

---

## 6. Workshop: Light/Dark Theme Toggle

```jsx
import React, { useState, useEffect, useCallback, useContext } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  Switch,
  ScrollView,
  StyleSheet,
  Animated,
  SafeAreaView,
  StatusBar,
} from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { useColorScheme } from 'react-native';

// Theme definitions (from above)
// ... (same as defined in ThemeContext section)

// ============================================
// Theme Toggle Component
// ============================================

const ThemeToggle = ({ value, onToggle, label }) => {
  const { theme } = useTheme();
  const animValue = useRef(new Animated.Value(value ? 1 : 0)).current;

  useEffect(() => {
    Animated.spring(animValue, {
      toValue: value ? 1 : 0,
      useNativeDriver: false,
      tension: 65,
      friction: 10,
    }).start();
  }, [value]);

  const translateX = animValue.interpolate({
    inputRange: [0, 1],
    outputRange: [2, 22],
  });

  const backgroundColor = animValue.interpolate({
    inputRange: [0, 1],
    outputRange: [theme.colors.border, theme.colors.primary],
  });

  return (
    <TouchableOpacity
      style={toggleStyles.container}
      onPress={onToggle}
      activeOpacity={0.8}
    >
      {label && (
        <Text style={[toggleStyles.label, { color: theme.colors.text }]}>
          {label}
        </Text>
      )}
      <Animated.View
        style={[
          toggleStyles.track,
          { backgroundColor },
        ]}
      >
        <Animated.View
          style={[
            toggleStyles.thumb,
            { transform: [{ translateX }] },
          ]}
        />
      </Animated.View>
    </TouchableOpacity>
  );
};

const toggleStyles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
  },
  label: { fontSize: 16 },
  track: {
    width: 50,
    height: 28,
    borderRadius: 14,
    justifyContent: 'center',
  },
  thumb: {
    width: 24,
    height: 24,
    borderRadius: 12,
    backgroundColor: '#fff',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.15,
    shadowRadius: 2,
    elevation: 2,
  },
});

// ============================================
// Theme Mode Picker
// ============================================

const ThemeModeOption = ({ mode, label, icon, isSelected, onSelect }) => {
  const { theme } = useTheme();

  return (
    <TouchableOpacity
      style={[
        modeStyles.option,
        {
          backgroundColor: isSelected ? theme.colors.primary + '15' : theme.colors.surface,
          borderColor: isSelected ? theme.colors.primary : theme.colors.border,
          borderWidth: isSelected ? 2 : 1,
        },
      ]}
      onPress={onSelect}
    >
      <Text style={{ fontSize: 32, marginBottom: 8 }}>{icon}</Text>
      <Text style={[
        modeStyles.optionText,
        {
          color: isSelected ? theme.colors.primary : theme.colors.text,
          fontWeight: isSelected ? '600' : '400',
        },
      ]}>
        {label}
      </Text>
      {isSelected && (
        <View style={[modeStyles.checkmark, { backgroundColor: theme.colors.primary }]}>
          <Text style={{ color: '#fff', fontSize: 10 }}>✓</Text>
        </View>
      )}
    </TouchableOpacity>
  );
};

const modeStyles = StyleSheet.create({
  option: {
    flex: 1,
    alignItems: 'center',
    padding: 16,
    borderRadius: 12,
    position: 'relative',
    aspectRatio: 0.85,
    justifyContent: 'center',
  },
  optionText: { fontSize: 13 },
  checkmark: {
    position: 'absolute',
    top: 8,
    right: 8,
    width: 20,
    height: 20,
    borderRadius: 10,
    justifyContent: 'center',
    alignItems: 'center',
  },
});

// ============================================
// Settings Screen (Showcase)
// ============================================

const SettingsScreen = () => {
  const { theme, themeMode, setTheme, isDark } = useTheme();
  const [notifications, setNotifications] = useState(true);
  const [haptics, setHaptics] = useState(true);

  const themeModes = [
    { mode: 'light', label: 'สว่าง', icon: '☀️' },
    { mode: 'dark', label: 'มืด', icon: '🌙' },
    { mode: 'system', label: 'ระบบ', icon: '📱' },
    { mode: 'sepia', label: 'ซีเปีย', icon: '📜' },
  ];

  const settingsSections = [
    {
      title: 'หน้าตา',
      items: [
        {
          type: 'theme-picker',
        },
      ],
    },
    {
      title: 'การแจ้งเตือน',
      items: [
        {
          type: 'toggle',
          label: 'การแจ้งเตือน',
          icon: '🔔',
          value: notifications,
          onToggle: () => setNotifications(!notifications),
        },
        {
          type: 'toggle',
          label: 'Haptic Feedback',
          icon: '📳',
          value: haptics,
          onToggle: () => setHaptics(!haptics),
        },
      ],
    },
    {
      title: 'บัญชี',
      items: [
        { type: 'nav', label: 'โปรไฟล์', icon: '👤', value: 'สมชาย ไทย' },
        { type: 'nav', label: 'ความปลอดภัย', icon: '🔒' },
        { type: 'nav', label: 'ข้อมูลส่วนตัว', icon: '🛡️' },
      ],
    },
    {
      title: 'เกี่ยวกับ',
      items: [
        { type: 'info', label: 'เวอร์ชัน', value: '1.0.0' },
        { type: 'nav', label: 'นโยบายความเป็นส่วนตัว', icon: '📋' },
        { type: 'nav', label: 'เงื่อนไขการใช้งาน', icon: '📄' },
      ],
    },
  ];

  return (
    <View style={{ flex: 1, backgroundColor: theme.colors.background }}>
      <StatusBar
        barStyle={isDark ? 'light-content' : 'dark-content'}
        backgroundColor={theme.colors.background}
      />
      <SafeAreaView>
        <View style={[settingsStyles.header, { borderBottomColor: theme.colors.border }]}>
          <Text style={[settingsStyles.headerTitle, { color: theme.colors.text }]}>
            การตั้งค่า
          </Text>
        </View>
      </SafeAreaView>

      <ScrollView contentContainerStyle={{ paddingBottom: 32 }}>
        {settingsSections.map((section, si) => (
          <View key={si} style={settingsStyles.section}>
            <Text style={[settingsStyles.sectionTitle, { color: theme.colors.textTertiary }]}>
              {section.title}
            </Text>

            <View style={[
              settingsStyles.sectionContent,
              { backgroundColor: theme.colors.surface, borderColor: theme.colors.border },
            ]}>
              {section.items.map((item, ii) => {
                if (item.type === 'theme-picker') {
                  return (
                    <View key={ii} style={settingsStyles.themePicker}>
                      <Text style={[settingsStyles.pickerLabel, { color: theme.colors.text }]}>
                        โหมดสี
                      </Text>
                      <View style={{ flexDirection: 'row', gap: 8, marginTop: 12 }}>
                        {themeModes.map(({ mode, label, icon }) => (
                          <ThemeModeOption
                            key={mode}
                            mode={mode}
                            label={label}
                            icon={icon}
                            isSelected={themeMode === mode}
                            onSelect={() => setTheme(mode)}
                          />
                        ))}
                      </View>
                    </View>
                  );
                }

                if (item.type === 'toggle') {
                  return (
                    <View
                      key={ii}
                      style={[
                        settingsStyles.row,
                        ii < section.items.length - 1 && {
                          borderBottomWidth: 1,
                          borderBottomColor: theme.colors.borderLight,
                        },
                      ]}
                    >
                      <View style={settingsStyles.rowLeft}>
                        <Text style={{ fontSize: 22, marginRight: 12 }}>{item.icon}</Text>
                        <Text style={[settingsStyles.rowLabel, { color: theme.colors.text }]}>
                          {item.label}
                        </Text>
                      </View>
                      <ThemeToggle
                        value={item.value}
                        onToggle={item.onToggle}
                      />
                    </View>
                  );
                }

                return (
                  <TouchableOpacity
                    key={ii}
                    style={[
                      settingsStyles.row,
                      ii < section.items.length - 1 && {
                        borderBottomWidth: 1,
                        borderBottomColor: theme.colors.borderLight,
                      },
                    ]}
                  >
                    <View style={settingsStyles.rowLeft}>
                      {item.icon && (
                        <Text style={{ fontSize: 22, marginRight: 12 }}>{item.icon}</Text>
                      )}
                      <Text style={[settingsStyles.rowLabel, { color: theme.colors.text }]}>
                        {item.label}
                      </Text>
                    </View>
                    <View style={settingsStyles.rowRight}>
                      {item.value && (
                        <Text style={{ color: theme.colors.textTertiary, fontSize: 14 }}>
                          {item.value}
                        </Text>
                      )}
                      {item.type === 'nav' && (
                        <Text style={{ color: theme.colors.textTertiary, fontSize: 18 }}>›</Text>
                      )}
                    </View>
                  </TouchableOpacity>
                );
              })}
            </View>
          </View>
        ))}
      </ScrollView>
    </View>
  );
};

const settingsStyles = StyleSheet.create({
  header: {
    padding: 16,
    borderBottomWidth: 1,
  },
  headerTitle: {
    fontSize: 28,
    fontWeight: 'bold',
  },
  section: {
    marginTop: 24,
    paddingHorizontal: 16,
  },
  sectionTitle: {
    fontSize: 13,
    fontWeight: '600',
    textTransform: 'uppercase',
    letterSpacing: 0.5,
    marginBottom: 8,
    paddingLeft: 4,
  },
  sectionContent: {
    borderRadius: 12,
    borderWidth: 1,
    overflow: 'hidden',
  },
  row: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingVertical: 14,
    paddingHorizontal: 16,
    minHeight: 52,
  },
  rowLeft: {
    flexDirection: 'row',
    alignItems: 'center',
    flex: 1,
  },
  rowLabel: { fontSize: 16 },
  rowRight: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
  },
  themePicker: {
    padding: 16,
  },
  pickerLabel: {
    fontSize: 16,
    fontWeight: '600',
  },
});

// ============================================
// App Root
// ============================================

const FullApp = () => {
  return (
    <ThemeProvider>
      <SettingsScreen />
    </ThemeProvider>
  );
};

export default FullApp;
```

---

## Tips และ Best Practices

### 1. อย่า Hardcode Colors
```jsx
// ❌ ไม่ดี
<View style={{ backgroundColor: '#fff' }}>

// ✅ ดี
const { theme } = useTheme();
<View style={{ backgroundColor: theme.colors.surface }}>
```

### 2. รองรับ High Contrast
```jsx
const { theme } = useTheme();
const highContrast = useAccessibilityInfo(); // react-native AccessibilityInfo

const textColor = highContrast ? '#000000' : theme.colors.textSecondary;
```

### 3. StatusBar กับ Theme
```jsx
<StatusBar barStyle={isDark ? 'light-content' : 'dark-content'} />
```

### 4. Image Tinting
```jsx
// เปลี่ยนสี icon image ตาม theme
<Image
  source={require('./icon.png')}
  style={{ tintColor: theme.colors.primary }}
/>
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การสร้าง Color Palette และ Design Token system
- การจัดการ Dark Mode ด้วย `useColorScheme`
- การสร้าง ThemeContext ที่สมบูรณ์
- การใช้ styled-components
- การใช้ React Native Paper
- Workshop: Settings Screen พร้อม theme toggle

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Platform-specific Code
