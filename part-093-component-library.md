# Part 093: สร้าง Custom Component Library ใน React Native

## บทนำ

การสร้าง Component Library ของตัวเองช่วยให้ทีม reuse โค้ดได้อย่างมีประสิทธิภาพ เราจะเรียนรู้ตั้งแต่การออกแบบ architecture, ทดสอบด้วย Storybook, เขียน documentation, จนถึงการ publish ไปยัง npm

## หัวข้อที่จะเรียน

1. Library Architecture
2. Storybook for React Native
3. Documentation
4. Publishing to npm
5. Versioning
6. Workshop: Build Your Own UI Library

---

## 1. Library Architecture

### โครงสร้างโปรเจค

```
my-ui-library/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.test.tsx
│   │   │   ├── Button.stories.tsx
│   │   │   ├── Button.types.ts
│   │   │   └── index.ts
│   │   ├── Input/
│   │   ├── Card/
│   │   ├── Modal/
│   │   ├── Toast/
│   │   └── index.ts  (exports ทุก component)
│   ├── hooks/
│   │   ├── useTheme.ts
│   │   └── index.ts
│   ├── theme/
│   │   ├── tokens.ts
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   └── index.ts
│   ├── utils/
│   │   └── index.ts
│   └── index.ts  (main entry point)
├── .storybook/
├── docs/
├── package.json
├── tsconfig.json
├── babel.config.js
└── README.md
```

### package.json

```json
{
  "name": "@yourorg/ui-library",
  "version": "1.0.0",
  "description": "React Native UI component library",
  "main": "lib/index.js",
  "module": "lib/index.esm.js",
  "types": "lib/index.d.ts",
  "files": [
    "lib"
  ],
  "scripts": {
    "build": "bob build",
    "test": "jest",
    "storybook": "start-storybook -p 6006",
    "build-storybook": "build-storybook",
    "lint": "eslint src --ext .ts,.tsx",
    "type-check": "tsc --noEmit",
    "release": "semantic-release"
  },
  "peerDependencies": {
    "react": ">=18.0.0",
    "react-native": ">=0.71.0"
  },
  "devDependencies": {
    "@react-native-builder-bob/build": "^0.20.0",
    "@storybook/react-native": "^6.5.0",
    "typescript": "^5.0.0"
  },
  "react-native-builder-bob": {
    "source": "src",
    "output": "lib",
    "targets": [
      "commonjs",
      "module",
      "typescript"
    ]
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["ES2017"],
    "allowJs": false,
    "jsx": "react-native",
    "moduleResolution": "node",
    "strict": true,
    "declaration": true,
    "declarationDir": "lib",
    "outDir": "lib",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "lib"]
}
```

---

## 2. สร้าง Components

### Theme System

```typescript
// src/theme/tokens.ts
export const tokens = {
  // Colors
  colors: {
    primary: {
      50: '#E3F2FD',
      100: '#BBDEFB',
      200: '#90CAF9',
      300: '#64B5F6',
      400: '#42A5F5',
      500: '#2196F3',
      600: '#1E88E5',
      700: '#1976D2',
      800: '#1565C0',
      900: '#0D47A1',
    },
    secondary: {
      500: '#9C27B0',
    },
    success: {
      500: '#4CAF50',
    },
    error: {
      500: '#f44336',
    },
    warning: {
      500: '#FF9800',
    },
    neutral: {
      0: '#FFFFFF',
      50: '#FAFAFA',
      100: '#F5F5F5',
      200: '#EEEEEE',
      300: '#E0E0E0',
      400: '#BDBDBD',
      500: '#9E9E9E',
      600: '#757575',
      700: '#616161',
      800: '#424242',
      900: '#212121',
    },
  },

  // Spacing
  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
    xxl: 48,
  },

  // Border Radius
  borderRadius: {
    sm: 4,
    md: 8,
    lg: 12,
    xl: 16,
    full: 9999,
  },

  // Typography
  typography: {
    fontFamily: {
      regular: 'System',
      medium: 'System',
      bold: 'System',
    },
    fontSize: {
      xs: 10,
      sm: 12,
      md: 14,
      lg: 16,
      xl: 20,
      xxl: 24,
      xxxl: 32,
    },
    lineHeight: {
      xs: 14,
      sm: 18,
      md: 20,
      lg: 24,
      xl: 28,
    },
  },

  // Shadows
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
      shadowOpacity: 0.1,
      shadowRadius: 4,
      elevation: 2,
    },
    lg: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 4 },
      shadowOpacity: 0.15,
      shadowRadius: 8,
      elevation: 4,
    },
  },
};

export type Tokens = typeof tokens;
```

