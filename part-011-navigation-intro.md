# Part 011: Navigation ด้วย React Navigation - บทนำ

## สารบัญ
1. ทำไมต้องใช้ React Navigation
2. ติดตั้ง React Navigation
3. NavigationContainer
4. ประเภทของ Navigator
5. Navigation Props
6. Workshop: ตั้งค่า Navigation เบื้องต้น

---

## 1. ทำไมต้องใช้ React Navigation

การนำทาง (Navigation) เป็นหนึ่งในส่วนสำคัญที่สุดของแอปพลิเคชัน Mobile เมื่อแอปมีหลายหน้า เราต้องการระบบที่ช่วยจัดการการเปลี่ยนหน้าได้อย่างราบรื่น

### ปัญหาถ้าไม่มี Navigation Library

```javascript
// ❌ วิธีที่ไม่ดี - จัดการ state เองทั้งหมด
import React, { useState } from 'react';
import { View } from 'react-native';

const HomeScreen = ({ onNavigate }) => (
  <View>
    {/* ต้อง pass callback ลงไปทุกระดับ */}
  </View>
);

const App = () => {
  const [currentScreen, setCurrentScreen] = useState('Home');
  
  // ปัญหา: 
  // 1. ไม่มี back button support
  // 2. ไม่มี animation
  // 3. ไม่มี deep linking
  // 4. State management ยุ่งยาก
  // 5. ไม่รองรับ native navigation patterns
  
  return (
    <View>
      {currentScreen === 'Home' && <HomeScreen onNavigate={setCurrentScreen} />}
      {currentScreen === 'Profile' && <ProfileScreen />}
    </View>
  );
};
```

### ทำไม React Navigation จึงเป็นตัวเลือกที่ดี

**React Navigation** คือ library ที่ได้รับความนิยมสูงสุดสำหรับ React Native Navigation โดยมีจุดเด่นดังนี้:

1. **ครบครัน** - รองรับ Stack, Tab, Drawer navigation
2. **Customizable** - ปรับแต่งได้อย่างยืดหยุ่น
3. **Cross-platform** - ทำงานได้ทั้ง iOS และ Android
4. **TypeScript support** - รองรับ TypeScript อย่างสมบูรณ์
5. **Active community** - มีการอัพเดทและ support อย่างต่อเนื่อง
6. **Deep linking** - รองรับการเปิดแอปจาก URL
7. **Native feel** - Animation และ gesture เหมือน native

### เปรียบเทียบ Navigation Libraries

| Feature | React Navigation | React Native Navigation (Wix) | Expo Router |
|---------|-----------------|-------------------------------|-------------|
| Setup | ง่าย | ยาก | ง่ายมาก |
| Performance | ดี | ดีมาก (Native) | ดี |
| Customization | สูง | สูง | ปานกลาง |
| File-based | ไม่ | ไม่ | ใช่ |
| Expo compatible | ✅ | ❌ | ✅ |

---

## 2. ติดตั้ง React Navigation

### การติดตั้งพื้นฐาน

```bash
# ติดตั้ง core package
npm install @react-navigation/native

# ติดตั้ง dependencies
npm install react-native-screens react-native-safe-area-context
```

### สำหรับ React Native CLI

```bash
# ติดตั้ง pods (iOS เท่านั้น)
cd ios && pod install && cd ..
```

### สำหรับ Expo

```bash
npx expo install react-native-screens react-native-safe-area-context
```

### ติดตั้ง Navigator ที่ต้องการ

```bash
# Stack Navigator
npm install @react-navigation/native-stack

# Tab Navigator
npm install @react-navigation/bottom-tabs

# Drawer Navigator
npm install @react-navigation/drawer
npm install react-native-gesture-handler react-native-reanimated
```

### ตั้งค่า react-native-gesture-handler

```javascript
// index.js หรือ App.js (ต้องอยู่บรรทัดแรกสุด)
import 'react-native-gesture-handler';
```

### ตั้งค่า Android (สำหรับ React Native CLI)

```xml
<!-- android/app/src/main/java/[...]/MainActivity.java -->
import com.facebook.react.ReactActivityDelegate;
import com.facebook.react.defaults.DefaultNewArchitectureEntryPoint;
import com.facebook.react.defaults.DefaultReactActivityDelegate;

public class MainActivity extends ReactActivity {
  @Override
  protected ReactActivityDelegate createReactActivityDelegate() {
    return new DefaultReactActivityDelegate(
      this,
      getMainComponentName(),
      DefaultNewArchitectureEntryPoint.getFabricEnabled()
    );
  }
}
```

