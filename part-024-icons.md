# Part 024: Icons และ Vector Icons

## บทนำ

Icons เป็นองค์ประกอบสำคัญของ UI ที่ช่วยให้ผู้ใช้เข้าใจ app ได้เร็วขึ้น ใน React Native มีหลายวิธีในการใช้ icons ตั้งแต่ emoji ธรรมดา ไปจนถึง vector icons และ custom SVG icons

## สารบัญ

1. react-native-vector-icons
2. MaterialIcons, FontAwesome, Ionicons
3. Custom SVG Icons
4. react-native-svg
5. Icon sizes และ colors
6. Workshop: Icon Showcase App

---

## 1. react-native-vector-icons

### ติดตั้ง

```bash
# สำหรับ Expo
npx expo install @expo/vector-icons

# สำหรับ bare React Native
npm install react-native-vector-icons
```

### Icon sets ที่มีให้ใช้

| Library | จำนวน Icons | เหมาะสำหรับ |
|---------|-------------|-------------|
| AntDesign | 297 | Ant Design UI |
| Entypo | 411 | General purpose |
| EvilIcons | 70 | Simple UI |
| Feather | 286 | Clean/minimal |
| FontAwesome | 675 | Web icons |
| FontAwesome5 | 1,600+ | Modern version |
| Foundation | 283 | Zurb Foundation |
| Ionicons | 1,300+ | iOS/Material hybrid |
| MaterialIcons | 1,100+ | Material Design |
| MaterialCommunityIcons | 6,000+ | Material extended |
| SimpleLineIcons | 189 | Simple lines |
| Octicons | 250+ | GitHub style |
| Zocial | 100 | Social media |

```jsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import {
  AntDesign,
  Feather,
  FontAwesome,
  Ionicons,
  MaterialIcons,
  MaterialCommunityIcons,
} from '@expo/vector-icons';

const IconLibraryDemo = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Icon Libraries</Text>
      
      <View style={styles.row}>
        <AntDesign name="home" size={32} color="#007AFF" />
        <AntDesign name="heart" size={32} color="#FF3B30" />
        <AntDesign name="star" size={32} color="#FF9500" />
        <AntDesign name="search1" size={32} color="#34C759" />
      </View>

      <View style={styles.row}>
        <Feather name="home" size={32} color="#007AFF" />
        <Feather name="heart" size={32} color="#FF3B30" />
        <Feather name="star" size={32} color="#FF9500" />
        <Feather name="search" size={32} color="#34C759" />
      </View>

      <View style={styles.row}>
        <MaterialIcons name="home" size={32} color="#007AFF" />
        <MaterialIcons name="favorite" size={32} color="#FF3B30" />
        <MaterialIcons name="star" size={32} color="#FF9500" />
        <MaterialIcons name="search" size={32} color="#34C759" />
      </View>

      <View style={styles.row}>
        <Ionicons name="home" size={32} color="#007AFF" />
        <Ionicons name="heart" size={32} color="#FF3B30" />
        <Ionicons name="star" size={32} color="#FF9500" />
        <Ionicons name="search" size={32} color="#34C759" />
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 20, backgroundColor: '#fff' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 24 },
  row: {
    flexDirection: 'row',
    gap: 24,
    marginBottom: 24,
    alignItems: 'center',
  },
});
```

---

## 2. MaterialIcons, FontAwesome, Ionicons

### MaterialIcons

Material Design icons โดย Google - เหมาะสำหรับ Android-style UI

```jsx
import { MaterialIcons } from '@expo/vector-icons';

// Navigation icons
const NavigationIcons = () => (
  <View style={{ flexDirection: 'row', gap: 16 }}>
    <MaterialIcons name="home" size={24} color="#333" />
    <MaterialIcons name="search" size={24} color="#333" />
    <MaterialIcons name="notifications" size={24} color="#333" />
    <MaterialIcons name="person" size={24} color="#333" />
    <MaterialIcons name="settings" size={24} color="#333" />
  </View>
);

// Action icons
const ActionIcons = () => (
  <View style={{ flexDirection: 'row', gap: 16 }}>
    <MaterialIcons name="add" size={24} color="#007AFF" />
    <MaterialIcons name="edit" size={24} color="#007AFF" />
    <MaterialIcons name="delete" size={24} color="#FF3B30" />
    <MaterialIcons name="share" size={24} color="#34C759" />
    <MaterialIcons name="favorite" size={24} color="#FF3B30" />
  </View>
);

// Media icons
const MediaIcons = () => (
  <View style={{ flexDirection: 'row', gap: 16 }}>
    <MaterialIcons name="play-circle-filled" size={32} color="#007AFF" />
    <MaterialIcons name="pause-circle-filled" size={32} color="#007AFF" />
    <MaterialIcons name="skip-next" size={32} color="#007AFF" />
    <MaterialIcons name="volume-up" size={32} color="#007AFF" />
  </View>
);
```

