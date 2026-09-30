# Part 094: Design System ใน React Native

## บทนำ

Design System เป็นชุดของ guidelines, components และ tools ที่ช่วยให้ทีม design และ development ทำงานร่วมกันอย่างสอดคล้อง เราจะสร้าง complete design system ตั้งแต่ design tokens จนถึง component variants

## หัวข้อที่จะเรียน

1. Design Tokens
2. Component Variants
3. Spacing System
4. Typography System
5. Color System
6. Workshop: Complete Design System

---

## 1. Design Tokens

Design Tokens เป็น named values ที่แทน design decisions

### tokens/base.ts

```typescript
// ค่า primitive ที่ไม่ขึ้นกับ brand
export const base = {
  // Font families
  fontFamily: {
    sans: "'Noto Sans Thai', 'Inter', 'System', sans-serif",
    mono: "'JetBrains Mono', 'Courier New', monospace",
  },

  // Base sizes
  size: {
    0: 0,
    0.5: 2,
    1: 4,
    1.5: 6,
    2: 8,
    2.5: 10,
    3: 12,
    3.5: 14,
    4: 16,
    5: 20,
    6: 24,
    7: 28,
    8: 32,
    9: 36,
    10: 40,
    12: 48,
    14: 56,
    16: 64,
    20: 80,
    24: 96,
    28: 112,
    32: 128,
  },

  // Base colors (palette)
  color: {
    // Blue
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

    // Red
    red50: '#FFF1F2',
    red100: '#FFE4E6',
    red400: '#F87171',
    red500: '#EF4444',
    red600: '#DC2626',
    red700: '#B91C1C',

    // Green
    green50: '#F0FDF4',
    green100: '#DCFCE7',
    green400: '#4ADE80',
    green500: '#22C55E',
    green600: '#16A34A',

    // Yellow
    yellow50: '#FFFBEB',
    yellow400: '#FBBF24',
    yellow500: '#F59E0B',

    // Gray
    gray0: '#FFFFFF',
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
    gray950: '#030712',
  },

  // Border radius
  radius: {
    none: 0,
    sm: 2,
    base: 4,
    md: 6,
    lg: 8,
    xl: 12,
    '2xl': 16,
    '3xl': 24,
    full: 9999,
  },

  // Shadows
  shadow: {
    xs: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 1 },
      shadowOpacity: 0.04,
      shadowRadius: 2,
      elevation: 1,
    },
    sm: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 1 },
      shadowOpacity: 0.07,
      shadowRadius: 3,
      elevation: 2,
    },
    md: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 4 },
      shadowOpacity: 0.1,
      shadowRadius: 6,
      elevation: 4,
    },
    lg: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 8 },
      shadowOpacity: 0.12,
      shadowRadius: 12,
      elevation: 8,
    },
    xl: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 16 },
      shadowOpacity: 0.15,
      shadowRadius: 24,
      elevation: 16,
    },
  },
};
```

### tokens/semantic.ts

```typescript
import { base } from './base';

// Semantic tokens - แทน design decisions
export const createSemanticTokens = (isDark: boolean) => ({
  // Colors
  color: {
    // Brand
    brand: {
      primary: isDark ? base.color.blue400 : base.color.blue600,
      primaryHover: isDark ? base.color.blue300 : base.color.blue700,
      primarySubtle: isDark ? `${base.color.blue400}20` : base.color.blue50,
    },

    // Status
    status: {
      success: isDark ? base.color.green400 : base.color.green600,
      successSubtle: isDark ? `${base.color.green400}20` : base.color.green50,
      error: isDark ? base.color.red400 : base.color.red600,
      errorSubtle: isDark ? `${base.color.red400}20` : base.color.red50,
      warning: isDark ? base.color.yellow400 : base.color.yellow500,
      warningSubtle: isDark ? `${base.color.yellow400}20` : base.color.yellow50,
    },

    // Text
    text: {
      primary: isDark ? base.color.gray50 : base.color.gray900,
      secondary: isDark ? base.color.gray400 : base.color.gray600,
      tertiary: isDark ? base.color.gray500 : base.color.gray500,
      disabled: isDark ? base.color.gray600 : base.color.gray400,
      inverse: isDark ? base.color.gray900 : base.color.gray50,
      onPrimary: '#FFFFFF',
    },

    // Background
    bg: {
      default: isDark ? base.color.gray900 : base.color.gray0,
      subtle: isDark ? base.color.gray800 : base.color.gray50,
      muted: isDark ? base.color.gray700 : base.color.gray100,
      elevated: isDark ? base.color.gray800 : base.color.gray0,
      overlay: isDark ? 'rgba(0,0,0,0.7)' : 'rgba(0,0,0,0.5)',
    },

    // Border
    border: {
      default: isDark ? base.color.gray700 : base.color.gray200,
      strong: isDark ? base.color.gray600 : base.color.gray300,
      focus: isDark ? base.color.blue400 : base.color.blue600,
    },
  },

  // Spacing
  space: {
    '1': base.size[1],
    '2': base.size[2],
    '3': base.size[3],
    '4': base.size[4],
    '5': base.size[5],
    '6': base.size[6],
    '8': base.size[8],
    '10': base.size[10],
    '12': base.size[12],
    '16': base.size[16],
  },

  // Radius
  radius: base.radius,

  // Shadows
  shadow: base.shadow,
});

export type SemanticTokens = ReturnType<typeof createSemanticTokens>;
```