### ตรวจสอบ package.json

```json
{
  "dependencies": {
    "@react-navigation/native": "^6.x.x",
    "@react-navigation/native-stack": "^6.x.x",
    "@react-navigation/bottom-tabs": "^6.x.x",
    "@react-navigation/drawer": "^6.x.x",
    "react-native-screens": "^3.x.x",
    "react-native-safe-area-context": "^4.x.x",
    "react-native-gesture-handler": "^2.x.x",
    "react-native-reanimated": "^3.x.x"
  }
}
```

---

## 3. NavigationContainer

`NavigationContainer` คือ component ที่ห่อหุ้ม Navigation ทั้งหมดในแอป ต้องอยู่ที่ root ของ component tree

### การใช้งานพื้นฐาน

```javascript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function HomeScreen() {
  return (
    <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
      <Text>หน้าหลัก</Text>
    </View>
  );
}

function App() {
  return (
    // NavigationContainer ต้องอยู่นอกสุด
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

export default App;
```

### NavigationContainer Props

```javascript
import { NavigationContainer, DefaultTheme, DarkTheme } from '@react-navigation/native';
import { useColorScheme } from 'react-native';

function App() {
  const scheme = useColorScheme();
  
  return (
    <NavigationContainer
      // Theme: กำหนด theme ให้ navigation
      theme={scheme === 'dark' ? DarkTheme : DefaultTheme}
      
      // linking: กำหนด deep link configuration
      linking={{
        prefixes: ['myapp://', 'https://myapp.com'],
        config: {
          screens: {
            Home: 'home',
            Profile: 'user/:id',
          },
        },
      }}
      
      // onReady: callback เมื่อ navigation พร้อมใช้งาน
      onReady={() => {
        console.log('Navigation is ready');
      }}
      
      // onStateChange: callback เมื่อ navigation state เปลี่ยน
      onStateChange={(state) => {
        console.log('New state:', state);
      }}
      
      // ref: สำหรับ navigate จากนอก component
      ref={navigationRef}
    >
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### Custom Theme

```javascript
import { NavigationContainer } from '@react-navigation/native';

const MyTheme = {
  dark: false,
  colors: {
    primary: '#6200EE',      // สี primary (links, buttons)
    background: '#F5F5F5',   // สีพื้นหลัง
    card: '#FFFFFF',          // สี card/header
    text: '#212121',          // สีข้อความ
    border: '#E0E0E0',        // สีเส้นขอบ
    notification: '#FF4444',  // สี badge notification
  },
};

function App() {
  return (
    <NavigationContainer theme={MyTheme}>
      {/* ... */}
    </NavigationContainer>
  );
}
```

### Navigation Ref - Navigate จากนอก Component

```javascript
import React, { useRef } from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNavigationContainerRef } from '@react-navigation/native';

// สร้าง ref สำหรับใช้ outside component
export const navigationRef = createNavigationContainerRef();

export function navigate(name, params) {
  if (navigationRef.isReady()) {
    navigationRef.navigate(name, params);
  }
}

// App.js
function App() {
  return (
    <NavigationContainer ref={navigationRef}>
      {/* ... */}
    </NavigationContainer>
  );
}

// ใช้ navigate จาก service หรือ Redux action
import { navigate } from './navigation/NavigationService';

// ในที่อื่น
function handleNotification() {
  navigate('Profile', { userId: '123' });
}
```

---

## 4. ประเภทของ Navigator

React Navigation มี Navigator หลักๆ 3 ประเภท

### 4.1 Stack Navigator

เหมาะสำหรับการนำทางแบบ stack (กองหน้า) เหมือนการซ้อนการ์ด

```javascript
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function AppNavigator() {
  return (
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Detail" component={DetailScreen} />
      <Stack.Screen name="Profile" component={ProfileScreen} />
    </Stack.Navigator>
  );
}
```

**เหมาะกับ:**
- App ที่มีหลายหน้าและต้องการ back navigation
- E-commerce: รายการสินค้า → รายละเอียด → ชำระเงิน
- Social: Feed → Post → Comment

### 4.2 Tab Navigator

แสดง Tab bar ด้านล่างหรือด้านบน

```javascript
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function AppNavigator() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