### FontAwesome 5

```jsx
import { FontAwesome5 } from '@expo/vector-icons';

// Regular icons
const FARegular = () => (
  <FontAwesome5 name="heart" size={24} color="#FF3B30" />  // regular (outline)
);

// Solid icons
const FASolid = () => (
  <FontAwesome5 name="heart" size={24} color="#FF3B30" solid />  // solid
);

// Brand icons (social media, etc.)
const FABrands = () => (
  <View style={{ flexDirection: 'row', gap: 16 }}>
    <FontAwesome5 name="facebook" size={24} color="#1877F2" brand />
    <FontAwesome5 name="twitter" size={24} color="#1DA1F2" brand />
    <FontAwesome5 name="instagram" size={24} color="#E1306C" brand />
    <FontAwesome5 name="youtube" size={24} color="#FF0000" brand />
    <FontAwesome5 name="github" size={24} color="#333" brand />
  </View>
);

// Light icons (Pro only - ต้องซื้อ FA Pro)
const FALight = () => (
  <FontAwesome5 name="heart" size={24} color="#FF3B30" light />
);
```

### Ionicons

iOS/Android cross-platform icons โดย Ionic

```jsx
import { Ionicons } from '@expo/vector-icons';
import { Platform } from 'react-native';

// Platform-adaptive icons
const AdaptiveIcons = () => {
  // Ionicons มี naming convention สำหรับ iOS (-outline) และ Android
  const homeIcon = Platform.OS === 'ios' ? 'home-outline' : 'home-sharp';
  const heartIcon = Platform.OS === 'ios' ? 'heart-outline' : 'heart';
  
  return (
    <View style={{ flexDirection: 'row', gap: 16 }}>
      <Ionicons name={homeIcon} size={24} color="#007AFF" />
      <Ionicons name={heartIcon} size={24} color="#FF3B30" />
    </View>
  );
};

// Icon families ใน Ionicons
const IoniconsFamilies = () => (
  <View>
    {/* Sharp (Android-style) */}
    <Ionicons name="home-sharp" size={24} color="#333" />
    
    {/* Outline (iOS-style) */}
    <Ionicons name="home-outline" size={24} color="#333" />
    
    {/* Default (filled) */}
    <Ionicons name="home" size={24} color="#333" />
  </View>
);
```

### Component ที่ใช้ icons สวยๆ

```jsx
import { MaterialIcons, Feather, Ionicons } from '@expo/vector-icons';

// Tab Bar ด้วย Vector Icons
const TabBar = ({ activeTab, onTabPress }) => {
  const tabs = [
    { id: 'home', label: 'หน้าหลัก', icon: 'home', library: 'Ionicons' },
    { id: 'explore', label: 'ค้นหา', icon: 'search', library: 'Feather' },
    { id: 'notifications', label: 'แจ้งเตือน', icon: 'bell', library: 'Feather' },
    { id: 'profile', label: 'โปรไฟล์', icon: 'user', library: 'Feather' },
  ];

  const renderIcon = (tab, isActive) => {
    const color = isActive ? '#007AFF' : '#8E8E93';
    const size = 24;

    switch (tab.library) {
      case 'Ionicons':
        return (
          <Ionicons
            name={isActive ? tab.icon : `${tab.icon}-outline`}
            size={size}
            color={color}
          />
        );
      case 'Feather':
        return <Feather name={tab.icon} size={size} color={color} />;
      default:
        return <MaterialIcons name={tab.icon} size={size} color={color} />;
    }
  };

  return (
    <View style={tabStyles.container}>
      {tabs.map((tab) => {
        const isActive = activeTab === tab.id;
        return (
          <TouchableOpacity
            key={tab.id}
            style={tabStyles.tab}
            onPress={() => onTabPress(tab.id)}
          >
            {renderIcon(tab, isActive)}
            <Text style={[tabStyles.label, isActive && tabStyles.activeLabel]}>
              {tab.label}
            </Text>
          </TouchableOpacity>
        );
      })}
    </View>
  );
};

const tabStyles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    backgroundColor: '#fff',
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    paddingBottom: 8,
  },
  tab: {
    flex: 1,
    alignItems: 'center',
    paddingTop: 8,
    gap: 4,
  },
  label: { fontSize: 11, color: '#8E8E93' },
  activeLabel: { color: '#007AFF', fontWeight: '600' },
});
```

