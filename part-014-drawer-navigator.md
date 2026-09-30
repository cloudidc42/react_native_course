# Part 014: Drawer Navigator

## สารบัญ
1. createDrawerNavigator
2. Custom Drawer Content
3. Drawer Items
4. Opening/Closing Drawer Programmatically
5. Workshop: App with Sidebar Menu

---

## 1. createDrawerNavigator

Drawer Navigator แสดง sidebar menu ที่ซ่อนอยู่ทางด้านซ้ายหรือขวาของหน้าจอ เหมาะสำหรับแอปที่มีหลาย section แต่ไม่ต้องการแสดง tab bar ตลอดเวลา

### การติดตั้ง

```bash
npm install @react-navigation/drawer
npm install react-native-gesture-handler react-native-reanimated
```

### ตั้งค่า react-native-reanimated (babel.config.js)

```javascript
// babel.config.js
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: ['react-native-reanimated/plugin'],
};
```

### ตั้งค่า index.js

```javascript
// index.js (ต้อง import ก่อนอื่นทั้งหมด)
import 'react-native-gesture-handler';
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

AppRegistry.registerComponent(appName, () => App);
```

### การใช้งานพื้นฐาน

```javascript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

function HomeScreen() {
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <Text>หน้าหลัก</Text>
    </View>
  );
}

function SettingsScreen() {
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <Text>ตั้งค่า</Text>
    </View>
  );
}

export default function App() {
  return (
    <NavigationContainer>
      <Drawer.Navigator initialRouteName="Home">
        <Drawer.Screen name="Home" component={HomeScreen} />
        <Drawer.Screen name="Settings" component={SettingsScreen} />
        <Drawer.Screen name="About" component={AboutScreen} />
      </Drawer.Navigator>
    </NavigationContainer>
  );
}
```

### Drawer Navigator Options

```javascript
<Drawer.Navigator
  initialRouteName="Home"
  screenOptions={{
    // ===== Drawer Style =====
    drawerStyle: {
      backgroundColor: '#FFF',
      width: 280,
    },
    drawerType: 'front',  // 'front' | 'back' | 'slide' | 'permanent'
    drawerPosition: 'left',  // 'left' | 'right'
    
    // ===== Drawer Items =====
    drawerActiveTintColor: '#6200EE',    // สีข้อความเมื่อ active
    drawerInactiveTintColor: '#555',     // สีข้อความเมื่อ inactive
    drawerActiveBackgroundColor: '#EDE7F6',  // สีพื้นหลังเมื่อ active
    drawerItemStyle: {
      borderRadius: 8,
      marginVertical: 2,
    },
    drawerLabelStyle: {
      fontSize: 15,
      fontWeight: '500',
    },
    
    // ===== Header =====
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#FFF',
    
    // ===== Overlay =====
    overlayColor: 'rgba(0,0,0,0.5)',
    
    // ===== Swipe =====
    swipeEnabled: true,
    swipeEdgeWidth: 50,
    swipeMinDistance: 5,
    
    // ===== Gesture =====
    gestureHandlerProps: {
      minOffsetX: 10,
    },
  }}
>
```

---

## 2. Custom Drawer Content

### Basic Custom Drawer

```javascript
import { DrawerContentScrollView, DrawerItemList } from '@react-navigation/drawer';

function CustomDrawerContent(props) {
  return (
    <DrawerContentScrollView {...props}>
      {/* Header */}
      <View style={styles.header}>
        <Image 
          source={{ uri: 'https://picsum.photos/id/1/80/80' }}
          style={styles.avatar}
        />
        <Text style={styles.name}>สมชาย ใจดี</Text>
        <Text style={styles.email}>somchai@example.com</Text>
      </View>
      
      {/* Default drawer items */}
      <DrawerItemList {...props} />
      
      {/* Footer */}
      <View style={styles.footer}>
        <TouchableOpacity style={styles.logoutBtn}>
          <Ionicons name="log-out-outline" size={22} color="#E53935" />
          <Text style={styles.logoutText}>ออกจากระบบ</Text>
        </TouchableOpacity>
      </View>
    </DrawerContentScrollView>
  );
}

const styles = StyleSheet.create({
  header: {
    padding: 20,
    borderBottomWidth: 1,
    borderBottomColor: '#EEE',
    marginBottom: 8,
  },
  avatar: {
    width: 70,
    height: 70,
    borderRadius: 35,
    marginBottom: 12,
  },
  name: { fontSize: 18, fontWeight: 'bold', color: '#212121' },
  email: { fontSize: 13, color: '#666', marginTop: 2 },
  footer: {
    padding: 16,
    marginTop: 'auto',
    borderTopWidth: 1,
    borderTopColor: '#EEE',
  },
  logoutBtn: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 12,
  },
  logoutText: { fontSize: 15, color: '#E53935', marginLeft: 12 },
});

// ใช้งาน
<Drawer.Navigator drawerContent={(props) => <CustomDrawerContent {...props} />}>
```