**เหมาะกับ:**
- App หลักที่มี 3-5 section หลัก
- Social media apps
- Banking apps

### 4.3 Drawer Navigator

แสดง sidebar menu

```javascript
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

function AppNavigator() {
  return (
    <Drawer.Navigator>
      <Drawer.Screen name="Home" component={HomeScreen} />
      <Drawer.Screen name="Settings" component={SettingsScreen} />
      <Drawer.Screen name="About" component={AboutScreen} />
    </Drawer.Navigator>
  );
}
```

**เหมาะกับ:**
- App ที่มีหลาย section แต่ไม่ใช้บ่อยเท่ากัน
- Admin panels
- News apps

---

## 5. Navigation Props

เมื่อ Screen ถูกเรียกผ่าน Navigator มันจะได้รับ props สองตัวคือ `navigation` และ `route`

### navigation prop

```javascript
function HomeScreen({ navigation }) {
  return (
    <View>
      {/* navigate: ไปหน้าอื่น */}
      <Button 
        title="ไปหน้า Detail" 
        onPress={() => navigation.navigate('Detail')} 
      />
      
      {/* navigate พร้อม params */}
      <Button 
        title="ไปหน้า Profile" 
        onPress={() => navigation.navigate('Profile', { userId: '123', name: 'สมชาย' })} 
      />
      
      {/* push: เพิ่มหน้าใน stack (แม้จะเป็นหน้าเดิม) */}
      <Button 
        title="Push หน้าใหม่" 
        onPress={() => navigation.push('Detail', { id: Math.random() })} 
      />
      
      {/* goBack: กลับหน้าก่อนหน้า */}
      <Button 
        title="กลับ" 
        onPress={() => navigation.goBack()} 
      />
      
      {/* replace: แทนที่หน้าปัจจุบัน */}
      <Button 
        title="Replace" 
        onPress={() => navigation.replace('NewScreen')} 
      />
      
      {/* reset: รีเซ็ต navigation stack */}
      <Button 
        title="ไปหน้าแรก" 
        onPress={() => navigation.reset({
          index: 0,
          routes: [{ name: 'Home' }],
        })} 
      />
    </View>
  );
}
```

### route prop

```javascript
function ProfileScreen({ route, navigation }) {
  // รับ params จากหน้าที่ navigate มา
  const { userId, name } = route.params;
  
  return (
    <View>
      <Text>User ID: {userId}</Text>
      <Text>ชื่อ: {name}</Text>
      <Text>Route name: {route.name}</Text>
    </View>
  );
}
```

### useNavigation Hook

สำหรับ component ที่ไม่ได้เป็น Screen โดยตรง

```javascript
import { useNavigation } from '@react-navigation/native';

// Component ทั่วไปที่ต้องการ navigate
function LoginButton() {
  const navigation = useNavigation();
  
  return (
    <Button 
      title="เข้าสู่ระบบ" 
      onPress={() => navigation.navigate('Login')} 
    />
  );
}
```

### useRoute Hook

```javascript
import { useRoute } from '@react-navigation/native';

function UserAvatar() {
  const route = useRoute();
  const { userId } = route.params;
  
  return <Image source={{ uri: `https://api.example.com/avatar/${userId}` }} />;
}
```

### useFocusEffect Hook

ทำงานเมื่อหน้าได้รับ focus (มองเห็น)

```javascript
import { useFocusEffect } from '@react-navigation/native';
import { useCallback } from 'react';

function ProfileScreen() {
  useFocusEffect(
    useCallback(() => {
      // ทำงานเมื่อหน้านี้ focus
      console.log('หน้านี้กำลังแสดงอยู่');
      fetchUserData();
      
      return () => {
        // Cleanup เมื่อออกจากหน้า
        console.log('ออกจากหน้านี้แล้ว');
      };
    }, [])
  );
  
  return <View>{/* ... */}</View>;
}
```

### useIsFocused Hook

```javascript
import { useIsFocused } from '@react-navigation/native';