---

## 2. Typography System

### tokens/typography.ts

```typescript
export const typography = {
  // Type scale (Major Third - 1.25 ratio)
  scale: {
    '2xs': 10,
    xs: 12,
    sm: 14,
    md: 16,
    lg: 18,
    xl: 20,
    '2xl': 24,
    '3xl': 30,
    '4xl': 36,
    '5xl': 48,
  },

  // Font weights
  weight: {
    thin: '100' as const,
    light: '300' as const,
    regular: '400' as const,
    medium: '500' as const,
    semibold: '600' as const,
    bold: '700' as const,
    extrabold: '800' as const,
    black: '900' as const,
  },

  // Line heights
  leading: {
    tight: 1.2,
    snug: 1.375,
    normal: 1.5,
    relaxed: 1.625,
    loose: 2,
  },

  // Letter spacing
  tracking: {
    tight: -0.5,
    normal: 0,
    wide: 0.5,
    wider: 1,
    widest: 2,
  },
};

// Predefined text styles
export const textStyles = {
  // Display
  displayLg: {
    fontSize: typography.scale['5xl'],
    fontWeight: typography.weight.bold,
    lineHeight: typography.scale['5xl'] * typography.leading.tight,
    letterSpacing: typography.tracking.tight,
  },
  displayMd: {
    fontSize: typography.scale['4xl'],
    fontWeight: typography.weight.bold,
    lineHeight: typography.scale['4xl'] * typography.leading.tight,
  },

  // Headings
  h1: {
    fontSize: typography.scale['3xl'],
    fontWeight: typography.weight.bold,
    lineHeight: typography.scale['3xl'] * typography.leading.snug,
  },
  h2: {
    fontSize: typography.scale['2xl'],
    fontWeight: typography.weight.semibold,
    lineHeight: typography.scale['2xl'] * typography.leading.snug,
  },
  h3: {
    fontSize: typography.scale.xl,
    fontWeight: typography.weight.semibold,
    lineHeight: typography.scale.xl * typography.leading.normal,
  },
  h4: {
    fontSize: typography.scale.lg,
    fontWeight: typography.weight.semibold,
    lineHeight: typography.scale.lg * typography.leading.normal,
  },

  // Body
  bodyLg: {
    fontSize: typography.scale.md,
    fontWeight: typography.weight.regular,
    lineHeight: typography.scale.md * typography.leading.relaxed,
  },
  bodyMd: {
    fontSize: typography.scale.sm,
    fontWeight: typography.weight.regular,
    lineHeight: typography.scale.sm * typography.leading.relaxed,
  },
  bodySm: {
    fontSize: typography.scale.xs,
    fontWeight: typography.weight.regular,
    lineHeight: typography.scale.xs * typography.leading.normal,
  },

  // Labels
  labelLg: {
    fontSize: typography.scale.sm,
    fontWeight: typography.weight.medium,
    lineHeight: typography.scale.sm * typography.leading.normal,
    letterSpacing: typography.tracking.wide,
  },
  labelMd: {
    fontSize: typography.scale.xs,
    fontWeight: typography.weight.medium,
    lineHeight: typography.scale.xs * typography.leading.normal,
  },
  labelSm: {
    fontSize: typography.scale['2xs'],
    fontWeight: typography.weight.medium,
    lineHeight: typography.scale['2xs'] * typography.leading.normal,
  },
};
```

### components/Text.tsx