### ThemeContext

```typescript
// src/theme/ThemeContext.tsx
import React, { createContext, useContext, ReactNode } from 'react';
import { tokens, Tokens } from './tokens';

type Theme = Tokens & {
  isDark: boolean;
};

const lightTheme: Theme = {
  ...tokens,
  isDark: false,
};

const darkTheme: Theme = {
  ...tokens,
  isDark: true,
  colors: {
    ...tokens.colors,
    neutral: {
      0: '#121212',
      50: '#1E1E1E',
      100: '#2C2C2C',
      200: '#3D3D3D',
      300: '#4F4F4F',
      400: '#616161',
      500: '#757575',
      600: '#9E9E9E',
      700: '#BDBDBD',
      800: '#E0E0E0',
      900: '#FFFFFF',
    },
  },
};

const ThemeContext = createContext<Theme>(lightTheme);

export const ThemeProvider: React.FC<{
  children: ReactNode;
  colorScheme?: 'light' | 'dark';
}> = ({ children, colorScheme = 'light' }) => {
  const theme = colorScheme === 'dark' ? darkTheme : lightTheme;
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
};

export const useTheme = () => useContext(ThemeContext);
```

### Button Component

```typescript
// src/components/Button/Button.types.ts
import { TouchableOpacityProps, ViewStyle, TextStyle } from 'react-native';

export type ButtonVariant = 'filled' | 'outlined' | 'ghost' | 'link';
export type ButtonSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl';
export type ButtonColorScheme = 'primary' | 'secondary' | 'success' | 'error' | 'warning';

export interface ButtonProps extends TouchableOpacityProps {
  variant?: ButtonVariant;
  size?: ButtonSize;
  colorScheme?: ButtonColorScheme;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  isLoading?: boolean;
  isDisabled?: boolean;
  isFullWidth?: boolean;
  children: React.ReactNode;
  style?: ViewStyle;
  textStyle?: TextStyle;
}
```

```typescript
// src/components/Button/Button.tsx
import React from 'react';
import {
  TouchableOpacity,
  Text,
  ActivityIndicator,
  View,
  StyleSheet,
} from 'react-native';
import { ButtonProps, ButtonVariant, ButtonSize } from './Button.types';
import { useTheme } from '../../theme/ThemeContext';

const Button: React.FC<ButtonProps> = ({
  variant = 'filled',
  size = 'md',
  colorScheme = 'primary',
  leftIcon,
  rightIcon,
  isLoading = false,
  isDisabled = false,
  isFullWidth = false,
  children,
  style,
  textStyle,
  onPress,
  ...props
}) => {
  const theme = useTheme();
  const color = theme.colors[colorScheme][500];
  const colorLight = theme.colors[colorScheme][50];

  const sizeStyles = {
    xs: { paddingHorizontal: 8, paddingVertical: 4, fontSize: 12, height: 28 },
    sm: { paddingHorizontal: 12, paddingVertical: 6, fontSize: 13, height: 32 },
    md: { paddingHorizontal: 16, paddingVertical: 10, fontSize: 14, height: 40 },
    lg: { paddingHorizontal: 20, paddingVertical: 12, fontSize: 16, height: 48 },
    xl: { paddingHorizontal: 24, paddingVertical: 14, fontSize: 18, height: 56 },
  };

  const getVariantStyle = () => {
    switch (variant) {
      case 'filled':
        return {
          container: { backgroundColor: isDisabled ? theme.colors.neutral[300] : color },
          text: { color: 'white' },
        };
      case 'outlined':
        return {
          container: {
            backgroundColor: 'transparent',
            borderWidth: 1.5,
            borderColor: isDisabled ? theme.colors.neutral[300] : color,
          },
          text: { color: isDisabled ? theme.colors.neutral[400] : color },
        };
      case 'ghost':
        return {
          container: {
            backgroundColor: 'transparent',
          },
          text: { color: isDisabled ? theme.colors.neutral[400] : color },
        };
      case 'link':
        return {
          container: { backgroundColor: 'transparent' },
          text: {
            color: isDisabled ? theme.colors.neutral[400] : color,
            textDecorationLine: 'underline' as const,
          },
        };
    }
  };

  const variantStyle = getVariantStyle();
  const sizeStyle = sizeStyles[size];

  return (
    <TouchableOpacity
      style={[
        styles.base,
        {
          paddingHorizontal: sizeStyle.paddingHorizontal,
          paddingVertical: sizeStyle.paddingVertical,
          height: sizeStyle.height,
          borderRadius: theme.borderRadius.md,
        },
        variantStyle.container,
        isFullWidth && styles.fullWidth,
        (isDisabled || isLoading) && styles.disabled,
        style,
      ]}
      onPress={onPress}
      disabled={isDisabled || isLoading}
      activeOpacity={0.7}
      {...props}
    >
      {isLoading ? (
        <ActivityIndicator
          size="small"
          color={variant === 'filled' ? 'white' : color}
        />
      ) : (
        <View style={styles.content}>
          {leftIcon && <View style={styles.iconLeft}>{leftIcon}</View>}
          <Text
            style={[
              styles.text,
              { fontSize: sizeStyle.fontSize },
              variantStyle.text,
              textStyle,
            ]}
          >
            {children}
          </Text>
          {rightIcon && <View style={styles.iconRight}>{rightIcon}</View>}
        </View>
      )}
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  base: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    borderRadius: 8,
  },
  content: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  text: {
    fontWeight: '600',
  },
  iconLeft: { marginRight: 6 },
  iconRight: { marginLeft: 6 },
  fullWidth: { width: '100%' },
  disabled: { opacity: 0.5 },
});

export default Button;
```