### Advanced Custom Drawer

```javascript
import React, { useState } from 'react';
import {
  View, Text, StyleSheet, Image, TouchableOpacity,
  Switch, ScrollView, Animated
} from 'react-native';
import { DrawerContentScrollView } from '@react-navigation/drawer';
import { Ionicons } from '@expo/vector-icons';
import { useNavigation } from '@react-navigation/native';

const menuItems = [
  {
    section: 'เมนูหลัก',
    items: [
      { name: 'Home', label: 'หน้าหลัก', icon: 'home-outline', activeIcon: 'home' },
      { name: 'News', label: 'ข่าวสาร', icon: 'newspaper-outline', activeIcon: 'newspaper' },
      { name: 'Events', label: 'กิจกรรม', icon: 'calendar-outline', activeIcon: 'calendar' },
    ],
  },
  {
    section: 'บัญชีของฉัน',
    items: [
      { name: 'Profile', label: 'โปรไฟล์', icon: 'person-outline', activeIcon: 'person' },
      { name: 'Favorites', label: 'รายการโปรด', icon: 'heart-outline', activeIcon: 'heart', badge: 3 },
      { name: 'Orders', label: 'คำสั่งซื้อ', icon: 'receipt-outline', activeIcon: 'receipt' },
    ],
  },
  {
    section: 'อื่นๆ',
    items: [
      { name: 'Settings', label: 'ตั้งค่า', icon: 'settings-outline', activeIcon: 'settings' },
      { name: 'Help', label: 'ช่วยเหลือ', icon: 'help-circle-outline', activeIcon: 'help-circle' },
      { name: 'About', label: 'เกี่ยวกับ', icon: 'information-circle-outline', activeIcon: 'information-circle' },
    ],
  },
];

export default function AdvancedDrawerContent({ state, navigation }) {
  const [darkMode, setDarkMode] = useState(false);
  const currentRoute = state.routes[state.index]?.name;

  const handleLogout = () => {
    navigation.closeDrawer();
    // logout logic
  };

  return (
    <View style={styles.container}>
      {/* User Profile Header */}
      <View style={styles.profileSection}>
        <TouchableOpacity onPress={() => navigation.navigate('Profile')}>
          <Image
            source={{ uri: 'https://picsum.photos/id/1/80/80' }}
            style={styles.avatar}
          />
          <View style={styles.onlineBadge} />
        </TouchableOpacity>
        <View style={styles.profileInfo}>
          <Text style={styles.profileName}>สมชาย ใจดี</Text>
          <Text style={styles.profileEmail}>somchai@example.com</Text>
          <View style={styles.profileStats}>
            <View style={styles.stat}>
              <Text style={styles.statNumber}>142</Text>
              <Text style={styles.statLabel}>โพสต์</Text>
            </View>
            <View style={styles.stat}>
              <Text style={styles.statNumber}>2.3K</Text>
              <Text style={styles.statLabel}>ผู้ติดตาม</Text>
            </View>
          </View>
        </View>
      </View>

      {/* Menu Items */}
      <ScrollView style={styles.menuContainer} showsVerticalScrollIndicator={false}>
        {menuItems.map((section) => (
          <View key={section.section}>
            <Text style={styles.sectionTitle}>{section.section}</Text>
            {section.items.map((item) => {
              const isActive = currentRoute === item.name;
              return (
                <TouchableOpacity
                  key={item.name}
                  style={[styles.menuItem, isActive && styles.activeMenuItem]}
                  onPress={() => {
                    navigation.navigate(item.name);
                    navigation.closeDrawer();
                  }}
                >
                  <View style={styles.menuItemLeft}>
                    <View style={[styles.iconContainer, isActive && styles.activeIconContainer]}>
                      <Ionicons
                        name={isActive ? item.activeIcon : item.icon}
                        size={20}
                        color={isActive ? '#6200EE' : '#666'}
                      />
                    </View>
                    <Text style={[styles.menuLabel, isActive && styles.activeMenuLabel]}>
                      {item.label}
                    </Text>
                  </View>
                  {item.badge && (
                    <View style={styles.badge}>
                      <Text style={styles.badgeText}>{item.badge}</Text>
                    </View>
                  )}
                  {isActive && (
                    <View style={styles.activeIndicator} />
                  )}
                </TouchableOpacity>
              );
            })}
          </View>
        ))}
      </ScrollView>

      {/* Footer */}
      <View style={styles.footer}>
        {/* Dark Mode Toggle */}
        <View style={styles.darkModeRow}>
          <View style={styles.darkModeLeft}>
            <Ionicons name="moon-outline" size={20} color="#666" />
            <Text style={styles.darkModeText}>Dark Mode</Text>
          </View>
          <Switch
            value={darkMode}
            onValueChange={setDarkMode}
            trackColor={{ false: '#DDD', true: '#6200EE' }}
            thumbColor={darkMode ? '#FFF' : '#FFF'}
          />
        </View>

        {/* Logout */}
        <TouchableOpacity style={styles.logoutBtn} onPress={handleLogout}>
          <Ionicons name="log-out-outline" size={20} color="#E53935" />
          <Text style={styles.logoutText}>ออกจากระบบ</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#FFF' },
  profileSection: {
    flexDirection: 'row',
    padding: 20,
    paddingTop: 50,
    backgroundColor: '#6200EE',
    alignItems: 'center',
  },
  avatar: {
    width: 60,
    height: 60,
    borderRadius: 30,
    borderWidth: 2,
    borderColor: '#FFF',
  },
  onlineBadge: {
    position: 'absolute',
    bottom: 0,
    right: 0,
    width: 14,
    height: 14,
    borderRadius: 7,
    backgroundColor: '#4CAF50',
    borderWidth: 2,
    borderColor: '#FFF',
  },
  profileInfo: { marginLeft: 12, flex: 1 },
  profileName: { fontSize: 16, fontWeight: 'bold', color: '#FFF' },
  profileEmail: { fontSize: 12, color: '#E0D7FF', marginTop: 2 },
  profileStats: { flexDirection: 'row', marginTop: 8 },
  stat: { marginRight: 16 },
  statNumber: { fontSize: 14, fontWeight: 'bold', color: '#FFF' },
  statLabel: { fontSize: 11, color: '#E0D7FF' },
  menuContainer: { flex: 1, paddingTop: 8 },
  sectionTitle: {
    fontSize: 12,
    fontWeight: '700',
    color: '#999',
    paddingHorizontal: 16,
    paddingTop: 16,
    paddingBottom: 4,
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
  menuItem: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingHorizontal: 16,
    paddingVertical: 10,
    marginHorizontal: 8,
    borderRadius: 10,
    overflow: 'hidden',
  },
  activeMenuItem: { backgroundColor: '#EDE7F6' },
  menuItemLeft: { flexDirection: 'row', alignItems: 'center' },
  iconContainer: {
    width: 36,
    height: 36,
    borderRadius: 10,
    backgroundColor: '#F5F5F5',
    alignItems: 'center',
    justifyContent: 'center',
    marginRight: 12,
  },
  activeIconContainer: { backgroundColor: '#D1C4E9' },
  menuLabel: { fontSize: 14, color: '#555', fontWeight: '500' },
  activeMenuLabel: { color: '#6200EE', fontWeight: '700' },
  badge: {
    backgroundColor: '#6200EE',
    borderRadius: 12,
    paddingHorizontal: 7,
    paddingVertical: 2,
    minWidth: 24,
    alignItems: 'center',
  },
  badgeText: { color: '#FFF', fontSize: 11, fontWeight: 'bold' },
  activeIndicator: {
    position: 'absolute',
    right: 0,
    top: '20%',
    bottom: '20%',
    width: 4,
    backgroundColor: '#6200EE',
    borderRadius: 2,
  },
  footer: {
    borderTopWidth: 1,
    borderTopColor: '#F0F0F0',
    padding: 16,
  },
  darkModeRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 8,
  },
  darkModeLeft: { flexDirection: 'row', alignItems: 'center' },
  darkModeText: { fontSize: 14, color: '#555', marginLeft: 12 },
  logoutBtn: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 12,
    marginTop: 4,
  },
  logoutText: { fontSize: 14, color: '#E53935', marginLeft: 12, fontWeight: '500' },
});
```