---

## 3. Custom SVG Icons

### ติดตั้ง react-native-svg

```bash
npx expo install react-native-svg
```

### สร้าง Custom SVG Icon

```jsx
import React from 'react';
import Svg, { Path, Circle, G, Rect, Polyline, Line } from 'react-native-svg';

// Icon พื้นฐาน
const HeartIcon = ({ size = 24, color = '#FF3B30', filled = false }) => (
  <Svg width={size} height={size} viewBox="0 0 24 24" fill="none">
    <Path
      d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"
      stroke={color}
      strokeWidth={2}
      strokeLinecap="round"
      strokeLinejoin="round"
      fill={filled ? color : 'none'}
    />
  </Svg>
);

// Icon ซับซ้อนขึ้น
const StarIcon = ({ size = 24, color = '#FF9500', filled = false }) => (
  <Svg width={size} height={size} viewBox="0 0 24 24">
    <Path
      d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"
      stroke={color}
      strokeWidth={2}
      strokeLinecap="round"
      strokeLinejoin="round"
      fill={filled ? color : 'none'}
    />
  </Svg>
);

// Custom App Icons
const CustomIcons = {
  Home: ({ size = 24, color = '#333' }) => (
    <Svg width={size} height={size} viewBox="0 0 24 24" fill="none">
      <Path
        d="M3 9l9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"
        stroke={color}
        strokeWidth={2}
        strokeLinecap="round"
        strokeLinejoin="round"
      />
      <Polyline
        points="9 22 9 12 15 12 15 22"
        stroke={color}
        strokeWidth={2}
        strokeLinecap="round"
        strokeLinejoin="round"
      />
    </Svg>
  ),

  Search: ({ size = 24, color = '#333' }) => (
    <Svg width={size} height={size} viewBox="0 0 24 24" fill="none">
      <Circle cx={11} cy={11} r={8} stroke={color} strokeWidth={2} />
      <Line
        x1={21} y1={21} x2={16.65} y2={16.65}
        stroke={color} strokeWidth={2} strokeLinecap="round"
      />
    </Svg>
  ),

  Bell: ({ size = 24, color = '#333', hasNotification = false }) => (
    <Svg width={size} height={size} viewBox="0 0 24 24" fill="none">
      <Path
        d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9"
        stroke={color}
        strokeWidth={2}
        strokeLinecap="round"
        strokeLinejoin="round"
      />
      <Path
        d="M13.73 21a2 2 0 0 1-3.46 0"
        stroke={color}
        strokeWidth={2}
        strokeLinecap="round"
        strokeLinejoin="round"
      />
      {hasNotification && (
        <Circle cx={18} cy={6} r={4} fill="#FF3B30" />
      )}
    </Svg>
  ),

  // Animated loading spinner
  Loader: ({ size = 24, color = '#007AFF' }) => {
    const rotation = useRef(new Animated.Value(0)).current;

    useEffect(() => {
      Animated.loop(
        Animated.timing(rotation, {
          toValue: 1,
          duration: 1000,
          useNativeDriver: true,
        })
      ).start();
    }, []);

    const spin = rotation.interpolate({
      inputRange: [0, 1],
      outputRange: ['0deg', '360deg'],
    });

    return (
      <Animated.View style={{ transform: [{ rotate: spin }] }}>
        <Svg width={size} height={size} viewBox="0 0 24 24" fill="none">
          <Circle cx={12} cy={12} r={10} stroke={color} strokeWidth={2} opacity={0.25} />
          <Path
            d="M12 2a10 10 0 0 1 10 10"
            stroke={color}
            strokeWidth={2}
            strokeLinecap="round"
          />
        </Svg>
      </Animated.View>
    );
  },
};
```

---

## 4. react-native-svg ขั้นสูง

