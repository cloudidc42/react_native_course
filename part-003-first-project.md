# Part 003: สร้าง Project แรกด้วย React Native

## สารบัญ
1. [สร้าง Project ด้วย CLI](#สร้าง-project)
2. [โครงสร้างไฟล์เบื้องต้น](#โครงสร้างไฟล์)
3. [รัน App บน Emulator/Simulator](#รัน-app)
4. [การใช้งาน Dev Menu](#dev-menu)
5. [Hot Reload และ Fast Refresh](#hot-reload)
6. [Debug เบื้องต้น](#debug)
7. [ทำความรู้จัก App.tsx](#app-tsx)
8. [Workshop: Hello World App](#workshop)

---

## สร้าง Project

### วิธีที่ 1: React Native Community CLI

```bash
# วิธีที่แนะนำ - ใช้ npx (ไม่ต้อง global install)
npx @react-native-community/cli@latest init HelloWorldApp

# ระบุ version ที่ต้องการ
npx @react-native-community/cli@latest init HelloWorldApp --version 0.74.0

# ใช้ TypeScript template (แนะนำ - default แล้ว)
npx @react-native-community/cli@latest init HelloWorldApp --template react-native-template-typescript

# ระบุ directory ปลายทาง
npx @react-native-community/cli@latest init HelloWorldApp --directory /path/to/projects
```

**Output ที่จะเห็น:**

```
✔ Downloading template
✔ Copying template
✔ Processing template
✔ Installing dependencies
✔ Installing CocoaPods dependencies (macOS only)

  Run instructions for Android:
    • Have an Android emulator running, or a device connected
    • cd "/path/to/HelloWorldApp" && npx react-native run-android

  Run instructions for iOS:
    • cd "/path/to/HelloWorldApp" && npx react-native run-ios
```

### วิธีที่ 2: Expo CLI

```bash
# สร้าง Expo project
npx create-expo-app HelloWorldExpo

# เลือก template
npx create-expo-app HelloWorldExpo --template

# Templates ที่มี:
# - blank (JavaScript)
# - blank-typescript (TypeScript)
# - tabs (Navigation)
# - bare-minimum

cd HelloWorldExpo
npx expo start
```

### เปิด Project ใน VS Code

```bash
cd HelloWorldApp
code .
```

---

## โครงสร้างไฟล์

หลังจากสร้าง project ด้วย React Native CLI โครงสร้างจะเป็น:

```
HelloWorldApp/
├── android/                  # Android native code
│   ├── app/
│   │   ├── src/
│   │   │   └── main/
│   │   │       ├── AndroidManifest.xml
│   │   │       └── java/com/helloworldapp/
│   │   │           └── MainActivity.kt
│   │   └── build.gradle
│   ├── gradle/
│   ├── build.gradle
│   └── settings.gradle
├── ios/                      # iOS native code
│   ├── HelloWorldApp/
│   │   ├── AppDelegate.swift
│   │   ├── Info.plist
│   │   └── main.m
│   ├── HelloWorldApp.xcodeproj/
│   ├── HelloWorldApp.xcworkspace/
│   └── Podfile
├── src/                      # ยังว่างอยู่ ต้องสร้างเอง (best practice)
├── __tests__/                # Test files
│   └── App.test.tsx
├── .eslintrc.js              # ESLint config
├── .prettierrc.js            # Prettier config
├── .watchmanconfig           # Watchman config
├── app.json                  # App configuration
├── babel.config.js           # Babel config
├── index.js                  # Entry point
├── jest.config.js            # Jest config
├── metro.config.js           # Metro bundler config
├── package.json              # Project dependencies
├── tsconfig.json             # TypeScript config
└── App.tsx                   # Root component
```

### ไฟล์สำคัญ

**index.js** - Entry Point หลักของแอป

```javascript
/**
 * @format
 */
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

// ลงทะเบียน Root Component
AppRegistry.registerComponent(appName, () => App);
```

**app.json** - Configuration ของแอป

```json
{
  "name": "HelloWorldApp",
  "displayName": "Hello World App"
}
```

**App.tsx** - Root Component (ที่เราจะแก้ไขเป็นหลัก)

```tsx
import React from 'react';
import {
  SafeAreaView,
  ScrollView,
  StatusBar,
  StyleSheet,
  Text,
  useColorScheme,
  View,
} from 'react-native';

import {
  Colors,
  DebugInstructions,
  Header,
  LearnMoreLinks,
  ReloadInstructions,
} from 'react-native/Libraries/NewAppScreen';

// TypeScript: กำหนด type ของ props
type SectionProps = {
  children: React.ReactNode;
  title: string;
};

// Functional Component
function Section({ children, title }: SectionProps): React.JSX.Element {
  const isDarkMode = useColorScheme() === 'dark';
  return (
    <View style={styles.sectionContainer}>
      <Text
        style={[
          styles.sectionTitle,
          { color: isDarkMode ? Colors.white : Colors.black },
        ]}>
        {title}
      </Text>
      <Text
        style={[
          styles.sectionDescription,
          { color: isDarkMode ? Colors.light : Colors.dark },
        ]}>
        {children}
      </Text>
    </View>
  );
}

// Root Component
function App(): React.JSX.Element {
  const isDarkMode = useColorScheme() === 'dark';

  const backgroundStyle = {
    backgroundColor: isDarkMode ? Colors.darker : Colors.lighter,
  };

  return (
    <SafeAreaView style={backgroundStyle}>
      <StatusBar
        barStyle={isDarkMode ? 'light-content' : 'dark-content'}
        backgroundColor={backgroundStyle.backgroundColor}
      />
      <ScrollView
        contentInsetAdjustmentBehavior="automatic"
        style={backgroundStyle}>
        <Header />
        <View
          style={{
            backgroundColor: isDarkMode ? Colors.black : Colors.white,
          }}>
          <Section title="Step One">
            Edit <Text style={styles.highlight}>App.tsx</Text> to change this
            screen and then come back to see your edits.
          </Section>
          <Section title="See Your Changes">
            <ReloadInstructions />
          </Section>
          <Section title="Debug">
            <DebugInstructions />
          </Section>
          <Section title="Learn More">
            Read the docs to discover what to do next:
          </Section>
          <LearnMoreLinks />
        </View>
      </ScrollView>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  sectionContainer: {
    marginTop: 32,
    paddingHorizontal: 24,
  },
  sectionTitle: {
    fontSize: 24,
    fontWeight: '600',
  },
  sectionDescription: {
    marginTop: 8,
    fontSize: 18,
    fontWeight: '400',
  },
  highlight: {
    fontWeight: '700',
  },
});

export default App;
```

---

## รัน App

### รัน Android

```bash
# ขั้นตอนที่ 1: ตรวจสอบ Emulator หรือ Device
adb devices
# ควรเห็น: emulator-5554 device

# ขั้นตอนที่ 2: รัน Metro Bundler (Terminal 1)
npx react-native start

# ขั้นตอนที่ 3: รัน Android (Terminal 2)
npx react-native run-android

# หรือรันพร้อมกัน (ใช้ได้เฉพาะบาง terminal)
npx react-native run-android --active-arch-only

# ระบุ device
npx react-native run-android --deviceId emulator-5554

# Build Debug APK
npx react-native run-android --mode debug

# Build Release APK
npx react-native run-android --mode release
```

### รัน iOS (macOS)

```bash
# ขั้นตอนที่ 1: ติดตั้ง pods (ครั้งแรกหรือเมื่อเพิ่ม dependency)
cd ios && pod install && cd ..

# ขั้นตอนที่ 2: รัน
npx react-native run-ios

# ระบุ simulator
npx react-native run-ios --simulator "iPhone 15"
npx react-native run-ios --simulator "iPhone 15 Pro Max"

# ดู simulator ที่มี
xcrun simctl list devices | grep "iPhone"

# รันบน physical device
npx react-native run-ios --device "iPhone ของฉัน"
```

### Metro Bundler

Metro คือ JavaScript bundler ของ React Native

```bash
# รัน Metro
npx react-native start

# รัน Metro พร้อม reset cache
npx react-native start --reset-cache

# รัน Metro บน port อื่น
npx react-native start --port 8082

# Output ที่เห็น:
# ┌──────────────────────────────────────────────────────────┐
# │                                                          │
# │  Metro waiting on exp+http://localhost:8081              │
# │                                                          │
# │  Scan the QR code above with Expo Go (Android) or the   │
# │  iOS Camera app to open your project.                    │
# │                                                          │
# │  › Press a │ open Android                               │
# │  › Press i │ open iOS simulator                         │
# │  › Press w │ open web                                   │
# │  › Press r │ reload app                                 │
# │  › Press d │ open Dev tools                             │
# │  › Press shift+d │ toggle auto opening DevTools         │
# │                                                          │
# └──────────────────────────────────────────────────────────┘
```

---

## Dev Menu

Developer Menu เป็นเมนูพิเศษสำหรับ Debug

### วิธีเปิด Dev Menu

**Android Emulator:**
- กด `Ctrl+M` (Windows/Linux)
- กด `Cmd+M` (macOS)
- หรือเขย่า device (สำหรับ physical device)

**iOS Simulator:**
- กด `Ctrl+D`
- หรือ Device > Shake (Cmd+Ctrl+Z)

### ตัวเลือกใน Dev Menu

```
Developer Menu:
┌─────────────────────────────────┐
│ Reload                          │  → รีโหลดแอป
│ Debug Remote JS                 │  → Debug ใน Chrome
│ Toggle Inspector                │  → ดู element inspector
│ Performance Monitor             │  → ดู FPS/memory
│ Toggle Fast Refresh             │  → เปิด/ปิด Fast Refresh
│ Settings                        │  → การตั้งค่าเพิ่มเติม
└─────────────────────────────────┘
```

### Reload App

```bash
# วิธีที่ 1: กด R สองครั้ง (Android Emulator)
# วิธีที่ 2: Dev Menu > Reload
# วิธีที่ 3: Shake device

# วิธีที่ 4: จาก Metro terminal - กด r
r  # กด r ใน Metro terminal
```

---

## Hot Reload และ Fast Refresh

### Hot Reload (เก่า)

Hot Reload คือระบบที่อัปเดต JavaScript ที่เปลี่ยนแปลงโดยไม่ต้อง reload ทั้งแอป แต่ยังคง state ไว้

### Fast Refresh (ใหม่ - ค่าเริ่มต้นใน RN 0.61+)

Fast Refresh เป็นระบบใหม่ที่ดีกว่า Hot Reload:

```
Fast Refresh Features:
1. อัปเดตเร็วขึ้น
2. รักษา state ไว้ (เมื่อแก้ไข component เดียว)
3. แสดง error ที่ชัดเจนขึ้น
4. ทำงานได้กับ Hooks
5. Full reload เมื่อจำเป็น (เช่น แก้ไข module-level code)
```

**การทำงานของ Fast Refresh:**

```
แก้ไขไฟล์
      ↓
Metro Bundler ตรวจจับการเปลี่ยนแปลง
      ↓
คำนวณ minimal update
      ↓
ส่ง update ไปยัง app
      ↓
React re-renders component ที่เปลี่ยน
      ↓
State ยังคงอยู่ (ถ้าไม่ได้แก้ state structure)
```

**ทดสอบ Fast Refresh:**

```tsx
// App.tsx - ทดสอบ Fast Refresh
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

const App = () => {
  const [count, setCount] = useState(0);

  return (
    <View style={styles.container}>
      <Text style={styles.count}>{count}</Text>
      <TouchableOpacity
        style={styles.button}
        onPress={() => setCount(count + 1)}
      >
        <Text style={styles.buttonText}>กด +1</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#fff',
  },
  count: {
    fontSize: 72,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 20,
  },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 32,
    paddingVertical: 16,
    borderRadius: 8,
  },
  buttonText: {
    color: '#fff',
    fontSize: 18,
    fontWeight: '600',
  },
});

export default App;
```

**ทดสอบ:** กดปุ่มจน count เป็น 5 แล้วเปลี่ยนสีปุ่มเป็น `#FF3B30` และบันทึก - count ควรยังอยู่ที่ 5

### เมื่อไหร่ที่ Fast Refresh จะ Full Reload

1. แก้ไข module-level code (นอก function)
2. เพิ่ม/ลบ Hooks
3. Error ที่ไม่สามารถ recover ได้
4. แก้ไข `index.js`

---

## Debug เบื้องต้น

### React Native Debugger

```bash
# ดาวน์โหลด React Native Debugger
# https://github.com/jhen0409/react-native-debugger

# macOS
brew install --cask react-native-debugger

# หรือดาวน์โหลด .dmg จาก GitHub releases
```

### Chrome DevTools

```bash
# เปิด Dev Menu > Debug Remote JS
# Chrome จะเปิดที่ http://localhost:8081/debugger-ui

# หรือเปิดตรง
open http://localhost:8081/debugger-ui
```

### Console.log

```tsx
// Debug ด้วย console.log
const App = () => {
  const data = { name: 'สมชาย', age: 25 };
  
  console.log('data:', data);              // object
  console.log('name:', data.name);        // string
  console.warn('คำเตือน!');               // warning (สีเหลือง)
  console.error('เกิดข้อผิดพลาด!');       // error (สีแดง)
  console.info('ข้อมูล');                  // info
  console.debug('debug info');            // debug
  
  // จัดรูปแบบ
  console.log(`Name: ${data.name}, Age: ${data.age}`);
  
  // เปรียบเทียบ object
  console.log(JSON.stringify(data, null, 2));
  
  return null;
};
```

**ดู logs ใน terminal:**

```bash
# Android
adb logcat | grep ReactNative

# หรือ
npx react-native log-android

# iOS
npx react-native log-ios
```

### Flipper (ตัวเลือก)

Flipper เป็น Desktop app สำหรับ Debug React Native

```bash
# ดาวน์โหลด Flipper
# https://fbflipper.com/

# Features:
# - Network inspector
# - Layout inspector
# - Crash reporter
# - React DevTools integration
# - Database viewer
```

### Error Boundary

```tsx
import React from 'react';
import { View, Text } from 'react-native';

// Error Boundary Component
class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; error: Error | null }
> {
  constructor(props: any) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // Log error to crash reporting service
    console.error('Caught error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <View>
          <Text>เกิดข้อผิดพลาด: {this.state.error?.message}</Text>
        </View>
      );
    }

    return this.props.children;
  }
}

// ใช้งาน
const App = () => (
  <ErrorBoundary>
    <MyComponent />
  </ErrorBoundary>
);
```

---

## ทำความรู้จัก App.tsx

### ทำความเข้าใจโค้ด Default

```tsx
// 1. Imports
import React from 'react';
// React ต้องอยู่ใน scope เสมอ (React 17+ ไม่จำเป็น แต่ยังนิยม)

import {
  SafeAreaView,   // หลีกเลี่ยง notch/status bar
  ScrollView,     // Scrollable container
  StatusBar,      // Control status bar appearance
  StyleSheet,     // Create optimized styles
  Text,           // แสดงข้อความ
  useColorScheme, // ดึง dark/light mode
  View,           // Container component
} from 'react-native';

// 2. TypeScript Type Definition
type SectionProps = {
  children: React.ReactNode;
  title: string;
};

// 3. Sub-component
function Section({ children, title }: SectionProps): React.JSX.Element {
  const isDarkMode = useColorScheme() === 'dark';
  
  return (
    <View style={styles.sectionContainer}>
      <Text style={[
        styles.sectionTitle,
        // Array ของ styles - จะ merge กัน
        { color: isDarkMode ? Colors.white : Colors.black }
      ]}>
        {title}
      </Text>
      <Text style={[
        styles.sectionDescription,
        { color: isDarkMode ? Colors.light : Colors.dark }
      ]}>
        {children}
      </Text>
    </View>
  );
}

// 4. Root Component
function App(): React.JSX.Element {
  const isDarkMode = useColorScheme() === 'dark';
  
  const backgroundStyle = {
    backgroundColor: isDarkMode ? Colors.darker : Colors.lighter,
  };

  return (
    // SafeAreaView - จัดการ safe area (notch, status bar)
    <SafeAreaView style={backgroundStyle}>
      
      {/* StatusBar - ควบคุมสีของ status bar */}
      <StatusBar
        barStyle={isDarkMode ? 'light-content' : 'dark-content'}
        backgroundColor={backgroundStyle.backgroundColor}
      />
      
      {/* ScrollView - ทำให้ scroll ได้ */}
      <ScrollView
        contentInsetAdjustmentBehavior="automatic"
        style={backgroundStyle}>
        
        {/* Content */}
        <View>
          <Section title="Step One">
            Edit App.tsx to change this screen
          </Section>
        </View>
      </ScrollView>
    </SafeAreaView>
  );
}

// 5. StyleSheet - สร้าง styles แบบ optimized
const styles = StyleSheet.create({
  sectionContainer: {
    marginTop: 32,
    paddingHorizontal: 24,
  },
  sectionTitle: {
    fontSize: 24,
    fontWeight: '600',
  },
  sectionDescription: {
    marginTop: 8,
    fontSize: 18,
    fontWeight: '400',
  },
  highlight: {
    fontWeight: '700',
  },
});

// 6. Export - ส่งออก Root Component
export default App;
```

---

## Workshop: Hello World App

### Workshop 3.1: สร้าง Simple Hello World

แก้ไข `App.tsx` ให้แสดงหน้าจอ Hello World แบบง่ายๆ:

```tsx
import React from 'react';
import {
  View,
  Text,
  StyleSheet,
  SafeAreaView,
  StatusBar,
} from 'react-native';

const App = () => {
  return (
    <SafeAreaView style={styles.safeArea}>
      <StatusBar barStyle="dark-content" backgroundColor="#fff" />
      <View style={styles.container}>
        <Text style={styles.title}>สวัสดี React Native! 🎉</Text>
        <Text style={styles.subtitle}>แอปแรกของฉัน</Text>
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  safeArea: {
    flex: 1,
    backgroundColor: '#fff',
  },
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f0f0f0',
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 10,
    textAlign: 'center',
  },
  subtitle: {
    fontSize: 18,
    color: '#666',
  },
});

export default App;
```

### Workshop 3.2: Profile Card App

สร้าง Profile Card ที่มี:
- รูปภาพ (ใช้ placeholder)
- ชื่อ
- อาชีพ
- ปุ่ม Follow

```tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  SafeAreaView,
  StatusBar,
} from 'react-native';

// Profile Card Component
const ProfileCard = () => {
  const [isFollowing, setIsFollowing] = useState(false);
  const [followers, setFollowers] = useState(1234);

  const handleFollow = () => {
    setIsFollowing(!isFollowing);
    setFollowers(isFollowing ? followers - 1 : followers + 1);
  };

  return (
    <View style={styles.card}>
      {/* Cover Image */}
      <View style={styles.coverImage} />
      
      {/* Profile Image */}
      <View style={styles.profileImageContainer}>
        <Image
          source={{
            uri: 'https://randomuser.me/api/portraits/men/1.jpg',
          }}
          style={styles.profileImage}
        />
      </View>

      {/* User Info */}
      <View style={styles.userInfo}>
        <Text style={styles.name}>สมชาย ใจดี</Text>
        <Text style={styles.role}>React Native Developer</Text>
        <Text style={styles.bio}>
          ชอบเขียน code และสร้างแอป Mobile ที่สวยงาม
        </Text>
      </View>

      {/* Stats */}
      <View style={styles.statsContainer}>
        <View style={styles.stat}>
          <Text style={styles.statNumber}>{followers.toLocaleString()}</Text>
          <Text style={styles.statLabel}>Followers</Text>
        </View>
        <View style={styles.statDivider} />
        <View style={styles.stat}>
          <Text style={styles.statNumber}>892</Text>
          <Text style={styles.statLabel}>Following</Text>
        </View>
        <View style={styles.statDivider} />
        <View style={styles.stat}>
          <Text style={styles.statNumber}>48</Text>
          <Text style={styles.statLabel}>Projects</Text>
        </View>
      </View>

      {/* Action Buttons */}
      <View style={styles.buttonContainer}>
        <TouchableOpacity
          style={[
            styles.followButton,
            isFollowing && styles.followingButton,
          ]}
          onPress={handleFollow}
        >
          <Text
            style={[
              styles.followButtonText,
              isFollowing && styles.followingButtonText,
            ]}
          >
            {isFollowing ? 'กำลังติดตาม' : 'ติดตาม'}
          </Text>
        </TouchableOpacity>

        <TouchableOpacity style={styles.messageButton}>
          <Text style={styles.messageButtonText}>ส่งข้อความ</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

// Main App
const App = () => {
  return (
    <SafeAreaView style={styles.safeArea}>
      <StatusBar barStyle="dark-content" backgroundColor="#f5f5f5" />
      <View style={styles.container}>
        <Text style={styles.header}>โปรไฟล์</Text>
        <ProfileCard />
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  safeArea: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  container: {
    flex: 1,
    padding: 16,
  },
  header: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 16,
  },
  card: {
    backgroundColor: '#fff',
    borderRadius: 16,
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 8,
    elevation: 4,
  },
  coverImage: {
    height: 120,
    backgroundColor: '#007AFF',
  },
  profileImageContainer: {
    alignItems: 'center',
    marginTop: -40,
  },
  profileImage: {
    width: 80,
    height: 80,
    borderRadius: 40,
    borderWidth: 3,
    borderColor: '#fff',
  },
  userInfo: {
    alignItems: 'center',
    padding: 16,
    paddingTop: 8,
  },
  name: {
    fontSize: 20,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 4,
  },
  role: {
    fontSize: 14,
    color: '#007AFF',
    marginBottom: 8,
  },
  bio: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center',
    lineHeight: 20,
  },
  statsContainer: {
    flexDirection: 'row',
    justifyContent: 'space-around',
    paddingVertical: 16,
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    marginHorizontal: 16,
  },
  stat: {
    alignItems: 'center',
  },
  statNumber: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#333',
  },
  statLabel: {
    fontSize: 12,
    color: '#999',
    marginTop: 2,
  },
  statDivider: {
    width: 1,
    height: '100%',
    backgroundColor: '#f0f0f0',
  },
  buttonContainer: {
    flexDirection: 'row',
    padding: 16,
    gap: 12,
  },
  followButton: {
    flex: 1,
    backgroundColor: '#007AFF',
    paddingVertical: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  followingButton: {
    backgroundColor: '#fff',
    borderWidth: 1,
    borderColor: '#007AFF',
  },
  followButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  followingButtonText: {
    color: '#007AFF',
  },
  messageButton: {
    flex: 1,
    backgroundColor: '#fff',
    paddingVertical: 12,
    borderRadius: 8,
    alignItems: 'center',
    borderWidth: 1,
    borderColor: '#ddd',
  },
  messageButtonText: {
    color: '#333',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default App;
```

### Workshop 3.3: เพิ่ม Dark Mode Support

```tsx
import React from 'react';
import {
  View,
  Text,
  StyleSheet,
  SafeAreaView,
  useColorScheme,
  TouchableOpacity,
} from 'react-native';

// Color theme
const theme = {
  light: {
    background: '#ffffff',
    card: '#f9f9f9',
    text: '#333333',
    subtext: '#666666',
    border: '#e0e0e0',
    primary: '#007AFF',
  },
  dark: {
    background: '#1c1c1e',
    card: '#2c2c2e',
    text: '#ffffff',
    subtext: '#ebebf5cc',
    border: '#38383a',
    primary: '#0a84ff',
  },
};

const App = () => {
  const colorScheme = useColorScheme();
  const colors = theme[colorScheme === 'dark' ? 'dark' : 'light'];

  return (
    <SafeAreaView style={[styles.safeArea, { backgroundColor: colors.background }]}>
      <View style={[styles.container, { backgroundColor: colors.background }]}>
        <Text style={[styles.title, { color: colors.text }]}>
          Dark Mode Support
        </Text>
        
        <View style={[styles.card, { backgroundColor: colors.card, borderColor: colors.border }]}>
          <Text style={[styles.cardTitle, { color: colors.text }]}>
            หัวข้อการ์ด
          </Text>
          <Text style={[styles.cardContent, { color: colors.subtext }]}>
            เนื้อหาของการ์ดที่รองรับ Dark Mode
          </Text>
          <TouchableOpacity
            style={[styles.button, { backgroundColor: colors.primary }]}
          >
            <Text style={styles.buttonText}>กดปุ่ม</Text>
          </TouchableOpacity>
        </View>

        <Text style={[styles.modeText, { color: colors.subtext }]}>
          โหมดปัจจุบัน: {colorScheme === 'dark' ? '🌙 Dark' : '☀️ Light'}
        </Text>
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  safeArea: {
    flex: 1,
  },
  container: {
    flex: 1,
    padding: 20,
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    marginBottom: 20,
  },
  card: {
    borderRadius: 12,
    padding: 16,
    borderWidth: 1,
    marginBottom: 16,
  },
  cardTitle: {
    fontSize: 18,
    fontWeight: '600',
    marginBottom: 8,
  },
  cardContent: {
    fontSize: 14,
    lineHeight: 20,
    marginBottom: 16,
  },
  button: {
    paddingVertical: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  modeText: {
    textAlign: 'center',
    fontSize: 14,
  },
});

export default App;
```

### Workshop 3.4: แบบฝึกหัด

**แบบฝึกหัด 1:** แก้ไข Hello World App ให้:
- แสดงชื่อของคุณ
- เพิ่ม emoji ที่เหมาะสม
- เปลี่ยนสีพื้นหลังเป็นสีที่คุณชอบ
- เพิ่มข้อความ quote ที่คุณชื่นชอบ

**แบบฝึกหัด 2:** สร้าง "Name Card" component:
- แสดงชื่อและนามสกุล
- แสดง title/position
- แสดง email
- แสดง phone
- Style ให้สวยงาม

**แบบฝึกหัด 3:** สังเกต Fast Refresh:
1. รัน app แล้วนับ counter จนถึง 5
2. เปลี่ยนสีปุ่ม
3. สังเกตว่า counter ยังอยู่ที่ 5 หรือไม่
4. แก้ไข state ใน useState
5. สังเกตว่า counter reset เป็น 0

---

## Tips และ Best Practices

### 1. ใช้ TypeScript ตั้งแต่ต้น

```tsx
// ✅ ดี - มี type safety
type Props = {
  name: string;
  age: number;
  onPress: () => void;
};

const UserCard = ({ name, age, onPress }: Props) => {
  // ...
};

// ❌ ไม่แนะนำ - ไม่มี type
const UserCard = ({ name, age, onPress }) => {
  // ...
};
```

### 2. ใช้ StyleSheet.create() แทน Inline Styles

```tsx
// ✅ ดี - Performance ดีกว่า, อ่านง่ายกว่า
const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
  },
});

// ❌ ไม่แนะนำ - สร้าง object ใหม่ทุก render
const MyComponent = () => (
  <View style={{ flex: 1, padding: 16 }}>
    ...
  </View>
);
```

### 3. แยก Component เมื่อใหญ่เกินไป

```
กฎทั่วไป:
- Component เกิน 100 บรรทัด → พิจารณาแยก
- Logic ซับซ้อน → แยกเป็น Custom Hook
- UI ซ้ำกัน → สร้าง Reusable Component
```

### 4. ตั้งชื่อให้ชัดเจน

```tsx
// ✅ ดี - ชื่อบ่งบอกหน้าที่ชัดเจน
const handleLoginPress = () => {};
const isLoadingUserData = true;
const userProfileData = {};

// ❌ ไม่ดี - ชื่อไม่บ่งบอกหน้าที่
const handle = () => {};
const loading = true;
const data = {};
```

---

## สรุป Part 003

### สิ่งที่ได้เรียนรู้

1. **สร้าง Project** - ด้วย `npx @react-native-community/cli@latest init`
2. **โครงสร้างไฟล์** - index.js, App.tsx, android/, ios/
3. **รัน App** - Metro Bundler + run-android/run-ios
4. **Dev Menu** - เปิดด้วย Ctrl+M (Android) หรือ Ctrl+D (iOS)
5. **Fast Refresh** - อัปเดต UI อัตโนมัติโดยคง state
6. **Debug** - console.log, React Native Debugger
7. **TypeScript** - ใช้ types สำหรับ safety

### คำสั่งสำคัญ

```bash
# สร้าง project
npx @react-native-community/cli@latest init ProjectName

# รัน
npx react-native start          # Metro
npx react-native run-android    # Android
npx react-native run-ios        # iOS

# Debug
npx react-native log-android    # Android logs
npx react-native log-ios        # iOS logs
```

---

**ต่อไป → Part 004: ทำความเข้าใจโครงสร้าง Project**
