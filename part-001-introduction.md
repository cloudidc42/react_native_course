# Part 001: แนะนำ React Native และ Mobile Development

## สารบัญ
1. [ประวัติและที่มาของ React Native](#ประวัติและที่มา)
2. [React Native คืออะไร](#react-native-คืออะไร)
3. [React Native vs Native Development](#react-native-vs-native)
4. [React Native vs Hybrid Framework](#react-native-vs-hybrid)
5. [React Native vs Flutter](#react-native-vs-flutter)
6. [สถาปัตยกรรมของ React Native](#สถาปัตยกรรม)
7. [New Architecture (JSI)](#new-architecture)
8. [ตลาดงานและอาชีพ](#ตลาดงาน)
9. [บริษัทที่ใช้ React Native](#บริษัทที่ใช้)
10. [Workshop: ดูตัวอย่างแอป](#workshop)

---

## ประวัติและที่มาของ React Native

### จุดเริ่มต้น

React Native ถูกสร้างขึ้นโดย **Meta (Facebook)** ในปี 2013 จากโปรเจกต์ภายในที่เรียกว่า **"Hackathon"** โดยทีมวิศวกรของ Facebook ต้องการหาวิธีที่จะพัฒนา Mobile Application โดยใช้ทักษะ Web Development ที่มีอยู่แล้ว

**ไทม์ไลน์สำคัญ:**

| ปี | เหตุการณ์ |
|-----|-----------|
| 2013 | Jordan Walke สร้าง React.js สำหรับ Web |
| 2013 | Facebook เริ่มทดลอง React สำหรับ Mobile |
| 2015 | React Native เปิดตัวสู่สาธารณะที่ React Conf |
| 2015 | รองรับ iOS เป็นครั้งแรก |
| 2015 | เพิ่มการรองรับ Android |
| 2018 | Microsoft เพิ่มการรองรับ Windows และ MacOS |
| 2019 | เริ่มพัฒนา New Architecture (JSI) |
| 2022 | React Native 0.70 พร้อม New Architecture เป็น default |
| 2024 | React Native 0.74+ ปรับปรุงประสิทธิภาพ |

### แรงบันดาลใจ

Jordan Walke นักพัฒนาของ Facebook กล่าวว่า:

> "เราต้องการให้นักพัฒนา Web สามารถสร้างแอปพลิเคชัน Mobile ที่มีประสิทธิภาพระดับ Native โดยไม่ต้องเรียนรู้ภาษาใหม่ทั้งหมด"

ก่อนหน้านี้ Facebook ใช้ WebView (HTML5) สำหรับแอปบางส่วน แต่พบปัญหาด้านประสิทธิภาพอย่างมาก จนถึงขนาดที่ Mark Zuckerberg ต้องออกมาขอโทษผู้ใช้งานในปี 2012

---

## React Native คืออะไร

React Native เป็น **Open-Source Framework** สำหรับพัฒนา Mobile Application โดยใช้ JavaScript และ React

### หลักการสำคัญ

**"Learn once, write anywhere"** ไม่ใช่ "Write once, run anywhere"

ความหมายคือ: เมื่อคุณเรียนรู้ React Native แล้ว คุณสามารถสร้างแอปสำหรับทั้ง iOS และ Android ได้ แต่อาจต้องปรับโค้ดบางส่วนให้เหมาะกับแต่ละ Platform

### วิธีการทำงานพื้นฐาน

```
JavaScript Code
      ↓
  React Native
      ↓
   Bridge (เก่า) / JSI (ใหม่)
      ↓
Native Components (iOS/Android)
```

เมื่อคุณเขียน `<View>` ใน React Native มันจะถูกแปลงเป็น:
- `UIView` บน iOS
- `android.view.View` บน Android

---

## React Native vs Native Development

### Native Development คืออะไร

**iOS Native:**
- ภาษา: Swift หรือ Objective-C
- IDE: Xcode
- เฉพาะสำหรับ Apple devices

**Android Native:**
- ภาษา: Kotlin หรือ Java
- IDE: Android Studio
- เฉพาะสำหรับ Android devices

### เปรียบเทียบ

| หัวข้อ | React Native | iOS Native | Android Native |
|--------|-------------|------------|----------------|
| ภาษา | JavaScript/TypeScript | Swift | Kotlin |
| ทีมพัฒนา | 1 ทีม | 1 ทีม iOS | 1 ทีม Android |
| ค่าใช้จ่าย | ต่ำกว่า | สูง | สูง |
| ประสิทธิภาพ | ดีมาก (ใกล้เคียง Native) | ดีที่สุด | ดีที่สุด |
| เวลาพัฒนา | เร็วกว่า | ช้ากว่า | ช้ากว่า |
| Access ฟีเจอร์ใหม่ | ล่าช้าเล็กน้อย | ทันที | ทันที |
| Community | ใหญ่มาก | ใหญ่ | ใหญ่ |

### ตัวอย่างโค้ด: Hello World

**React Native:**
```jsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const App = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.text}>Hello, World!</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  text: {
    fontSize: 24,
    fontWeight: 'bold',
  },
});

export default App;
```

**Swift (iOS):**
```swift
import UIKit

class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let label = UILabel()
        label.text = "Hello, World!"
        label.font = UIFont.boldSystemFont(ofSize: 24)
        label.translatesAutoresizingMaskIntoConstraints = false
        
        view.addSubview(label)
        
        NSLayoutConstraint.activate([
            label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            label.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }
}
```

**Kotlin (Android):**
```kotlin
import android.os.Bundle
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        val textView = TextView(this)
        textView.text = "Hello, World!"
        textView.textSize = 24f
        setContentView(textView)
    }
}
```

### เมื่อไหร่ควรใช้ Native Development?

- แอปที่ต้องการประสิทธิภาพสูงสุด (เกม 3D, AR/VR)
- ต้องใช้ฟีเจอร์ใหม่ล่าสุดของ Platform ทันที
- ทีมมีความเชี่ยวชาญใน Native development อยู่แล้ว
- แอปที่ Logic ซับซ้อนมากและแตกต่างกันระหว่าง iOS และ Android

---

## React Native vs Hybrid Framework

### Hybrid Framework คืออะไร

Hybrid Apps คือการฝัง WebView ลงใน Native Shell และแสดง HTML/CSS/JavaScript

### Cordova/PhoneGap

```
Web App (HTML/CSS/JS)
         ↓
    WebView
         ↓
Native Shell (ห่อหุ้มอีกที)
```

**ข้อเสีย Cordova:**
- ประสิทธิภาพต่ำกว่า (ทำงานใน WebView)
- UI ไม่เหมือน Native จริงๆ
- ช้าในการ render
- ปัญหาการเข้าถึง Device APIs

### Ionic

Ionic ใช้ Angular/React/Vue + Apache Cordova หรือ Capacitor

**ข้อดี Ionic:**
- ใช้ Web Technologies ล้วนๆ
- นักพัฒนา Web เรียนรู้ได้เร็ว

**ข้อเสีย Ionic:**
- ยังคงเป็น WebView-based
- ประสิทธิภาพด้อยกว่า React Native

### ตาราง เปรียบเทียบ

| หัวข้อ | React Native | Cordova | Ionic |
|--------|-------------|---------|-------|
| Rendering | Native Components | WebView | WebView |
| ประสิทธิภาพ | สูง | ต่ำ | ปานกลาง |
| Look & Feel | Native | Web-like | Web-like |
| JavaScript | React | Any | Angular/React/Vue |
| Community | ใหญ่มาก | เล็กลง | กลาง |
| อนาคต | สดใส | ไม่แน่นอน | ปานกลาง |

---

## React Native vs Flutter

### Flutter คืออะไร

Flutter เป็น UI Toolkit ของ Google ที่ใช้ภาษา Dart และ render UI ด้วย Skia/Impeller Engine แทน Native Components

```
Flutter Architecture:
Dart Code
    ↓
Flutter Engine (C++)
    ↓
Skia/Impeller (Custom Rendering)
    ↓
Platform Canvas (ไม่ใช้ Native Components)
```

### เปรียบเทียบ React Native vs Flutter

| หัวข้อ | React Native | Flutter |
|--------|-------------|---------|
| ภาษา | JavaScript/TypeScript | Dart |
| UI Rendering | Native Components | Custom Canvas |
| ประสิทธิภาพ | ดีมาก | ดีมาก (บางครั้งดีกว่า) |
| Look | เหมือน Native จริงๆ | Consistent แต่ไม่ใช่ Native |
| Package Ecosystem | npm (ใหญ่มาก) | pub.dev (เล็กกว่า) |
| ทีมหลัก | Meta | Google |
| เรียนรู้ | ง่าย (ถ้ารู้ JS) | ง่าย (Dart เรียนไม่นาน) |
| Hot Reload | มี | มี |
| Web Support | มี (React Native Web) | มี (ดีกว่า) |
| Desktop | มี (Windows/Mac) | มี (ครบกว่า) |
| Job Market | มากกว่า | กำลังเติบโต |

### Code เปรียบเทียบ: Simple Card Component

**React Native:**
```jsx
import React from 'react';
import { View, Text, Image, StyleSheet, TouchableOpacity } from 'react-native';

const ProductCard = ({ title, price, imageUrl, onPress }) => {
  return (
    <TouchableOpacity style={styles.card} onPress={onPress}>
      <Image
        source={{ uri: imageUrl }}
        style={styles.image}
      />
      <View style={styles.info}>
        <Text style={styles.title}>{title}</Text>
        <Text style={styles.price}>฿{price}</Text>
      </View>
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 8,
    elevation: 3,
    marginBottom: 16,
  },
  image: {
    width: '100%',
    height: 200,
    borderTopLeftRadius: 12,
    borderTopRightRadius: 12,
  },
  info: {
    padding: 16,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  price: {
    fontSize: 16,
    color: '#007AFF',
    fontWeight: '600',
  },
});

export default ProductCard;
```

**Flutter (Dart):**
```dart
import 'package:flutter/material.dart';

class ProductCard extends StatelessWidget {
  final String title;
  final double price;
  final String imageUrl;
  final VoidCallback onPress;

  const ProductCard({
    Key? key,
    required this.title,
    required this.price,
    required this.imageUrl,
    required this.onPress,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onPress,
      child: Card(
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
        ),
        elevation: 3,
        margin: const EdgeInsets.only(bottom: 16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            ClipRRect(
              borderRadius: const BorderRadius.vertical(
                top: Radius.circular(12),
              ),
              child: Image.network(
                imageUrl,
                width: double.infinity,
                height: 200,
                fit: BoxFit.cover,
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    title,
                    style: const TextStyle(
                      fontSize: 18,
                      fontWeight: FontWeight.bold,
                      color: Color(0xFF333333),
                    ),
                  ),
                  const SizedBox(height: 8),
                  Text(
                    '฿${price.toStringAsFixed(0)}',
                    style: const TextStyle(
                      fontSize: 16,
                      color: Color(0xFF007AFF),
                      fontWeight: FontWeight.w600,
                    ),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### เมื่อไหร่ควรเลือก React Native?

✅ ทีมมีทักษะ JavaScript/TypeScript อยู่แล้ว
✅ ต้องการใช้ npm ecosystem ที่ใหญ่มาก
✅ ต้องการให้แอปดูเหมือน Native จริงๆ
✅ ทีม Web Developer ที่อยากทำ Mobile

### เมื่อไหร่ควรเลือก Flutter?

✅ ต้องการ UI ที่ consistent 100% ระหว่าง platforms
✅ ทีมไม่มี JavaScript background
✅ ต้องการ Performance สูงสุด
✅ สนใจพัฒนา Web + Desktop ด้วย

---

## สถาปัตยกรรมของ React Native

### Old Architecture (Bridge-based)

สถาปัตยกรรมเดิมใช้ **JavaScript Bridge** เป็นตัวกลางในการสื่อสาร

```
┌─────────────────────────────────────────────┐
│              JavaScript Thread              │
│   React Components → Virtual DOM → Diff    │
└─────────────────┬───────────────────────────┘
                  │ Async Bridge (JSON serialization)
                  │ (ช้า + bottleneck)
┌─────────────────▼───────────────────────────┐
│              Native Thread                  │
│   Native Modules → UI Components → Screen  │
└─────────────────────────────────────────────┘
```

**ปัญหาของ Bridge:**
1. **Async by default** - การสื่อสารเป็น async เสมอ แม้แต่งานที่ต้องการ sync
2. **JSON Serialization** - ข้อมูลทุกอย่างต้อง serialize เป็น JSON ก่อนส่ง
3. **Single threaded** - Bridge มีเพียง 1 เส้นทาง
4. **Memory overhead** - ต้องแปลงข้อมูลไปมาระหว่าง JS และ Native

### Threads ใน React Native (Old)

React Native มี 3 Threads หลัก:

```
1. JavaScript Thread (Main Logic)
   - รัน JavaScript code
   - React reconciliation
   - Business logic

2. Native/UI Thread (Main Thread)
   - จัดการ UI
   - Handle gestures
   - Native modules

3. Shadow Thread (Layout)
   - คำนวณ layout (Yoga/Flexbox)
   - ส่งผลลัพธ์ไปยัง UI Thread
```

### New Architecture (JSI - JavaScript Interface)

React Native เริ่มพัฒนา New Architecture ตั้งแต่ปี 2019 และ stable ใน 0.71+

```
┌─────────────────────────────────────────────┐
│              JavaScript Thread              │
│         JSI (Direct Binding)               │
└─────────────────┬───────────────────────────┘
                  │ Synchronous (ไม่ต้อง serialize!)
                  │
┌─────────────────▼───────────────────────────┐
│           C++ Host Objects                  │
│     (Shared Memory, Direct Access)          │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│           Native Modules/UI                 │
└─────────────────────────────────────────────┘
```

### ส่วนประกอบของ New Architecture

**1. JSI (JavaScript Interface)**
- แทนที่ Bridge ด้วย C++ Host Objects
- JavaScript สามารถ call Native methods แบบ synchronous
- ไม่ต้อง serialize เป็น JSON
- ประสิทธิภาพดีขึ้นมาก

```javascript
// JSI ช่วยให้ทำแบบนี้ได้:
// JavaScript สามารถ access Native Object โดยตรง
const nativeModule = global.nativeModuleRef;
const result = nativeModule.computeSync(data); // Synchronous!
```

**2. Fabric (New Renderer)**
- แทนที่ UIManager เดิม
- Concurrent rendering (เหมือน React 18)
- ลด layout overhead
- รองรับ Suspense

```
Old: JS → Bridge → Shadow Thread → UI Thread
New: JS → Fabric (C++) → Direct UI Updates
```

**3. TurboModules**
- แทนที่ Native Modules เดิม
- Lazy loading (โหลดเมื่อต้องการเท่านั้น)
- Type-safe interface
- ประสิทธิภาพดีขึ้น

```javascript
// TurboModules: Load on demand
import { TurboModuleRegistry } from 'react-native';

// โหลดเฉพาะตอนที่ใช้งาน
const CameraModule = TurboModuleRegistry.getEnforcing('Camera');
```

**4. Codegen**
- Auto-generate type-safe bindings
- ตรวจสอบ Type ตั้งแต่ build time
- ลด runtime errors

### การทำงานร่วมกัน

```
React Component
      │
      ▼
React Reconciler (JS)
      │
      ▼
Fabric Renderer (C++)
      │
      ├── Layout (Yoga)
      │
      ▼
Shadow Tree → Commit → Screen
```

---

## New Architecture

### Concurrent Features

New Architecture ช่วยให้ React Native รองรับ React 18 Concurrent Features:

```jsx
import React, { Suspense, startTransition, useState } from 'react';
import { View, Text, ActivityIndicator } from 'react-native';

// Lazy Component
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

const App = () => {
  const [showHeavy, setShowHeavy] = useState(false);

  const handleShow = () => {
    // startTransition ทำให้ UI responsive ระหว่าง load
    startTransition(() => {
      setShowHeavy(true);
    });
  };

  return (
    <View>
      <Suspense fallback={<ActivityIndicator />}>
        {showHeavy && <HeavyComponent />}
      </Suspense>
    </View>
  );
};
```

### Performance Improvements

```
Old Architecture Benchmarks:
- List scroll: 45-50fps (บาง frame drop)
- Animation: 55-60fps
- Cold start: ~2-3 seconds

New Architecture Benchmarks:
- List scroll: 60fps (สม่ำเสมอกว่า)
- Animation: 60fps
- Cold start: ~1-1.5 seconds
```

---

## ตลาดงานและอาชีพ

### ข้อมูลตลาดงาน 2024-2025

**ความต้องการ React Native Developer:**

ในประเทศไทย:
- บริษัท Startup: ต้องการมาก (1 คนทำได้ทั้ง iOS + Android)
- บริษัทขนาดกลาง: เติบโตต่อเนื่อง
- บริษัทขนาดใหญ่: มีทีม Mobile โดยเฉพาะ

เงินเดือน (ประมาณการ):
| ระดับ | ประสบการณ์ | เงินเดือน (บาท/เดือน) |
|-------|-----------|---------------------|
| Junior | 0-2 ปี | 25,000 - 45,000 |
| Mid-level | 2-5 ปี | 45,000 - 80,000 |
| Senior | 5+ ปี | 80,000 - 150,000+ |
| Tech Lead | 7+ ปี | 120,000 - 200,000+ |

**ทักษะที่นายจ้างต้องการ:**
1. React Native (หลัก)
2. TypeScript
3. Redux / Zustand / MobX
4. REST API / GraphQL
5. Git
6. Testing (Jest, Detox)
7. CI/CD (Fastlane, GitHub Actions)
8. Firebase
9. Native Bridge (Bonus)
10. App Store / Play Store deployment

### Roadmap สู่ React Native Developer

```
เริ่มต้น (0-3 เดือน):
├── HTML/CSS/JavaScript พื้นฐาน
├── ES6+ (Arrow functions, Destructuring, Async/Await)
└── React.js พื้นฐาน (JSX, Props, State, Hooks)

React Native เบื้องต้น (3-6 เดือน):
├── Setup Environment
├── Core Components
├── StyleSheet & Flexbox
├── Navigation (React Navigation)
└── Basic State Management

ระดับกลาง (6-12 เดือน):
├── Advanced State Management (Redux/Zustand)
├── API Integration
├── Firebase
├── AsyncStorage / MMKV
├── Native Modules basics
└── Performance Optimization

ระดับสูง (1-2 ปี):
├── Native Bridge Development
├── Custom Animations (Reanimated)
├── Testing (Jest, Detox)
├── CI/CD Pipeline
├── App Store Deployment
└── Performance Profiling
```

---

## บริษัทที่ใช้ React Native

### บริษัทระดับโลก

**Facebook/Meta**
- Facebook app บางส่วน
- Facebook Ads Manager

**Microsoft**
- Skype
- Xbox Game Pass
- Microsoft Teams (mobile)

**Shopify**
- Shopify POS

**Discord**
- Discord Mobile App

**Wix**
- Wix Mobile

**Bloomberg**
- Bloomberg Professional

**Tesla**
- Tesla Mobile App (บางส่วน)

**Walmart**
- Walmart App

### บริษัทไทยที่ใช้ React Native

- บริษัท Fintech หลายแห่ง
- E-commerce platforms
- Startup ด้าน Food Delivery
- บริษัท EdTech
- Healthcare Apps

### Case Study: Shopify

Shopify ย้ายจาก Native (iOS + Android แยกกัน) มาใช้ React Native และประหยัดเวลาพัฒนาได้ 30-40% โดยไม่สูญเสียประสิทธิภาพ

---

## Workshop: ดูตัวอย่างแอปที่สร้างด้วย React Native

### Workshop 1.1: ติดตั้งและรัน React Native Showcase

ดาวน์โหลดแอปตัวอย่างและดูว่า React Native ทำอะไรได้บ้าง

**ขั้นตอน:**

1. เปิด App Store หรือ Play Store
2. ค้นหาแอปเหล่านี้และสังเกตประสบการณ์ผู้ใช้:
   - Facebook
   - Discord
   - Shopify POS

3. สังเกตสิ่งต่อไปนี้:
   - ความลื่นไหลของ Animation
   - ความเร็วในการตอบสนอง
   - Look & Feel เทียบกับ Native App อื่นๆ

### Workshop 1.2: สำรวจ React Native Directory

เปิดเว็บไซต์ https://reactnative.directory และสำรวจ:

1. Library ที่ได้รับความนิยมสูงสุด
2. Library สำหรับ Camera, Maps, Auth
3. ดูจำนวน Stars บน GitHub

### Workshop 1.3: อ่าน React Native Blog

เปิด https://reactnative.dev/blog และอ่าน:
- Blog post ล่าสุดเกี่ยวกับ New Architecture
- Release notes ของ version ล่าสุด

### Workshop 1.4: เขียน Component แรก (Preview)

แม้ยังไม่ได้ติดตั้ง Environment แต่ลองเขียนดูก่อน:

```jsx
// ลองคิดดูว่าโค้ดนี้จะแสดงผลอะไร
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const MyProfile = () => {
  const name = "สมชาย ใจดี";
  const role = "React Native Developer";
  const skills = ["JavaScript", "React", "TypeScript"];

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.name}>{name}</Text>
        <Text style={styles.role}>{role}</Text>
      </View>

      {/* Skills */}
      <View style={styles.skillsContainer}>
        <Text style={styles.sectionTitle}>ทักษะ:</Text>
        {skills.map((skill, index) => (
          <View key={index} style={styles.skillBadge}>
            <Text style={styles.skillText}>{skill}</Text>
          </View>
        ))}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    backgroundColor: '#f5f5f5',
  },
  header: {
    backgroundColor: '#007AFF',
    padding: 20,
    borderRadius: 12,
    marginBottom: 20,
  },
  name: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#fff',
  },
  role: {
    fontSize: 16,
    color: '#rgba(255,255,255,0.8)',
    marginTop: 4,
  },
  skillsContainer: {
    backgroundColor: '#fff',
    padding: 16,
    borderRadius: 12,
  },
  sectionTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 12,
    color: '#333',
  },
  skillBadge: {
    backgroundColor: '#E3F2FD',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 20,
    marginBottom: 8,
    alignSelf: 'flex-start',
  },
  skillText: {
    color: '#1565C0',
    fontWeight: '600',
  },
});

export default MyProfile;
```

**คำถามสำหรับคิด:**
1. `flex: 1` หมายความว่าอะไร?
2. ทำไมต้องใช้ `StyleSheet.create()` แทน inline style?
3. `map()` ทำงานอย่างไรใน JSX?

---

## สรุป Part 001

### สิ่งที่ได้เรียนรู้

1. **ประวัติ React Native** - สร้างโดย Meta ในปี 2015 จาก Hackathon
2. **หลักการ** - "Learn once, write anywhere" โดยใช้ JavaScript
3. **เปรียบเทียบ** - React Native มีจุดเด่นด้านความเร็วในการพัฒนาและ code reuse
4. **สถาปัตยกรรม** - Bridge (เก่า) → JSI/Fabric (ใหม่)
5. **ตลาดงาน** - ความต้องการสูง เงินเดือนดี

### ข้อดีของ React Native

- ✅ Code sharing ระหว่าง iOS และ Android
- ✅ ชุมชนใหญ่และ ecosystem ครบ
- ✅ Hot Reload เพิ่มความเร็วในการพัฒนา
- ✅ Native Performance (ไม่ใช่ WebView)
- ✅ นักพัฒนา Web สามารถเปลี่ยนมาทำ Mobile ได้

### ข้อจำกัด

- ⚠️ ประสิทธิภาพต่ำกว่า Pure Native เล็กน้อยในบางกรณี
- ⚠️ ฟีเจอร์ใหม่ของ Platform อาจช้ากว่า
- ⚠️ Debugging อาจซับซ้อนกว่า Web
- ⚠️ ขนาดแอปใหญ่กว่า Native เล็กน้อย

---

## แหล่งเรียนรู้เพิ่มเติม

- [React Native Official Docs](https://reactnative.dev)
- [React Native Blog](https://reactnative.dev/blog)
- [React Native Directory](https://reactnative.directory)
- [GitHub: facebook/react-native](https://github.com/facebook/react-native)
- [React Native Community](https://github.com/react-native-community)
- YouTube: [React Native School](https://www.youtube.com/c/ReactNativeSchool)

---

**ต่อไป → Part 002: การติดตั้งและตั้งค่า Development Environment**