---

## 3. Drawer Items

### DrawerItem Component

```javascript
import { DrawerItem } from '@react-navigation/drawer';

function CustomDrawerContent(props) {
  return (
    <DrawerContentScrollView {...props}>
      <DrawerItemList {...props} />
      
      {/* Custom items */}
      <DrawerItem
        label="เว็บไซต์เรา"
        icon={({ color, size }) => (
          <Ionicons name="globe-outline" size={size} color={color} />
        )}
        onPress={() => Linking.openURL('https://example.com')}
        activeTintColor="#6200EE"
        inactiveTintColor="#555"
      />
      
      <DrawerItem
        label="แชร์แอป"
        icon={({ color, size }) => (
          <Ionicons name="share-social-outline" size={size} color={color} />
        )}
        onPress={() => Share.share({ message: 'ลองใช้แอปนี้สิ!' })}
      />
      
      <DrawerItem
        label="ออกจากระบบ"
        icon={({ color, size }) => (
          <Ionicons name="log-out-outline" size={size} color="#E53935" />
        )}
        labelStyle={{ color: '#E53935' }}
        onPress={() => console.log('Logout')}
      />
    </DrawerContentScrollView>
  );
}
```

### Screen-level Drawer Options

```javascript
<Drawer.Screen
  name="Home"
  component={HomeScreen}
  options={{
    title: 'หน้าหลัก',
    
    // Icon ใน drawer
    drawerIcon: ({ focused, color, size }) => (
      <Ionicons 
        name={focused ? 'home' : 'home-outline'} 
        size={size} 
        color={color} 
      />
    ),
    
    // Label ใน drawer
    drawerLabel: 'หน้าหลัก',
    
    // ซ่อนจาก drawer (แต่ยัง navigate ได้)
    drawerItemStyle: { display: 'none' },
    
    // Active/inactive colors
    drawerActiveTintColor: '#6200EE',
    drawerInactiveTintColor: '#666',
  }}
/>
```