```jsx
import Svg, {
  Path,
  Circle,
  G,
  Defs,
  LinearGradient,
  Stop,
  RadialGradient,
  ClipPath,
  Rect,
  Text as SvgText,
} from 'react-native-svg';

// Icon พร้อม Gradient
const GradientIcon = ({ size = 48 }) => (
  <Svg width={size} height={size} viewBox="0 0 48 48">
    <Defs>
      <LinearGradient id="gradient" x1="0%" y1="0%" x2="100%" y2="100%">
        <Stop offset="0%" stopColor="#007AFF" />
        <Stop offset="100%" stopColor="#5856D6" />
      </LinearGradient>
    </Defs>
    <Circle cx={24} cy={24} r={20} fill="url(#gradient)" />
    <Path
      d="M24 14v10M24 34h.01"
      stroke="#fff"
      strokeWidth={3}
      strokeLinecap="round"
    />
  </Svg>
);

// Progress Circle
const ProgressCircle = ({ progress = 0.7, size = 80, color = '#007AFF' }) => {
  const radius = (size - 8) / 2;
  const circumference = 2 * Math.PI * radius;
  const strokeDashoffset = circumference * (1 - progress);
  const percentage = Math.round(progress * 100);

  return (
    <View style={{ width: size, height: size }}>
      <Svg width={size} height={size}>
        {/* Background circle */}
        <Circle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          fill="none"
          stroke="#f0f0f0"
          strokeWidth={6}
        />
        {/* Progress circle */}
        <Circle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          fill="none"
          stroke={color}
          strokeWidth={6}
          strokeDasharray={circumference}
          strokeDashoffset={strokeDashoffset}
          strokeLinecap="round"
          transform={`rotate(-90, ${size / 2}, ${size / 2})`}
        />
      </Svg>
      <View
        style={{
          position: 'absolute',
          width: size,
          height: size,
          justifyContent: 'center',
          alignItems: 'center',
        }}
      >
        <Text style={{ fontSize: size * 0.2, fontWeight: 'bold', color }}>
          {percentage}%
        </Text>
      </View>
    </View>
  );
};

// App Icon (ที่ใช้บน home screen)
const AppIcon = ({ size = 60 }) => (
  <View style={{
    width: size,
    height: size,
    borderRadius: size * 0.22,
    overflow: 'hidden',
    backgroundColor: '#007AFF',
  }}>
    <Svg width={size} height={size} viewBox="0 0 60 60">
      <Defs>
        <LinearGradient id="appGrad" x1="0%" y1="0%" x2="100%" y2="100%">
          <Stop offset="0%" stopColor="#007AFF" />
          <Stop offset="100%" stopColor="#0050CC" />
        </LinearGradient>
      </Defs>
      <Rect width={60} height={60} fill="url(#appGrad)" />
      <Path
        d="M30 10 L50 45 L10 45 Z"
        fill="rgba(255,255,255,0.9)"
      />
    </Svg>
  </View>
);
```

---

## 5. Icon sizes และ colors

### Design System สำหรับ Icons

```jsx
// Icon size system
export const ICON_SIZES = {
  xs: 14,    // ใน text หรือ chip
  sm: 16,    // เล็กน้อย
  md: 20,    // ปกติ
  lg: 24,    // tab bar, navigation
  xl: 32,    // card header
  '2xl': 40, // feature icons
  '3xl': 48, // empty state
  '4xl': 64, // hero icons
};

// Icon colors
export const ICON_COLORS = {
  primary: '#007AFF',
  secondary: '#5856D6',
  success: '#34C759',
  warning: '#FF9500',
  danger: '#FF3B30',
  info: '#5AC8FA',
  dark: '#1C1C1E',
  medium: '#636366',
  light: '#C7C7CC',
  white: '#FFFFFF',
};

// Icon component ที่ใช้ Design System
const Icon = ({ name, library = 'Feather', size = 'md', color = 'dark', style }) => {
  const resolvedSize = typeof size === 'number' ? size : ICON_SIZES[size];
  const resolvedColor = ICON_COLORS[color] || color;

  const IconComponent = {
    Feather,
    MaterialIcons,
    Ionicons,
    FontAwesome5,
    AntDesign,
  }[library];

  if (!IconComponent) return null;

  return (
    <IconComponent
      name={name}
      size={resolvedSize}
      color={resolvedColor}
      style={style}
    />
  );
};

// ตัวอย่างการใช้
const IconUsageExample = () => (
  <View style={{ gap: 16, padding: 20 }}>
    <Icon name="home" size="lg" color="primary" />
    <Icon name="heart" size="xl" color="danger" library="FontAwesome5" solid />
    <Icon name="bell" size="md" color="warning" library="Ionicons" />
    <Icon name="check-circle" size="2xl" color="success" />
  </View>
);
```