function HomeScreen() {
  const isFocused = useIsFocused();
  
  return (
    <View>
      <Text>
        {isFocused ? 'หน้านี้กำลังแสดงอยู่' : 'หน้านี้ซ่อนอยู่'}
      </Text>
    </View>
  );
}
```

---

## 6. Screen Options

### Static Options

```javascript
<Stack.Screen
  name="Home"
  component={HomeScreen}
  options={{
    title: 'หน้าหลัก',
    headerStyle: {
      backgroundColor: '#6200EE',
    },
    headerTintColor: '#fff',
    headerTitleStyle: {
      fontWeight: 'bold',
      fontSize: 18,
    },
  }}
/>
```

### Dynamic Options

```javascript
// ใน component
function HomeScreen({ navigation }) {
  useEffect(() => {
    navigation.setOptions({
      title: 'หน้าหลัก (อัพเดท)',
      headerRight: () => (
        <Button onPress={() => alert('กด!')} title="Info" />
      ),
    });
  }, [navigation]);
  
  return <View>{/* ... */}</View>;
}
```

### Options จาก Route Params

```javascript
<Stack.Screen
  name="Profile"
  component={ProfileScreen}
  options={({ route }) => ({
    title: `โปรไฟล์ของ ${route.params.name}`,
  })}
/>
```

---

## 7. TypeScript กับ React Navigation

```typescript
// types/navigation.ts
import { NativeStackScreenProps } from '@react-navigation/native-stack';

// กำหนด type ของ params แต่ละหน้า
export type RootStackParamList = {
  Home: undefined;  // ไม่มี params
  Profile: { userId: string; name: string };
  Settings: { initialTab?: string };
};

// Type สำหรับ Screen props
export type HomeScreenProps = NativeStackScreenProps<RootStackParamList, 'Home'>;
export type ProfileScreenProps = NativeStackScreenProps<RootStackParamList, 'Profile'>;
```

```typescript
// screens/HomeScreen.tsx
import { HomeScreenProps } from '../types/navigation';

function HomeScreen({ navigation, route }: HomeScreenProps) {
  return (
    <View>
      <Button
        title="ไปหน้า Profile"
        onPress={() => navigation.navigate('Profile', {
          userId: '123',
          name: 'สมชาย',
        })}
      />
    </View>
  );
}
```

---

## Workshop: ตั้งค่า Navigation เบื้องต้น

### เป้าหมาย
สร้างแอปที่มี 3 หน้า: Home, About, Contact พร้อม Stack Navigator

### โครงสร้างไฟล์

```
MyApp/
├── App.js
├── screens/
│   ├── HomeScreen.js
│   ├── AboutScreen.js
│   └── ContactScreen.js
└── navigation/
    └── AppNavigator.js
```

### ขั้นตอนที่ 1: ติดตั้ง Dependencies

```bash
npm install @react-navigation/native @react-navigation/native-stack
npx expo install react-native-screens react-native-safe-area-context
```

### ขั้นตอนที่ 2: สร้าง Screens

```javascript
// screens/HomeScreen.js
import React from 'react';
import { View, Text, StyleSheet, TouchableOpacity } from 'react-native';