---

## 4. Opening/Closing Drawer Programmatically

### ใช้ navigation prop

```javascript
function HomeScreen({ navigation }) {
  return (
    <View>
      {/* เปิด drawer */}
      <Button title="เปิดเมนู" onPress={() => navigation.openDrawer()} />
      
      {/* ปิด drawer */}
      <Button title="ปิดเมนู" onPress={() => navigation.closeDrawer()} />
      
      {/* Toggle drawer */}
      <Button title="Toggle เมนู" onPress={() => navigation.toggleDrawer()} />
    </View>
  );
}
```

### ใช้ useNavigation Hook

```javascript
import { useNavigation, DrawerActions } from '@react-navigation/native';

// Component ที่ไม่ใช่ Screen
function HamburgerButton() {
  const navigation = useNavigation();
  
  return (
    <TouchableOpacity onPress={() => navigation.dispatch(DrawerActions.openDrawer())}>
      <Ionicons name="menu" size={28} color="#FFF" />
    </TouchableOpacity>
  );
}

// ใช้ DrawerActions
navigation.dispatch(DrawerActions.openDrawer());
navigation.dispatch(DrawerActions.closeDrawer());
navigation.dispatch(DrawerActions.toggleDrawer());
```

### ปุ่ม Hamburger ใน Header

```javascript
<Drawer.Navigator
  screenOptions={({ navigation }) => ({
    headerLeft: () => (
      <TouchableOpacity
        style={{ marginLeft: 15 }}
        onPress={() => navigation.toggleDrawer()}
      >
        <Ionicons name="menu" size={28} color="#FFF" />
      </TouchableOpacity>
    ),
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#FFF',
  })}
>
```