```typescript
// src/components/Button/index.ts
export { default } from './Button';
export type { ButtonProps, ButtonVariant, ButtonSize, ButtonColorScheme } from './Button.types';
```

### Input Component

```typescript
// src/components/Input/Input.tsx
import React, { useState, forwardRef } from 'react';
import {
  TextInput,
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  TextInputProps,
} from 'react-native';
import { useTheme } from '../../theme/ThemeContext';

interface InputProps extends TextInputProps {
  label?: string;
  helperText?: string;
  errorMessage?: string;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  isDisabled?: boolean;
  isRequired?: boolean;
  variant?: 'outlined' | 'filled' | 'flushed';
}

const Input = forwardRef<TextInput, InputProps>(({
  label,
  helperText,
  errorMessage,
  leftIcon,
  rightIcon,
  isDisabled = false,
  isRequired = false,
  variant = 'outlined',
  style,
  ...props
}, ref) => {
  const theme = useTheme();
  const [isFocused, setIsFocused] = useState(false);
  const hasError = !!errorMessage;

  const getBorderColor = () => {
    if (hasError) return theme.colors.error[500];
    if (isFocused) return theme.colors.primary[500];
    return theme.colors.neutral[300];
  };

  const getVariantStyle = () => {
    switch (variant) {
      case 'filled':
        return {
          backgroundColor: isFocused ? theme.colors.neutral[100] : theme.colors.neutral[100],
          borderWidth: 0,
          borderBottomWidth: 2,
          borderBottomColor: getBorderColor(),
          borderRadius: 8,
        };
      case 'flushed':
        return {
          backgroundColor: 'transparent',
          borderWidth: 0,
          borderBottomWidth: 1,
          borderBottomColor: getBorderColor(),
          borderRadius: 0,
          paddingHorizontal: 0,
        };
      default: // outlined
        return {
          backgroundColor: 'white',
          borderWidth: 1.5,
          borderColor: getBorderColor(),
          borderRadius: 8,
        };
    }
  };

  return (
    <View style={styles.container}>
      {label && (
        <Text style={[styles.label, { color: theme.colors.neutral[700] }]}>
          {label}
          {isRequired && <Text style={{ color: theme.colors.error[500] }}> *</Text>}
        </Text>
      )}

      <View style={[styles.inputWrapper, getVariantStyle()]}>
        {leftIcon && <View style={styles.iconLeft}>{leftIcon}</View>}

        <TextInput
          ref={ref}
          style={[
            styles.input,
            { color: theme.colors.neutral[900] },
            isDisabled && styles.disabledInput,
            style,
          ]}
          placeholderTextColor={theme.colors.neutral[400]}
          editable={!isDisabled}
          onFocus={() => setIsFocused(true)}
          onBlur={() => setIsFocused(false)}
          {...props}
        />

        {rightIcon && <View style={styles.iconRight}>{rightIcon}</View>}
      </View>

      {(helperText || errorMessage) && (
        <Text
          style={[
            styles.helperText,
            { color: hasError ? theme.colors.error[500] : theme.colors.neutral[500] },
          ]}
        >
          {errorMessage || helperText}
        </Text>
      )}
    </View>
  );
});

const styles = StyleSheet.create({
  container: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '500', marginBottom: 6 },
  inputWrapper: {
    flexDirection: 'row', alignItems: 'center', overflow: 'hidden',
  },
  input: { flex: 1, fontSize: 15, paddingHorizontal: 12, paddingVertical: 10, minHeight: 44 },
  disabledInput: { opacity: 0.5 },
  iconLeft: { paddingLeft: 12 },
  iconRight: { paddingRight: 12 },
  helperText: { fontSize: 12, marginTop: 4 },
});

export default Input;
```