### Icon ใน Text (Inline Icons)

```jsx
// ใช้ icon ควบคู่กับ text
const InlineIconText = () => (
  <View>
    {/* ไม่แนะนำ: icon ใน Text จะมีปัญหา alignment */}
    {/* <Text>❤️ ชอบแล้ว</Text> */}

    {/* ✅ แนะนำ: ใช้ View + Row */}
    <View style={{ flexDirection: 'row', alignItems: 'center', gap: 6 }}>
      <MaterialIcons name="favorite" size={16} color="#FF3B30" />
      <Text>ชอบแล้ว</Text>
    </View>

    {/* Rating stars */}
    <View style={{ flexDirection: 'row', gap: 2 }}>
      {[1, 2, 3, 4, 5].map((star) => (
        <MaterialIcons
          key={star}
          name="star"
          size={16}
          color={star <= 4 ? '#FF9500' : '#e0e0e0'}
        />
      ))}
      <Text style={{ marginLeft: 4, color: '#666' }}>4.0 (128 รีวิว)</Text>
    </View>
  </View>
);
```

---

## 6. Workshop: Icon Showcase App

```jsx
import React, { useState, useMemo } from 'react';
import {
  View,
  Text,
  FlatList,
  TextInput,
  TouchableOpacity,
  ScrollView,
  StyleSheet,
  SafeAreaView,
  Clipboard,
  Alert,
} from 'react-native';
import * as AllIcons from '@expo/vector-icons';

// Icon libraries ที่จะแสดง
const ICON_LIBRARIES = [
  { name: 'Feather', component: AllIcons.Feather, icons: ['home', 'search', 'heart', 'star', 'bell', 'settings', 'user', 'camera', 'mail', 'phone', 'map-pin', 'clock', 'calendar', 'edit', 'trash', 'share', 'download', 'upload', 'plus', 'minus', 'check', 'x', 'arrow-left', 'arrow-right', 'chevron-down', 'chevron-up', 'menu', 'more-horizontal', 'more-vertical', 'bookmark', 'tag', 'coffee', 'music', 'film', 'book', 'globe', 'wifi', 'bluetooth', 'battery', 'zap'] },
  { name: 'MaterialIcons', component: AllIcons.MaterialIcons, icons: ['home', 'search', 'favorite', 'star', 'notifications', 'settings', 'person', 'camera', 'email', 'phone', 'location-on', 'access-time', 'event', 'edit', 'delete', 'share', 'file-download', 'file-upload', 'add', 'remove', 'check', 'close', 'arrow-back', 'arrow-forward', 'keyboard-arrow-down', 'keyboard-arrow-up', 'menu', 'more-horiz', 'more-vert', 'bookmark', 'label', 'local-cafe', 'music-note', 'movie', 'book', 'language', 'wifi', 'bluetooth', 'battery-full', 'flash-on'] },
  { name: 'Ionicons', component: AllIcons.Ionicons, icons: ['home', 'search', 'heart', 'star', 'notifications', 'settings', 'person', 'camera', 'mail', 'call', 'location', 'time', 'calendar', 'create', 'trash', 'share', 'download', 'cloud-upload', 'add', 'remove', 'checkmark', 'close', 'arrow-back', 'arrow-forward', 'chevron-down', 'chevron-up', 'menu', 'ellipsis-horizontal', 'ellipsis-vertical', 'bookmark', 'pricetag', 'cafe', 'musical-notes', 'film', 'book', 'globe', 'wifi', 'bluetooth', 'battery-full', 'flash'] },
];

// Icon Card Component
const IconCard = ({ name, IconComponent, onCopy }) => {
  return (
    <TouchableOpacity
      style={showcaseStyles.iconCard}
      onPress={() => onCopy(name)}
      activeOpacity={0.7}
    >
      <IconComponent name={name} size={28} color="#007AFF" />
      <Text style={showcaseStyles.iconName} numberOfLines={1}>
        {name}
      </Text>
    </TouchableOpacity>
  );
};

// Size Demo Component
const SizeDemo = ({ name, IconComponent }) => {
  const sizes = [12, 16, 20, 24, 32, 40, 48];
  
  return (
    <View style={showcaseStyles.sizeDemo}>
      <Text style={showcaseStyles.sizeDemoTitle}>ขนาด: {name}</Text>
      <View style={{ flexDirection: 'row', alignItems: 'flex-end', gap: 12 }}>
        {sizes.map((size) => (
          <View key={size} style={{ alignItems: 'center', gap: 4 }}>
            <IconComponent name={name} size={size} color="#007AFF" />
            <Text style={{ fontSize: 10, color: '#999' }}>{size}</Text>
          </View>
        ))}
      </View>
    </View>
  );
};

// Color Demo Component
const ColorDemo = ({ name, IconComponent }) => {
  const colors = [
    { color: '#007AFF', name: 'Blue' },
    { color: '#FF3B30', name: 'Red' },
    { color: '#34C759', name: 'Green' },
    { color: '#FF9500', name: 'Orange' },
    { color: '#5856D6', name: 'Purple' },
    { color: '#FF2D55', name: 'Pink' },
  ];

  return (
    <View style={showcaseStyles.colorDemo}>
      <Text style={showcaseStyles.sizeDemoTitle}>สี: {name}</Text>
      <View style={{ flexDirection: 'row', gap: 16 }}>
        {colors.map(({ color, name: colorName }) => (
          <View key={color} style={{ alignItems: 'center', gap: 4 }}>
            <IconComponent name={name} size={28} color={color} />
            <Text style={{ fontSize: 10, color: '#999' }}>{colorName}</Text>
          </View>
        ))}
      </View>
    </View>
  );
};

// Main Showcase App
const IconShowcaseApp = () => {
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedLibrary, setSelectedLibrary] = useState(0);
  const [selectedIcon, setSelectedIcon] = useState(null);

  const currentLib = ICON_LIBRARIES[selectedLibrary];

  const filteredIcons = useMemo(() => {
    if (!searchQuery.trim()) return currentLib.icons;
    return currentLib.icons.filter((icon) =>
      icon.toLowerCase().includes(searchQuery.toLowerCase())
    );
  }, [searchQuery, currentLib]);

  const handleCopyIcon = (iconName) => {
    const code = `<${currentLib.name} name="${iconName}" size={24} color="#007AFF" />`;
    Clipboard.setString(code);
    Alert.alert('✓ คัดลอกแล้ว', code);
    setSelectedIcon(iconName);
  };

  return (
    <SafeAreaView style={{ flex: 1, backgroundColor: '#f5f5f5' }}>
      {/* Header */}
      <View style={showcaseStyles.header}>
        <Text style={showcaseStyles.headerTitle}>🎨 Icon Showcase</Text>
        <Text style={showcaseStyles.headerSubtitle}>
          {filteredIcons.length} icons
        </Text>
      </View>

      {/* Search */}
      <View style={showcaseStyles.searchContainer}>
        <AllIcons.Feather name="search" size={18} color="#999" />
        <TextInput
          style={showcaseStyles.searchInput}
          placeholder="ค้นหา icon..."
          value={searchQuery}
          onChangeText={setSearchQuery}
          clearButtonMode="while-editing"
        />
      </View>

      {/* Library Selector */}
      <ScrollView
        horizontal
        showsHorizontalScrollIndicator={false}
        contentContainerStyle={showcaseStyles.libraryTabs}
      >
        {ICON_LIBRARIES.map((lib, index) => (
          <TouchableOpacity
            key={lib.name}
            style={[
              showcaseStyles.libraryTab,
              selectedLibrary === index && showcaseStyles.activeTab,
            ]}
            onPress={() => {
              setSelectedLibrary(index);
              setSearchQuery('');
              setSelectedIcon(null);
            }}
          >
            <Text
              style={[
                showcaseStyles.tabText,
                selectedLibrary === index && showcaseStyles.activeTabText,
              ]}
            >
              {lib.name}
            </Text>
            <Text style={showcaseStyles.tabCount}>
              {lib.icons.length}
            </Text>
          </TouchableOpacity>
        ))}
      </ScrollView>

      {/* Selected Icon Detail */}
      {selectedIcon && (
        <View style={showcaseStyles.detailPanel}>
          <SizeDemo name={selectedIcon} IconComponent={currentLib.component} />
          <ColorDemo name={selectedIcon} IconComponent={currentLib.component} />
          <TouchableOpacity
            style={showcaseStyles.copyButton}
            onPress={() => handleCopyIcon(selectedIcon)}
          >
            <AllIcons.Feather name="copy" size={16} color="#fff" />
            <Text style={showcaseStyles.copyButtonText}>คัดลอก Code</Text>
          </TouchableOpacity>
        </View>
      )}

      {/* Icon Grid */}
      <FlatList
        data={filteredIcons}
        numColumns={4}
        keyExtractor={(item) => item}
        contentContainerStyle={showcaseStyles.grid}
        renderItem={({ item }) => (
          <IconCard
            name={item}
            IconComponent={currentLib.component}
            onCopy={handleCopyIcon}
          />
        )}
        ListEmptyComponent={
          <View style={{ padding: 40, alignItems: 'center' }}>
            <AllIcons.Feather name="search" size={48} color="#ccc" />
            <Text style={{ color: '#999', marginTop: 12 }}>
              ไม่พบ icon "{searchQuery}"
            </Text>
          </View>
        }
      />
    </SafeAreaView>
  );
};

const showcaseStyles = StyleSheet.create({
  header: {
    padding: 16,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  headerTitle: { fontSize: 24, fontWeight: 'bold' },
  headerSubtitle: { color: '#666', fontSize: 14 },
  searchContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    margin: 12,
    paddingHorizontal: 16,
    backgroundColor: '#fff',
    borderRadius: 12,
    borderWidth: 1,
    borderColor: '#e0e0e0',
    gap: 8,
  },
  searchInput: { flex: 1, paddingVertical: 12, fontSize: 16 },
  libraryTabs: {
    paddingHorizontal: 12,
    paddingVertical: 8,
    gap: 8,
  },
  libraryTab: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 8,
    paddingHorizontal: 14,
    borderRadius: 20,
    backgroundColor: '#fff',
    borderWidth: 1,
    borderColor: '#e0e0e0',
    gap: 6,
  },
  activeTab: {
    backgroundColor: '#007AFF',
    borderColor: '#007AFF',
  },
  tabText: { fontSize: 13, color: '#333' },
  activeTabText: { color: '#fff', fontWeight: '600' },
  tabCount: { fontSize: 11, color: '#999' },
  detailPanel: {
    margin: 12,
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
    gap: 16,
  },
  sizeDemo: {},
  colorDemo: {},
  sizeDemoTitle: {
    fontSize: 14,
    fontWeight: '600',
    color: '#666',
    marginBottom: 8,
  },
  copyButton: {
    backgroundColor: '#007AFF',
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    padding: 12,
    borderRadius: 10,
    gap: 8,
  },
  copyButtonText: { color: '#fff', fontWeight: '600' },
  grid: {
    padding: 8,
    paddingBottom: 32,
  },
  iconCard: {
    flex: 1,
    margin: 4,
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 12,
    alignItems: 'center',
    gap: 8,
    aspectRatio: 1,
    justifyContent: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.05,
    shadowRadius: 2,
    elevation: 1,
  },
  iconName: {
    fontSize: 9,
    color: '#666',
    textAlign: 'center',
  },
});

export default IconShowcaseApp;
```