### Drawer State

```javascript
import { useDrawerStatus } from '@react-navigation/drawer';

function HomeScreen() {
  const drawerStatus = useDrawerStatus();
  // 'open' | 'closed'
  
  return (
    <View>
      <Text>Drawer is: {drawerStatus}</Text>
    </View>
  );
}
```

---

## Workshop: App with Sidebar Menu

### โครงสร้างแอป News App

```
NewsApp/
├── App.js
├── navigation/
│   └── AppNavigator.js
├── screens/
│   ├── HomeScreen.js
│   ├── CategoryScreen.js
│   ├── BookmarksScreen.js
│   ├── ProfileScreen.js
│   ├── SettingsScreen.js
│   └── ArticleScreen.js
├── components/
│   ├── DrawerContent.js
│   ├── ArticleCard.js
│   └── CategoryBadge.js
└── data/
    └── news.js
```

### Data

```javascript
// data/news.js
export const categories = [
  { id: 'all', name: 'ทั้งหมด', color: '#6200EE' },
  { id: 'tech', name: 'เทคโนโลยี', color: '#2196F3' },
  { id: 'business', name: 'ธุรกิจ', color: '#4CAF50' },
  { id: 'sports', name: 'กีฬา', color: '#FF5722' },
  { id: 'health', name: 'สุขภาพ', color: '#E91E63' },
];

export const articles = [
  {
    id: '1',
    category: 'tech',
    title: 'Apple เปิดตัว iPhone 16 Pro สุดล้ำด้วย AI',
    summary: 'Apple เปิดตัว iPhone รุ่นล่าสุดพร้อม chip A18 Pro และ AI features ที่ทรงพลัง',
    image: 'https://picsum.photos/id/1/400/200',
    source: 'TechThailand',
    time: '2 ชม.',
    readTime: '5 นาที',
    views: 12453,
    bookmarked: false,
  },
  {
    id: '2',
    category: 'business',
    title: 'ตลาดหุ้นไทยปรับตัวขึ้น 1.5% หลังข่าวดี',
    summary: 'ดัชนี SET ปรับตัวขึ้นอย่างแข็งแกร่งหลังจากมีข่าวดีด้านเศรษฐกิจ',
    image: 'https://picsum.photos/id/2/400/200',
    source: 'BusinessDaily',
    time: '4 ชม.',
    readTime: '3 นาที',
    views: 8234,
    bookmarked: true,
  },
  {
    id: '3',
    category: 'sports',
    title: 'ทีมฟุตบอลไทยผ่านรอบคัดเลือก World Cup 2026',
    summary: 'ทีมชาติไทยเอาชนะเวียดนาม 2-0 ผ่านเข้าสู่รอบต่อไปของ World Cup',
    image: 'https://picsum.photos/id/3/400/200',
    source: 'SportsTH',
    time: '6 ชม.',
    readTime: '4 นาที',
    views: 45678,
    bookmarked: false,
  },
];
```

### DrawerContent Component