export default function HomeScreen({ navigation }) {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>🏠 หน้าหลัก</Text>
      <Text style={styles.subtitle}>ยินดีต้อนรับสู่แอปของเรา!</Text>
      
      <TouchableOpacity 
        style={styles.button}
        onPress={() => navigation.navigate('About')}
      >
        <Text style={styles.buttonText}>เกี่ยวกับเรา →</Text>
      </TouchableOpacity>
      
      <TouchableOpacity 
        style={[styles.button, styles.secondButton]}
        onPress={() => navigation.navigate('Contact')}
      >
        <Text style={styles.buttonText}>ติดต่อเรา →</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: '#F5F5F5',
    padding: 20,
  },
  title: {
    fontSize: 32,
    fontWeight: 'bold',
    marginBottom: 8,
    color: '#212121',
  },
  subtitle: {
    fontSize: 16,
    color: '#666',
    marginBottom: 40,
    textAlign: 'center',
  },
  button: {
    backgroundColor: '#6200EE',
    paddingHorizontal: 30,
    paddingVertical: 15,
    borderRadius: 10,
    marginBottom: 15,
    width: '80%',
    alignItems: 'center',
  },
  secondButton: {
    backgroundColor: '#03DAC6',
  },
  buttonText: {
    color: '#FFF',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

```javascript
// screens/AboutScreen.js
import React from 'react';
import { View, Text, StyleSheet, ScrollView, TouchableOpacity } from 'react-native';

export default function AboutScreen({ navigation }) {
  return (
    <ScrollView contentContainerStyle={styles.container}>
      <Text style={styles.title}>ℹ️ เกี่ยวกับเรา</Text>
      
      <View style={styles.card}>
        <Text style={styles.cardTitle}>บริษัท ABC จำกัด</Text>
        <Text style={styles.cardText}>
          เราเป็นบริษัทเทคโนโลยีที่ก่อตั้งในปี 2020 
          มุ่งมั่นสร้างผลิตภัณฑ์ที่มีคุณภาพเพื่อลูกค้าของเรา
        </Text>
      </View>
      
      <View style={styles.card}>
        <Text style={styles.cardTitle}>วิสัยทัศน์</Text>
        <Text style={styles.cardText}>
          เป็นบริษัทเทคโนโลยีชั้นนำในภูมิภาคเอเชียตะวันออกเฉียงใต้
        </Text>
      </View>
      
      <TouchableOpacity
        style={styles.button}
        onPress={() => navigation.navigate('Contact')}
      >
        <Text style={styles.buttonText}>ติดต่อเรา</Text>
      </TouchableOpacity>
      
      <TouchableOpacity
        style={styles.backButton}
        onPress={() => navigation.goBack()}
      >
        <Text style={styles.backButtonText}>← กลับ</Text>
      </TouchableOpacity>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    padding: 20,
    backgroundColor: '#F5F5F5',
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    marginBottom: 20,
    color: '#212121',
    textAlign: 'center',
  },
  card: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    padding: 16,
    marginBottom: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  cardTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 8,
    color: '#6200EE',
  },
  cardText: {
    fontSize: 15,
    color: '#555',
    lineHeight: 22,
  },
  button: {
    backgroundColor: '#6200EE',
    padding: 15,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 10,
  },
  buttonText: {
    color: '#FFF',
    fontSize: 16,
    fontWeight: '600',
  },
  backButton: {
    padding: 15,
    alignItems: 'center',
  },
  backButtonText: {
    color: '#6200EE',
    fontSize: 16,
  },
});
```

```javascript
// screens/ContactScreen.js
import React, { useState } from 'react';
import { 
  View, Text, StyleSheet, TextInput, 
  TouchableOpacity, Alert, ScrollView 
} from 'react-native';

export default function ContactScreen({ navigation }) {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [message, setMessage] = useState('');

  const handleSubmit = () => {
    if (!name || !email || !message) {
      Alert.alert('ข้อผิดพลาด', 'กรุณากรอกข้อมูลให้ครบ');
      return;
    }
    Alert.alert(
      'ส่งสำเร็จ!', 
      `ขอบคุณ ${name} เราจะติดต่อกลับที่ ${email}`,
      [{ text: 'ตกลง', onPress: () => navigation.navigate('Home') }]
    );
  };

  return (
    <ScrollView contentContainerStyle={styles.container}>
      <Text style={styles.title}>📧 ติดต่อเรา</Text>
      
      <View style={styles.form}>
        <Text style={styles.label}>ชื่อ</Text>
        <TextInput
          style={styles.input}
          placeholder="กรอกชื่อของคุณ"
          value={name}
          onChangeText={setName}
        />
        
        <Text style={styles.label}>อีเมล</Text>
        <TextInput
          style={styles.input}
          placeholder="example@email.com"
          value={email}
          onChangeText={setEmail}
          keyboardType="email-address"
          autoCapitalize="none"
        />
        
        <Text style={styles.label}>ข้อความ</Text>
        <TextInput
          style={[styles.input, styles.textarea]}
          placeholder="พิมพ์ข้อความของคุณ..."
          value={message}
          onChangeText={setMessage}
          multiline
          numberOfLines={4}
          textAlignVertical="top"
        />
        
        <TouchableOpacity style={styles.button} onPress={handleSubmit}>
          <Text style={styles.buttonText}>ส่งข้อความ</Text>
        </TouchableOpacity>
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    padding: 20,
    backgroundColor: '#F5F5F5',
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    marginBottom: 24,
    color: '#212121',
    textAlign: 'center',
  },
  form: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    padding: 20,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  label: {
    fontSize: 15,
    fontWeight: '600',
    marginBottom: 6,
    color: '#333',
  },
  input: {
    borderWidth: 1,
    borderColor: '#DDD',
    borderRadius: 8,
    padding: 12,
    fontSize: 15,
    marginBottom: 16,
    backgroundColor: '#FAFAFA',
  },
  textarea: {
    height: 100,
  },
  button: {
    backgroundColor: '#6200EE',
    padding: 15,
    borderRadius: 10,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonText: {
    color: '#FFF',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### ขั้นตอนที่ 3: สร้าง AppNavigator

```javascript
// navigation/AppNavigator.js
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import HomeScreen from '../screens/HomeScreen';
import AboutScreen from '../screens/AboutScreen';
import ContactScreen from '../screens/ContactScreen';

const Stack = createNativeStackNavigator();

export default function AppNavigator() {
  return (
    <Stack.Navigator
      initialRouteName="Home"
      screenOptions={{
        headerStyle: {
          backgroundColor: '#6200EE',
        },
        headerTintColor: '#fff',
        headerTitleStyle: {
          fontWeight: 'bold',
        },
      }}
    >
      <Stack.Screen 
        name="Home" 
        component={HomeScreen}
        options={{ title: 'หน้าหลัก' }}
      />
      <Stack.Screen 
        name="About" 
        component={AboutScreen}
        options={{ title: 'เกี่ยวกับเรา' }}
      />
      <Stack.Screen 
        name="Contact" 
        component={ContactScreen}
        options={{ title: 'ติดต่อเรา' }}
      />
    </Stack.Navigator>
  );
}
```

### ขั้นตอนที่ 4: App.js หลัก

```javascript
// App.js
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import AppNavigator from './navigation/AppNavigator';

export default function App() {
  return (
    <NavigationContainer>
      <AppNavigator />
    </NavigationContainer>
  );
}
```

### ผลลัพธ์ที่ได้

แอปจะมี:
- Header สีม่วงพร้อม Back button อัตโนมัติ
- หน้า Home พร้อมปุ่มไปหน้าอื่น
- หน้า About แสดงข้อมูลบริษัท
- หน้า Contact พร้อม Form

---

## แบบฝึกหัดเพิ่มเติม

### แบบฝึกหัดที่ 1: เพิ่ม Header Button
เพิ่มปุ่มแชร์ใน Header ของหน้า About

```javascript
// ใน AboutScreen.js
useEffect(() => {
  navigation.setOptions({
    headerRight: () => (
      <TouchableOpacity onPress={() => Share.share({ message: 'เรียนรู้ React Navigation!' })}>
        <Text style={{ color: '#FFF', marginRight: 10 }}>แชร์</Text>
      </TouchableOpacity>
    ),
  });
}, [navigation]);
```

### แบบฝึกหัดที่ 2: ส่งข้อมูลระหว่างหน้า
ปรับ ContactScreen ให้แสดงชื่อที่รับมาจาก HomeScreen

### แบบฝึกหัดที่ 3: เพิ่ม Loading Screen
สร้าง SplashScreen ที่แสดง 2 วินาทีก่อนไปหน้า Home

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ React Navigation** - จัดการ navigation อย่างมีประสิทธิภาพ
2. **การติดตั้ง** - core package และ navigator ต่างๆ
3. **NavigationContainer** - component ที่ห่อหุ้มทุกอย่าง
4. **Navigator types** - Stack, Tab, Drawer
5. **Navigation Props** - navigation และ route
6. **Workshop** - สร้างแอป 3 หน้าพื้นฐาน

ในบทต่อไปเราจะเจาะลึก **Stack Navigator** ซึ่งเป็น Navigator ที่ใช้งานบ่อยที่สุด

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ NavigationContainer ที่ root เสมอ
- กำหนด TypeScript types สำหรับ navigation params
- ใช้ useFocusEffect แทน useEffect สำหรับ data fetching ที่ต้องอัพเดททุกครั้ง
- จัดการ navigation logic ใน navigator file ไม่ใช่ใน screen

❌ DON'T:
- ไม่ใส่ NavigationContainer ซ้อนกัน
- ไม่ hardcode route names - ใช้ constants แทน
- ไม่ navigate โดยตรงใน useEffect โดยไม่ตรวจสอบ mounted state
```