---

## Tips และ Best Practices

### 1. Performance
```jsx
// ✅ import เฉพาะ library ที่ต้องการ
import { Feather } from '@expo/vector-icons';  // ✅

// ❌ อย่า import ทั้งหมด
import * as AllIcons from '@expo/vector-icons';  // ❌
```

### 2. Consistent Icon Style
```jsx
// ✅ ใช้ library เดียวกันตลอด app หรือกำหนด mapping ชัดเจน
const AppIcons = {
  home: { library: 'Feather', name: 'home' },
  search: { library: 'Feather', name: 'search' },
  // ...
};
```

### 3. Accessibility
```jsx
<TouchableOpacity accessibilityLabel="ค้นหา">
  <Feather name="search" size={24} color="#333" />
</TouchableOpacity>
```

### 4. Preload Icons
```jsx
// ใน Expo ต้องโหลด fonts ก่อน
import { useFonts } from 'expo-font';

const [fontsLoaded] = useFonts({
  ...Feather.font,
  ...MaterialIcons.font,
});

if (!fontsLoaded) return null;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การใช้ `@expo/vector-icons` กับ libraries ต่างๆ
- ความแตกต่างระหว่าง MaterialIcons, FontAwesome, Ionicons
- การสร้าง Custom SVG icons ด้วย `react-native-svg`
- Design system สำหรับ icon sizes และ colors
- Workshop: Icon showcase app

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Fonts และ Typography