```javascript
// components/DrawerContent.js
import React from 'react';
import {
  View, Text, StyleSheet, Image,
  TouchableOpacity, ScrollView
} from 'react-native';
import { DrawerItem } from '@react-navigation/drawer';
import { Ionicons } from '@expo/vector-icons';

const mainMenu = [
  { name: 'Home', label: 'หน้าหลัก', icon: 'home', screen: 'Home' },
  { name: 'Bookmarks', label: 'บุ๊กมาร์ก', icon: 'bookmark', screen: 'Bookmarks', badge: 2 },
  { name: 'Profile', label: 'โปรไฟล์', icon: 'person', screen: 'Profile' },
];

const categories = [
  { id: 'tech', name: 'เทคโนโลยี', color: '#2196F3' },
  { id: 'business', name: 'ธุรกิจ', color: '#4CAF50' },
  { id: 'sports', name: 'กีฬา', color: '#FF5722' },
  { id: 'health', name: 'สุขภาพ', color: '#E91E63' },
];

export default function DrawerContent({ state, navigation }) {
  const currentRoute = state.routes[state.index]?.name;

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <View style={styles.logoContainer}>
          <Text style={styles.logo}>📰</Text>
          <Text style={styles.appName}>NewsApp</Text>
        </View>
        <TouchableOpacity 
          style={styles.closeBtn}
          onPress={() => navigation.closeDrawer()}
        >
          <Ionicons name="close" size={24} color="#FFF" />
        </TouchableOpacity>
      </View>

      <ScrollView showsVerticalScrollIndicator={false}>
        {/* Main Menu */}
        <Text style={styles.sectionLabel}>เมนูหลัก</Text>
        {mainMenu.map(item => {
          const isActive = currentRoute === item.screen;
          return (
            <TouchableOpacity
              key={item.name}
              style={[styles.menuItem, isActive && styles.activeItem]}
              onPress={() => {
                navigation.navigate(item.screen);
                navigation.closeDrawer();
              }}
            >
              <Ionicons
                name={isActive ? item.icon : `${item.icon}-outline`}
                size={22}
                color={isActive ? '#6200EE' : '#555'}
              />
              <Text style={[styles.menuText, isActive && styles.activeText]}>
                {item.label}
              </Text>
              {item.badge && (
                <View style={styles.badge}>
                  <Text style={styles.badgeText}>{item.badge}</Text>
                </View>
              )}
            </TouchableOpacity>
          );
        })}

        {/* Categories */}
        <Text style={styles.sectionLabel}>หมวดหมู่</Text>
        {categories.map(cat => (
          <TouchableOpacity
            key={cat.id}
            style={styles.categoryItem}
            onPress={() => {
              navigation.navigate('Category', { 
                categoryId: cat.id, 
                categoryName: cat.name 
              });
              navigation.closeDrawer();
            }}
          >
            <View style={[styles.categoryDot, { backgroundColor: cat.color }]} />
            <Text style={styles.categoryText}>{cat.name}</Text>
          </TouchableOpacity>
        ))}
      </ScrollView>

      {/* Footer */}
      <View style={styles.footer}>
        <TouchableOpacity
          style={styles.settingsBtn}
          onPress={() => {
            navigation.navigate('Settings');
            navigation.closeDrawer();
          }}
        >
          <Ionicons name="settings-outline" size={22} color="#666" />
          <Text style={styles.settingsText}>ตั้งค่า</Text>
        </TouchableOpacity>

        <TouchableOpacity style={styles.logoutBtn}>
          <Ionicons name="log-out-outline" size={22} color="#E53935" />
          <Text style={styles.logoutText}>ออกจากระบบ</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#FFF' },
  header: {
    backgroundColor: '#6200EE',
    padding: 20,
    paddingTop: 50,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  logoContainer: { flexDirection: 'row', alignItems: 'center' },
  logo: { fontSize: 30 },
  appName: {
    fontSize: 22,
    fontWeight: 'bold',
    color: '#FFF',
    marginLeft: 10,
  },
  closeBtn: { padding: 4 },
  sectionLabel: {
    fontSize: 12,
    fontWeight: '700',
    color: '#AAAAAA',
    paddingHorizontal: 16,
    paddingTop: 20,
    paddingBottom: 6,
    textTransform: 'uppercase',
    letterSpacing: 0.8,
  },
  menuItem: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingVertical: 12,
    marginHorizontal: 8,
    borderRadius: 10,
  },
  activeItem: { backgroundColor: '#EDE7F6' },
  menuText: { fontSize: 15, color: '#555', marginLeft: 14, flex: 1 },
  activeText: { color: '#6200EE', fontWeight: '600' },
  badge: {
    backgroundColor: '#6200EE',
    borderRadius: 10,
    paddingHorizontal: 7,
    paddingVertical: 2,
  },
  badgeText: { color: '#FFF', fontSize: 11, fontWeight: 'bold' },
  categoryItem: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingVertical: 10,
    marginHorizontal: 8,
  },
  categoryDot: { width: 10, height: 10, borderRadius: 5, marginRight: 14 },
  categoryText: { fontSize: 14, color: '#555' },
  footer: {
    borderTopWidth: 1,
    borderTopColor: '#F0F0F0',
    padding: 16,
  },
  settingsBtn: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 10,
  },
  settingsText: { fontSize: 14, color: '#666', marginLeft: 12 },
  logoutBtn: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 10,
    marginTop: 4,
  },
  logoutText: { fontSize: 14, color: '#E53935', marginLeft: 12 },
});
```