```typescript
import React from 'react';
import { Text as RNText, TextProps as RNTextProps, StyleSheet } from 'react-native';
import { textStyles } from '../tokens/typography';
import { useDesignSystem } from '../DesignSystemContext';

type TextVariant = keyof typeof textStyles;

interface TextProps extends RNTextProps {
  variant?: TextVariant;
  color?: string;
  align?: 'left' | 'center' | 'right' | 'justify';
  numberOfLines?: number;
}

const Text: React.FC<TextProps> = ({
  variant = 'bodyMd',
  color,
  align,
  style,
  children,
  ...props
}) => {
  const { tokens } = useDesignSystem();
  const variantStyle = textStyles[variant];
  
  return (
    <RNText
      style={[
        variantStyle,
        {
          color: color || tokens.color.text.primary,
          textAlign: align,
        },
        style,
      ]}
      {...props}
    >
      {children}
    </RNText>
  );
};

export default Text;
```

---

## 3. Color System

### tokens/colors.ts

```typescript
// Brand colors สำหรับ company ของคุณ
export const brandColors = {
  primary: {
    50: '#EEF2FF',
    100: '#E0E7FF',
    200: '#C7D2FE',
    300: '#A5B4FC',
    400: '#818CF8',
    500: '#6366F1', // Main brand color
    600: '#4F46E5',
    700: '#4338CA',
    800: '#3730A3',
    900: '#312E81',
  },
  
  accent: {
    500: '#EC4899', // Pink accent
  },
};

// Functional colors
export const functionalColors = {
  success: {
    bg: '#F0FDF4',
    border: '#BBF7D0',
    text: '#166534',
    icon: '#22C55E',
  },
  error: {
    bg: '#FFF1F2',
    border: '#FECDD3',
    text: '#9F1239',
    icon: '#F43F5E',
  },
  warning: {
    bg: '#FFFBEB',
    border: '#FDE68A',
    text: '#92400E',
    icon: '#F59E0B',
  },
  info: {
    bg: '#EFF6FF',
    border: '#BFDBFE',
    text: '#1E40AF',
    icon: '#3B82F6',
  },
};
```

### components/Badge.tsx

```typescript
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { functionalColors } from '../tokens/colors';

type BadgeVariant = 'success' | 'error' | 'warning' | 'info' | 'neutral';
type BadgeSize = 'sm' | 'md' | 'lg';

interface BadgeProps {
  label: string;
  variant?: BadgeVariant;
  size?: BadgeSize;
  dot?: boolean;
}

const Badge: React.FC<BadgeProps> = ({
  label,
  variant = 'neutral',
  size = 'md',
  dot = false,
}) => {
  const colors = variant === 'neutral'
    ? { bg: '#F3F4F6', border: '#D1D5DB', text: '#4B5563', icon: '#6B7280' }
    : functionalColors[variant];

  const sizes = {
    sm: { paddingH: 6, paddingV: 2, fontSize: 10, dotSize: 6 },
    md: { paddingH: 8, paddingV: 3, fontSize: 12, dotSize: 7 },
    lg: { paddingH: 10, paddingV: 4, fontSize: 13, dotSize: 8 },
  };

  const s = sizes[size];

  return (
    <View
      style={[
        styles.badge,
        {
          paddingHorizontal: s.paddingH,
          paddingVertical: s.paddingV,
          backgroundColor: colors.bg,
          borderColor: colors.border,
        },
      ]}
    >
      {dot && (
        <View
          style={[
            styles.dot,
            {
              width: s.dotSize,
              height: s.dotSize,
              backgroundColor: colors.icon,
            },
          ]}
        />
      )}
      <Text style={[styles.label, { fontSize: s.fontSize, color: colors.text }]}>
        {label}
      </Text>
    </View>
  );
};

const styles = StyleSheet.create({
  badge: {
    flexDirection: 'row',
    alignItems: 'center',
    borderRadius: 100,
    borderWidth: 1,
    alignSelf: 'flex-start',
    gap: 4,
  },
  dot: { borderRadius: 100 },
  label: { fontWeight: '500' },
});

export default Badge;
```

---

## 4. Design System Provider

### DesignSystemContext.tsx