---

## 3. Storybook Setup

```bash
npx storybook@latest init --type react_native
```

### .storybook/main.js

```javascript
module.exports = {
  stories: ['../src/**/*.stories.?(ts|tsx|js|jsx)'],
  addons: [
    '@storybook/addon-controls',
    '@storybook/addon-actions',
    '@storybook/addon-knobs',
    '@storybook/addon-docs',
  ],
};
```

### Button.stories.tsx

```typescript
import React from 'react';
import { Meta, Story } from '@storybook/react-native';
import Button from './Button';
import { ButtonProps } from './Button.types';

const meta: Meta<ButtonProps> = {
  title: 'Components/Button',
  component: Button,
  argTypes: {
    variant: {
      control: { type: 'select' },
      options: ['filled', 'outlined', 'ghost', 'link'],
    },
    size: {
      control: { type: 'select' },
      options: ['xs', 'sm', 'md', 'lg', 'xl'],
    },
    colorScheme: {
      control: { type: 'select' },
      options: ['primary', 'secondary', 'success', 'error', 'warning'],
    },
  },
};

export default meta;

const Template: Story<ButtonProps> = (args) => <Button {...args} />;

export const Primary = Template.bind({});
Primary.args = {
  children: 'Primary Button',
  variant: 'filled',
  colorScheme: 'primary',
  size: 'md',
};

export const Outlined = Template.bind({});
Outlined.args = {
  children: 'Outlined Button',
  variant: 'outlined',
};

export const Loading = Template.bind({});
Loading.args = {
  children: 'Loading...',
  isLoading: true,
};

export const AllVariants: Story = () => (
  <View style={{ gap: 12, padding: 16 }}>
    <Button variant="filled">Filled</Button>
    <Button variant="outlined">Outlined</Button>
    <Button variant="ghost">Ghost</Button>
    <Button variant="link">Link</Button>
  </View>
);
```

---

## 4. การ Publish ไปยัง npm

### ตั้งค่า package.json สำหรับ publish

```json
{
  "name": "@yourscope/ui-library",
  "version": "1.0.0",
  "publishConfig": {
    "registry": "https://registry.npmjs.org",
    "access": "public"
  }
}
```

### .npmignore

```
src/
.storybook/
__tests__/
docs/
*.test.ts
*.test.tsx
*.stories.ts
*.stories.tsx
```

### scripts/publish.sh

```bash
#!/bin/bash
# Build library
npm run build

# Run tests
npm test

# Type check
npm run type-check

# Publish to npm
npm publish --access public

echo "Published successfully! 🚀"
```

---

## 5. Semantic Versioning

### commitlint.config.js

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2, 'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'test', 'chore', 'breaking']
    ],
  },
};
```

### CHANGELOG.md format

```markdown
# Changelog

## [1.1.0] - 2024-01-15
### Added
- เพิ่ม `Toast` component
- เพิ่ม `Modal` component
- Support สำหรับ dark mode

### Changed
- ปรับปรุง `Button` sizing

### Fixed
- แก้ไข `Input` focus state บน Android

## [1.0.0] - 2024-01-01
### Added
- Initial release
- `Button`, `Input`, `Card` components
```

---

## 6. Main Index

```typescript
// src/index.ts
// Components
export { default as Button } from './components/Button';
export type { ButtonProps } from './components/Button';

export { default as Input } from './components/Input';
export type { InputProps } from './components/Input';

// Theme
export { ThemeProvider, useTheme } from './theme/ThemeContext';
export { tokens } from './theme/tokens';
export type { Tokens } from './theme/tokens';

// Hooks
export * from './hooks';
```

---

## Workshop Exercises

1. **Badge Component** - สร้าง notification badge
2. **Skeleton Component** - loading placeholder
3. **Tooltip Component** - hover/press tooltip
4. **Select Component** - dropdown selector

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Architecture** - โครงสร้าง library ที่ดี
2. **Component Design** - สร้าง flexible components
3. **Storybook** - document และ test components
4. **Publishing** - เผยแพร่ library ไปยัง npm
5. **Versioning** - จัดการ versions อย่างมีระบบ