### AppNavigator

```javascript
// navigation/AppNavigator.js
import React from 'react';
import { createDrawerNavigator } from '@react-navigation/drawer';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import DrawerContent from '../components/DrawerContent';
import HomeScreen from '../screens/HomeScreen';
import CategoryScreen from '../screens/CategoryScreen';
import BookmarksScreen from '../screens/BookmarksScreen';
import ProfileScreen from '../screens/ProfileScreen';
import SettingsScreen from '../screens/SettingsScreen';
import ArticleScreen from '../screens/ArticleScreen';

const Drawer = createDrawerNavigator();
const Stack = createNativeStackNavigator();

// Stack สำหรับ Home (เพื่อ navigate ไปหน้า Article)
function HomeStack() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen 
        name="Article" 
        component={ArticleScreen}
        options={{ headerShown: true, title: 'บทความ' }}
      />
    </Stack.Navigator>
  );
}

export default function AppNavigator() {
  return (
    <Drawer.Navigator
      drawerContent={(props) => <DrawerContent {...props} />}
      screenOptions={{
        drawerStyle: { width: 280 },
        headerStyle: { backgroundColor: '#6200EE' },
        headerTintColor: '#FFF',
        headerTitleStyle: { fontWeight: 'bold' },
        drawerActiveTintColor: '#6200EE',
        drawerInactiveTintColor: '#666',
        overlayColor: 'rgba(0,0,0,0.5)',
      }}
    >
      <Drawer.Screen
        name="HomeStack"
        component={HomeStack}
        options={{
          title: 'หน้าหลัก',
          headerTitle: '📰 NewsApp',
        }}
      />
      <Drawer.Screen
        name="Category"
        component={CategoryScreen}
        options={({ route }) => ({
          title: route.params?.categoryName || 'หมวดหมู่',
        })}
      />
      <Drawer.Screen
        name="Bookmarks"
        component={BookmarksScreen}
        options={{ title: 'บุ๊กมาร์ก' }}
      />
      <Drawer.Screen
        name="Profile"
        component={ProfileScreen}
        options={{ title: 'โปรไฟล์ของฉัน' }}
      />
      <Drawer.Screen
        name="Settings"
        component={SettingsScreen}
        options={{ title: 'ตั้งค่า' }}
      />
    </Drawer.Navigator>
  );
}
```

### HomeScreen