```typescript
import React, { createContext, useContext, ReactNode, useMemo } from 'react';
import { useColorScheme } from 'react-native';
import { createSemanticTokens, SemanticTokens } from './tokens/semantic';
import { typography, textStyles } from './tokens/typography';
import { brandColors } from './tokens/colors';

interface DesignSystemContextType {
  tokens: SemanticTokens;
  typography: typeof typography;
  textStyles: typeof textStyles;
  isDark: boolean;
  colorScheme: 'light' | 'dark';
}

const DesignSystemContext = createContext<DesignSystemContextType | null>(null);

interface DesignSystemProviderProps {
  children: ReactNode;
  colorScheme?: 'light' | 'dark' | 'system';
  customTokens?: Partial<SemanticTokens>;
}

export const DesignSystemProvider: React.FC<DesignSystemProviderProps> = ({
  children,
  colorScheme: propColorScheme = 'system',
  customTokens,
}) => {
  const systemColorScheme = useColorScheme();
  
  const activeColorScheme = useMemo(() => {
    if (propColorScheme === 'system') {
      return systemColorScheme || 'light';
    }
    return propColorScheme;
  }, [propColorScheme, systemColorScheme]);

  const isDark = activeColorScheme === 'dark';

  const tokens = useMemo(() => {
    const baseTokens = createSemanticTokens(isDark);
    if (customTokens) {
      return deepMerge(baseTokens, customTokens);
    }
    return baseTokens;
  }, [isDark, customTokens]);

  const value = useMemo(() => ({
    tokens,
    typography,
    textStyles,
    isDark,
    colorScheme: activeColorScheme,
  }), [tokens, isDark, activeColorScheme]);

  return (
    <DesignSystemContext.Provider value={value}>
      {children}
    </DesignSystemContext.Provider>
  );
};

export const useDesignSystem = (): DesignSystemContextType => {
  const context = useContext(DesignSystemContext);
  if (!context) {
    throw new Error('useDesignSystem must be used within DesignSystemProvider');
  }
  return context;
};

// Helper
const deepMerge = <T extends object>(target: T, source: Partial<T>): T => {
  const output = { ...target };
  for (const key in source) {
    if (source[key] instanceof Object && !Array.isArray(source[key])) {
      output[key] = deepMerge(target[key] as any, source[key] as any);
    } else {
      output[key] = source[key] as any;
    }
  }
  return output;
};
```

---

## 5. Design System Demo Screen

### screens/DesignSystemDemo.tsx