```javascript
// screens/HomeScreen.js
import React, { useState } from 'react';
import {
  View, Text, StyleSheet, FlatList,
  Image, TouchableOpacity, ScrollView
} from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import { articles } from '../data/news';

function ArticleCard({ article, onPress, onBookmark }) {
  return (
    <TouchableOpacity style={styles.card} onPress={() => onPress(article)}>
      <Image source={{ uri: article.image }} style={styles.cardImage} />
      <View style={styles.cardContent}>
        <View style={styles.cardMeta}>
          <Text style={styles.source}>{article.source}</Text>
          <Text style={styles.time}>{article.time}</Text>
        </View>
        <Text style={styles.title} numberOfLines={2}>{article.title}</Text>
        <Text style={styles.summary} numberOfLines={2}>{article.summary}</Text>
        <View style={styles.cardFooter}>
          <Text style={styles.readTime}>⏱ {article.readTime}</Text>
          <View style={styles.footerRight}>
            <Text style={styles.views}>👁 {article.views.toLocaleString()}</Text>
            <TouchableOpacity onPress={() => onBookmark(article.id)}>
              <Ionicons
                name={article.bookmarked ? 'bookmark' : 'bookmark-outline'}
                size={20}
                color={article.bookmarked ? '#6200EE' : '#999'}
              />
            </TouchableOpacity>
          </View>
        </View>
      </View>
    </TouchableOpacity>
  );
}

export default function HomeScreen({ navigation }) {
  const [articleList, setArticleList] = useState(articles);
  const [selectedCategory, setSelectedCategory] = useState('all');

  const filteredArticles = selectedCategory === 'all'
    ? articleList
    : articleList.filter(a => a.category === selectedCategory);

  const handleBookmark = (id) => {
    setArticleList(prev =>
      prev.map(a => a.id === id ? { ...a, bookmarked: !a.bookmarked } : a)
    );
  };

  const categoryTabs = [
    { id: 'all', name: 'ทั้งหมด' },
    { id: 'tech', name: 'เทคโนโลยี' },
    { id: 'business', name: 'ธุรกิจ' },
    { id: 'sports', name: 'กีฬา' },
    { id: 'health', name: 'สุขภาพ' },
  ];

  return (
    <View style={styles.container}>
      {/* Category Filter */}
      <ScrollView
        horizontal
        showsHorizontalScrollIndicator={false}
        style={styles.categoryScroll}
        contentContainerStyle={{ paddingHorizontal: 12 }}
      >
        {categoryTabs.map(cat => (
          <TouchableOpacity
            key={cat.id}
            style={[
              styles.categoryTab,
              selectedCategory === cat.id && styles.activeCategoryTab
            ]}
            onPress={() => setSelectedCategory(cat.id)}
          >
            <Text style={[
              styles.categoryTabText,
              selectedCategory === cat.id && styles.activeCategoryTabText
            ]}>
              {cat.name}
            </Text>
          </TouchableOpacity>
        ))}
      </ScrollView>

      {/* Articles */}
      <FlatList
        data={filteredArticles}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <ArticleCard
            article={item}
            onPress={(article) => navigation.navigate('Article', { article })}
            onBookmark={handleBookmark}
          />
        )}
        contentContainerStyle={styles.list}
        showsVerticalScrollIndicator={false}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  categoryScroll: { backgroundColor: '#FFF', paddingVertical: 8 },
  categoryTab: {
    paddingHorizontal: 16,
    paddingVertical: 6,
    borderRadius: 20,
    marginRight: 8,
    backgroundColor: '#F0F0F0',
  },
  activeCategoryTab: { backgroundColor: '#6200EE' },
  categoryTabText: { color: '#666', fontSize: 13, fontWeight: '500' },
  activeCategoryTabText: { color: '#FFF' },
  list: { padding: 12 },
  card: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    marginBottom: 12,
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 4,
    elevation: 3,
  },
  cardImage: { width: '100%', height: 160 },
  cardContent: { padding: 12 },
  cardMeta: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 6 },
  source: { fontSize: 12, color: '#6200EE', fontWeight: '600' },
  time: { fontSize: 12, color: '#999' },
  title: { fontSize: 16, fontWeight: 'bold', color: '#212121', lineHeight: 22 },
  summary: { fontSize: 13, color: '#666', marginTop: 4, lineHeight: 18 },
  cardFooter: { flexDirection: 'row', justifyContent: 'space-between', marginTop: 10 },
  readTime: { fontSize: 12, color: '#999' },
  footerRight: { flexDirection: 'row', alignItems: 'center', gap: 12 },
  views: { fontSize: 12, color: '#999' },
});
```

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ Drawer สำหรับ secondary navigation (ไม่ใช่ primary)
- สร้าง custom drawer content เสมอสำหรับ app จริง
- เพิ่ม user info ใน drawer header
- Handle การ close drawer หลัง navigate

❌ DON'T:
- ไม่ใช้ Drawer เป็น primary navigation สำหรับแอปที่ใช้งานบ่อย
- ไม่ใส่ menu มากเกินไปใน drawer
- ไม่ลืม import gesture-handler ที่ต้นไฟล์

📝 Drawer Types:
- 'front': Drawer ขึ้นมาทับเนื้อหา (default)
- 'back': เนื้อหาเลื่อนออกให้ Drawer
- 'slide': Drawer และเนื้อหาเลื่อนพร้อมกัน
- 'permanent': Drawer แสดงตลอดเวลา (tablet/desktop)
```

---

## สรุป

Drawer Navigator เหมาะสำหรับ app ที่ต้องการ sidebar navigation:

1. **createDrawerNavigator** - สร้างและกำหนดค่า
2. **Custom Drawer Content** - สร้าง drawer แบบ custom พร้อม profile
3. **Drawer Items** - ใช้ DrawerItem และ DrawerItemList
4. **Programmatic Control** - openDrawer, closeDrawer, toggleDrawer
5. **Workshop** - News App พร้อม sidebar menu สมบูรณ์