```typescript
import React, { useState } from 'react';
import { ScrollView, View, StyleSheet } from 'react-native';
import { useDesignSystem } from '../DesignSystemContext';
import Text from '../components/Text';
import Button from '../components/Button';
import Badge from '../components/Badge';
import Input from '../components/Input';

const DesignSystemDemo: React.FC = () => {
  const { tokens } = useDesignSystem();
  const [inputValue, setInputValue] = useState('');

  return (
    <ScrollView
      style={[styles.container, { backgroundColor: tokens.color.bg.subtle }]}
    >
      {/* Typography */}
      <View style={[styles.section, { backgroundColor: tokens.color.bg.default }]}>
        <Text variant="h2">Typography</Text>
        
        {(['displayMd', 'h1', 'h2', 'h3', 'h4', 'bodyLg', 'bodyMd', 'bodySm'] as const).map(v => (
          <View key={v} style={styles.typeRow}>
            <Text variant="labelSm" color={tokens.color.text.tertiary}>
              {v}
            </Text>
            <Text variant={v}>สวัสดี Hello World</Text>
          </View>
        ))}
      </View>

      {/* Colors */}
      <View style={[styles.section, { backgroundColor: tokens.color.bg.default }]}>
        <Text variant="h2">Colors</Text>
        
        <View style={styles.colorGrid}>
          {Object.entries(tokens.color.brand).map(([key, value]) => (
            <View key={key} style={styles.colorItem}>
              <View style={[styles.colorSwatch, { backgroundColor: value }]} />
              <Text variant="labelSm">{key}</Text>
            </View>
          ))}
        </View>
      </View>

      {/* Buttons */}
      <View style={[styles.section, { backgroundColor: tokens.color.bg.default }]}>
        <Text variant="h2" style={styles.sectionTitle}>Buttons</Text>
        
        <Text variant="labelMd" color={tokens.color.text.secondary} style={styles.subsectionTitle}>
          Variants
        </Text>
        <View style={styles.buttonRow}>
          <Button variant="filled">Filled</Button>
          <Button variant="outlined">Outlined</Button>
          <Button variant="ghost">Ghost</Button>
        </View>
        
        <Text variant="labelMd" color={tokens.color.text.secondary} style={styles.subsectionTitle}>
          Sizes
        </Text>
        <View style={styles.buttonColumn}>
          {(['xs', 'sm', 'md', 'lg', 'xl'] as const).map(size => (
            <Button key={size} size={size}>Button {size}</Button>
          ))}
        </View>
        
        <Text variant="labelMd" color={tokens.color.text.secondary} style={styles.subsectionTitle}>
          Colors
        </Text>
        <View style={styles.buttonRow}>
          {(['primary', 'success', 'error', 'warning'] as const).map(color => (
            <Button key={color} colorScheme={color} size="sm">{color}</Button>
          ))}
        </View>
      </View>

      {/* Badges */}
      <View style={[styles.section, { backgroundColor: tokens.color.bg.default }]}>
        <Text variant="h2" style={styles.sectionTitle}>Badges</Text>
        
        <View style={styles.badgeRow}>
          <Badge label="Success" variant="success" dot />
          <Badge label="Error" variant="error" dot />
          <Badge label="Warning" variant="warning" dot />
          <Badge label="Info" variant="info" dot />
          <Badge label="Neutral" variant="neutral" />
        </View>
        
        <View style={styles.badgeRow}>
          <Badge label="Small" size="sm" variant="success" />
          <Badge label="Medium" size="md" variant="success" />
          <Badge label="Large" size="lg" variant="success" />
        </View>
      </View>

      {/* Inputs */}
      <View style={[styles.section, { backgroundColor: tokens.color.bg.default }]}>
        <Text variant="h2" style={styles.sectionTitle}>Inputs</Text>
        
        <Input
          label="Outlined (default)"
          placeholder="กรอกข้อมูล..."
          value={inputValue}
          onChangeText={setInputValue}
        />
        <Input
          label="Filled"
          variant="filled"
          placeholder="กรอกข้อมูล..."
        />
        <Input
          label="With Error"
          errorMessage="กรุณากรอกข้อมูลให้ถูกต้อง"
          value=""
        />
        <Input
          label="With Helper"
          helperText="ข้อความช่วยเหลือ"
          placeholder="พิมพ์ที่นี่..."
        />
      </View>

      {/* Spacing */}
      <View style={[styles.section, { backgroundColor: tokens.color.bg.default }]}>
        <Text variant="h2" style={styles.sectionTitle}>Spacing</Text>
        
        {Object.entries(tokens.space).map(([key, value]) => (
          <View key={key} style={styles.spacingRow}>
            <Text variant="labelSm" color={tokens.color.text.tertiary} style={styles.spacingLabel}>
              space.{key} = {value}px
            </Text>
            <View
              style={[
                styles.spacingBar,
                {
                  width: value * 2,
                  backgroundColor: tokens.color.brand.primary,
                }
              ]}
            />
          </View>
        ))}
      </View>

      <View style={{ height: 40 }} />
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  section: {
    margin: 16, marginBottom: 0, padding: 20, borderRadius: 12,
    shadowColor: '#000', shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.05, shadowRadius: 3, elevation: 1,
  },
  sectionTitle: { marginBottom: 16 },
  subsectionTitle: { marginTop: 12, marginBottom: 8 },
  typeRow: { marginBottom: 12, borderBottomWidth: 1, borderBottomColor: '#f0f0f0', paddingBottom: 12 },
  colorGrid: { flexDirection: 'row', flexWrap: 'wrap', gap: 12 },
  colorItem: { alignItems: 'center', gap: 4 },
  colorSwatch: { width: 48, height: 48, borderRadius: 8 },
  buttonRow: { flexDirection: 'row', gap: 8, flexWrap: 'wrap' },
  buttonColumn: { gap: 8 },
  badgeRow: { flexDirection: 'row', gap: 8, flexWrap: 'wrap', marginBottom: 8 },
  spacingRow: { flexDirection: 'row', alignItems: 'center', marginBottom: 8, gap: 12 },
  spacingLabel: { width: 130 },
  spacingBar: { height: 16, borderRadius: 4 },
});

export default DesignSystemDemo;
```

---

## Workshop Exercises

1. **Custom Brand** - ปรับ design system ให้ใช้ brand ของคุณเอง
2. **Dark Mode** - สร้าง toggle สลับ light/dark mode
3. **Theme Extension** - extend design system ด้วย custom tokens
4. **Component Audit** - ตรวจสอบว่า components ใช้ tokens อย่างสม่ำเสมอ

---

## สรุป

ในบทนี้เราได้สร้าง complete design system:
1. **Design Tokens** - base + semantic tokens
2. **Typography** - scale, weights, line heights
3. **Color System** - brand, functional, semantic colors
4. **Components** - Text, Button, Badge, Input
5. **Provider** - context สำหรับ theme switching
