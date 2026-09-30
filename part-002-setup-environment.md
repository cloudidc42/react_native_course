# Part 002: การติดตั้งและตั้งค่า Development Environment

## สารบัญ
1. [ข้อกำหนดระบบ](#ข้อกำหนดระบบ)
2. [ติดตั้ง Node.js และ npm/yarn](#ติดตั้ง-nodejs)
3. [ติดตั้ง React Native CLI](#ติดตั้ง-react-native-cli)
4. [ติดตั้ง Expo CLI](#ติดตั้ง-expo-cli)
5. [Setup Android Studio](#setup-android-studio)
6. [Setup Xcode (Mac)](#setup-xcode)
7. [Setup Emulator/Simulator](#setup-emulator)
8. [Setup VS Code + Extensions](#setup-vs-code)
9. [ตรวจสอบ Environment](#ตรวจสอบ-environment)
10. [Workshop](#workshop)
11. [แก้ปัญหาที่พบบ่อย](#แก้ปัญหา)

---

## ข้อกำหนดระบบ

### สำหรับ Windows

| Component | ขั้นต่ำ | แนะนำ |
|-----------|---------|-------|
| OS | Windows 10 | Windows 10/11 64-bit |
| RAM | 8 GB | 16 GB+ |
| CPU | Core i5 | Core i7 หรือ AMD Ryzen 7 |
| Storage | 20 GB | 50 GB+ SSD |
| Virtualization | ต้องเปิดใช้ | BIOS VT-x/AMD-V |

### สำหรับ macOS

| Component | ขั้นต่ำ | แนะนำ |
|-----------|---------|-------|
| OS | macOS 12 | macOS 13/14 |
| RAM | 8 GB | 16 GB+ |
| CPU | Apple M1 / Intel i5 | Apple M2/M3 |
| Storage | 30 GB | 60 GB+ |
| Xcode | 14+ | ล่าสุด |

### สำหรับ Linux

| Component | ขั้นต่ำ | แนะนำ |
|-----------|---------|-------|
| Distro | Ubuntu 20.04 | Ubuntu 22.04+ |
| RAM | 8 GB | 16 GB+ |
| Storage | 20 GB | 50 GB+ |
| Note | iOS ไม่รองรับ | Android เท่านั้น |

> ⚠️ **สำคัญ**: หากต้องการพัฒนาสำหรับ iOS จำเป็นต้องใช้ macOS เท่านั้น

---

## ติดตั้ง Node.js

### วิธีที่ 1: ดาวน์โหลดจาก nodejs.org (ง่ายที่สุด)

1. ไปที่ https://nodejs.org
2. ดาวน์โหลด **LTS version** (แนะนำ)
3. รัน Installer
4. ตรวจสอบการติดตั้ง:

```bash
node --version
# ควรเห็น: v20.x.x หรือ v22.x.x

npm --version
# ควรเห็น: 10.x.x
```

### วิธีที่ 2: ใช้ NVM (Node Version Manager) - แนะนำสำหรับ Developer

NVM ช่วยให้จัดการหลาย Node versions ได้

**macOS/Linux:**

```bash
# ติดตั้ง NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# โหลด NVM (เพิ่มใน ~/.bashrc หรือ ~/.zshrc)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# รีสตาร์ท Terminal แล้วติดตั้ง Node
nvm install 20
nvm use 20
nvm alias default 20

# ตรวจสอบ
node --version  # v20.x.x
nvm current     # v20.x.x
```

**Windows (ใช้ nvm-windows):**

```powershell
# ดาวน์โหลดจาก https://github.com/coreybutler/nvm-windows/releases
# รัน nvm-setup.exe

# หลังติดตั้ง เปิด PowerShell ใหม่ (Admin)
nvm install 20.0.0
nvm use 20.0.0

# ตรวจสอบ
node --version
```

### ติดตั้ง Yarn (ทางเลือก)

```bash
# ติดตั้ง Yarn v1 (Classic) - แนะนำสำหรับ React Native
npm install -g yarn

# หรือ Yarn v3+ (Berry)
npm install -g yarn@stable

# ตรวจสอบ
yarn --version
```

### npm vs yarn - ควรใช้อะไร?

| | npm | yarn |
|--|-----|------|
| ความเร็ว | ปานกลาง | เร็วกว่า (caching) |
| Lock file | package-lock.json | yarn.lock |
| workspace | มี | มี (ดีกว่า) |
| ความนิยม | มากกว่า | น้อยกว่าเล็กน้อย |
| React Native | ✅ | ✅ |

> 💡 **คำแนะนำ**: ใช้ npm ในคอร์สนี้เพื่อความง่าย แต่ yarn ก็ดีไม่แพ้กัน

---

## ติดตั้ง React Native CLI

### วิธีที่ 1: React Native Community CLI (แนะนำ)

```bash
# ไม่จำเป็นต้องติดตั้ง global แล้ว - ใช้ npx
npx @react-native-community/cli@latest init MyFirstApp

# หรือถ้าต้องการติดตั้ง global
npm install -g @react-native-community/cli
react-native init MyFirstApp
```

### ข้อดีของ React Native CLI

- ✅ Full control over native code
- ✅ ใส่ native modules ได้ทุกอย่าง
- ✅ ไม่มี abstraction layer
- ✅ เหมาะสำหรับ production apps

### ข้อเสียของ React Native CLI

- ❌ Setup ยากกว่า Expo
- ❌ ต้องการ Xcode + Android Studio
- ❌ Build time นานกว่า

---

## ติดตั้ง Expo CLI

### Expo คืออะไร?

Expo คือ Platform และ Framework ที่ช่วยให้ React Native พัฒนาได้ง่ายขึ้น โดยไม่ต้องยุ่งกับ Native Code

```
Expo Workflow Options:

1. Managed Workflow (ง่ายที่สุด)
   - Expo จัดการ Native code ให้ทั้งหมด
   - ไม่ต้องติดตั้ง Xcode/Android Studio
   - จำกัด Native modules บางอย่าง

2. Bare Workflow (ยืดหยุ่นกว่า)
   - มี Native code ให้ปรับแต่ง
   - ใช้ Expo modules ได้
   - ต้องการ Xcode/Android Studio

3. Expo Go (สำหรับทดสอบ)
   - แอปสำหรับรัน Expo projects
   - ไม่ต้อง build native app
   - เหมาะสำหรับเรียนรู้
```

### ติดตั้ง Expo

```bash
# ติดตั้ง Expo CLI
npm install -g expo-cli

# หรือใช้ npx (ไม่ต้อง global install)
npx create-expo-app MyExpoApp

# สร้าง project
expo init MyExpoApp
# หรือ
npx create-expo-app MyExpoApp --template

# รัน project
cd MyExpoApp
npx expo start
```

### เปรียบเทียบ CLI vs Expo

| | React Native CLI | Expo (Managed) |
|--|----------------|----------------|
| Setup | ยาก | ง่ายมาก |
| Native Modules | ทุกอย่าง | จำกัด |
| Build | ต้อง local | Cloud (EAS Build) |
| เหมาะสำหรับ | Production | Prototype/Learning |
| iOS build | ต้องมี Mac | ไม่จำเป็น |
| OTA Updates | ต้อง setup | มีใน Expo |

---

## Setup Android Studio

### ขั้นตอนการติดตั้ง Android Studio

**ขั้นตอนที่ 1: ดาวน์โหลด**

ไปที่ https://developer.android.com/studio และดาวน์โหลด

**ขั้นตอนที่ 2: ติดตั้ง**

```
Windows:
1. รัน .exe installer
2. เลือก Components:
   ✅ Android Studio
   ✅ Android Virtual Device
3. เลือก Installation Path (default แนะนำ)
4. ทำตามขั้นตอน

macOS:
1. เปิด .dmg file
2. ลาก Android Studio ไปยัง Applications
3. เปิด Android Studio
4. ทำตาม Setup Wizard
```

**ขั้นตอนที่ 3: SDK Setup**

เมื่อเปิด Android Studio ครั้งแรก:

```
1. SDK Components Setup:
   ✅ Android SDK
   ✅ Android SDK Platform
   ✅ Android Virtual Device
   ✅ Performance (Intel HAXM) - Intel CPU เท่านั้น

2. SDK Platforms:
   - เปิด SDK Manager (Tools > SDK Manager)
   - เลือก Android 14 (API 34) หรือสูงกว่า
   - ✅ Android SDK Platform 34
   - ✅ Intel x86 Atom_64 System Image
   - ✅ Google APIs Intel x86 Atom System Image
```

### ตั้งค่า Environment Variables

**macOS/Linux (เพิ่มใน ~/.bashrc หรือ ~/.zshrc):**

```bash
# Android SDK
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin

# Java (หากจำเป็น)
export JAVA_HOME=/Applications/Android\ Studio.app/Contents/jbr/Contents/Home
```

**Windows (Environment Variables):**

```powershell
# เปิด System Properties > Environment Variables
# เพิ่ม ANDROID_HOME:
C:\Users\<username>\AppData\Local\Android\Sdk

# เพิ่มใน PATH:
%ANDROID_HOME%\platform-tools
%ANDROID_HOME%\emulator
%ANDROID_HOME%\tools
%ANDROID_HOME%\tools\bin
```

**ตรวจสอบ:**

```bash
# ตรวจสอบ adb (Android Debug Bridge)
adb version
# Android Debug Bridge version 1.0.41

# ตรวจสอบ emulator
emulator -list-avds
```

### Java Development Kit (JDK)

React Native ต้องการ JDK 17

```bash
# macOS - ใช้ Homebrew
brew install --cask temurin@17

# Ubuntu/Debian
sudo apt-get install openjdk-17-jdk

# ตรวจสอบ
java --version
# openjdk 17.x.x

javac --version
# javac 17.x.x
```

---

## Setup Xcode

> ⚠️ **เฉพาะ macOS เท่านั้น**

### ติดตั้ง Xcode

**ขั้นตอนที่ 1:**
1. เปิด App Store
2. ค้นหา "Xcode"
3. กด Install (ขนาด ~12GB)
4. รอการดาวน์โหลด

**ขั้นตอนที่ 2: ติดตั้ง Command Line Tools**

```bash
xcode-select --install

# ตรวจสอบ
xcode-select -p
# /Applications/Xcode.app/Contents/Developer
```

**ขั้นตอนที่ 3: Accept License Agreement**

```bash
sudo xcodebuild -license accept
```

**ขั้นตอนที่ 4: ติดตั้ง CocoaPods**

```bash
# ติดตั้ง CocoaPods (Ruby gem)
sudo gem install cocoapods

# หรือใช้ Homebrew (แนะนำ)
brew install cocoapods

# ตรวจสอบ
pod --version
# 1.x.x
```

### ตั้งค่า iOS Simulator

```bash
# เปิด Simulator ผ่าน Xcode
# Xcode > Open Developer Tool > Simulator

# หรือผ่าน command line
open -a Simulator

# ดู simulators ที่มี
xcrun simctl list devices

# Boot simulator
xcrun simctl boot "iPhone 15"
```

---

## Setup Emulator

### Android Virtual Device (AVD) Manager

**สร้าง Emulator ใหม่:**

1. เปิด Android Studio
2. Tools > Device Manager (หรือ AVD Manager)
3. กด "+ Create Virtual Device"
4. เลือก:
   - Hardware: Pixel 7 (แนะนำ)
   - System Image: API 34 (Android 14)
5. กด Finish

**รัน Emulator:**

```bash
# ผ่าน Android Studio: กด Play button

# ผ่าน command line
emulator -avd Pixel_7_API_34

# ดู AVDs ที่มี
emulator -list-avds

# ตรวจสอบ devices ที่เชื่อมต่อ
adb devices
```

### Physical Device (แนะนำ)

การใช้ Physical Device เร็วกว่า Emulator มาก

**Android:**

```bash
# 1. เปิด Developer Options บน Android
#    Settings > About Phone > กด Build Number 7 ครั้ง

# 2. เปิด USB Debugging
#    Settings > Developer Options > USB Debugging

# 3. เชื่อมต่อด้วย USB แล้วตรวจสอบ
adb devices
# List of devices attached
# XXXXXXXXXX    device

# หรือใช้ Wireless Debugging (Android 11+)
adb connect <IP>:5555
```

**iOS:**

```bash
# 1. เชื่อมต่อ iPhone ด้วย USB
# 2. Trust computer บน iPhone
# 3. ตรวจสอบใน Xcode:
#    Window > Devices and Simulators
```

---

## Setup VS Code

### ดาวน์โหลดและติดตั้ง

ไปที่ https://code.visualstudio.com และดาวน์โหลดสำหรับ OS ของคุณ

### Extensions ที่แนะนำ

**จำเป็น:**

```
1. ES7+ React/Redux/React-Native snippets
   Publisher: dsznajder
   ID: dsznajder.es7-react-js-snippets
   ใช้สำหรับ: Code snippets สำหรับ React/React Native

2. Prettier - Code formatter
   Publisher: Prettier
   ID: esbenp.prettier-vscode
   ใช้สำหรับ: Format code อัตโนมัติ

3. ESLint
   Publisher: Microsoft
   ID: dbaeumer.vscode-eslint
   ใช้สำหรับ: ตรวจสอบ code quality

4. TypeScript Hero
   Publisher: rbbit
   ใช้สำหรับ: TypeScript imports

5. Auto Import
   Publisher: steoates
   ID: steoates.autoimport
   ใช้สำหรับ: Auto import modules
```

**แนะนำ:**

```
6. GitLens
   ดู git history และ blame

7. Path Intellisense
   Auto-complete file paths

8. Bracket Pair Colorizer 2
   จับคู่ brackets ด้วยสี (มีใน VS Code แล้ว)

9. Material Icon Theme
   ไอคอนสวยงาม

10. One Dark Pro / Dracula Theme
    Theme สวยงาม

11. Thunder Client
    REST API testing (แทน Postman)

12. React Native Tools
    Publisher: Microsoft
    Debugging สำหรับ React Native
```

### ติดตั้ง Extensions ผ่าน Command Line

```bash
# ติดตั้งทีละตัว
code --install-extension dsznajder.es7-react-js-snippets
code --install-extension esbenp.prettier-vscode
code --install-extension dbaeumer.vscode-eslint
code --install-extension eamodio.gitlens
code --install-extension christian-kohler.path-intellisense

# ตรวจสอบ extensions ที่ติดตั้ง
code --list-extensions
```

### ตั้งค่า VS Code สำหรับ React Native

สร้างไฟล์ `.vscode/settings.json` ใน project:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.fontSize": 14,
  "editor.lineHeight": 1.6,
  "editor.fontFamily": "Fira Code, Consolas, monospace",
  "editor.fontLigatures": true,
  "editor.wordWrap": "on",
  "editor.minimap.enabled": false,
  "terminal.integrated.fontSize": 13,
  "files.associations": {
    "*.jsx": "javascriptreact",
    "*.tsx": "typescriptreact"
  },
  "emmet.includeLanguages": {
    "javascript": "javascriptreact"
  },
  "javascript.preferences.quoteStyle": "single",
  "typescript.preferences.quoteStyle": "single",
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

### ตั้งค่า Prettier

สร้างไฟล์ `.prettierrc` ใน root ของ project:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "jsxBracketSameLine": false,
  "arrowParens": "avoid"
}
```

### ตั้งค่า ESLint

สร้างไฟล์ `.eslintrc.js`:

```javascript
module.exports = {
  root: true,
  extends: [
    '@react-native',
    'prettier',
  ],
  plugins: ['react', 'react-native', 'react-hooks'],
  rules: {
    // React Native specific
    'react-native/no-unused-styles': 'warn',
    'react-native/no-inline-styles': 'warn',
    'react-native/no-color-literals': 'warn',
    
    // React hooks
    'react-hooks/rules-of-hooks': 'error',
    'react-hooks/exhaustive-deps': 'warn',
    
    // General
    'no-console': 'warn',
    'no-unused-vars': ['warn', { argsIgnorePattern: '^_' }],
    'prefer-const': 'error',
  },
};
```

### Keyboard Shortcuts ที่ควรรู้

```
General:
Ctrl/Cmd + P      - เปิด file ด้วยชื่อ
Ctrl/Cmd + Shift + P  - Command Palette
Ctrl/Cmd + `      - เปิด Terminal
Ctrl/Cmd + B      - Toggle Sidebar

Editing:
Ctrl/Cmd + D      - Select next occurrence
Alt + Up/Down     - Move line up/down
Ctrl/Cmd + /      - Toggle comment
Ctrl/Cmd + Shift + K  - Delete line
Alt + Shift + Down  - Duplicate line

Navigation:
Ctrl/Cmd + G      - Go to line
F12               - Go to definition
Alt + F12         - Peek definition
Ctrl/Cmd + -      - Navigate back
```

---

## ตรวจสอบ Environment

### ใช้ react-native doctor

```bash
# สร้าง project แล้วรัน
npx @react-native-community/cli doctor

# หรือ
npx react-native doctor
```

ผลลัพธ์ที่ควรเห็น:

```
React Native Doctor
✓ Node.js: 20.10.0 (required: >= 18.0.0)
✓ npm: 10.2.4
✓ Android: Android Studio Hedgehog installed
  ✓ ANDROID_HOME set to: /Users/user/Library/Android/sdk
  ✓ adb: 1.0.41
  ✓ Emulator: Pixel_7_API_34 (running)
✓ iOS (macOS only):
  ✓ Xcode: 15.2 installed
  ✓ CocoaPods: 1.14.3
  ✓ iOS Simulator available
```

### ตรวจสอบด้วยตนเอง

```bash
# ตรวจสอบทุกอย่าง
echo "=== Node.js ===" && node --version
echo "=== npm ===" && npm --version
echo "=== yarn ===" && yarn --version
echo "=== Java ===" && java --version
echo "=== adb ===" && adb version
echo "=== Android SDK ===" && echo $ANDROID_HOME

# macOS เพิ่มเติม
echo "=== Xcode ===" && xcodebuild -version
echo "=== CocoaPods ===" && pod --version
echo "=== Simulators ===" && xcrun simctl list devices | grep "iPhone" | head -5
```

### สร้าง Project ทดสอบ

```bash
# สร้าง test project
npx @react-native-community/cli@latest init TestProject --version 0.74

cd TestProject

# Android
npx react-native run-android

# iOS (macOS เท่านั้น)
cd ios && pod install && cd ..
npx react-native run-ios
```

---

## Workshop

### Workshop 2.1: ติดตั้ง Environment ครบชุด

**เป้าหมาย:** ติดตั้งทุกอย่างที่จำเป็นและตรวจสอบว่าทำงานได้

**Checklist:**

```
[ ] ติดตั้ง Node.js 20+ LTS
[ ] ติดตั้ง yarn หรือ npm
[ ] ติดตั้ง Android Studio
[ ] ตั้งค่า ANDROID_HOME
[ ] ติดตั้ง JDK 17
[ ] สร้าง Android Emulator (Pixel 7 API 34)
[ ] (macOS) ติดตั้ง Xcode
[ ] (macOS) ติดตั้ง CocoaPods
[ ] ติดตั้ง VS Code
[ ] ติดตั้ง Extensions ที่แนะนำ
[ ] รัน react-native doctor ผ่านหมด
```

### Workshop 2.2: สร้าง Project ทดสอบ

```bash
# สร้าง project
npx @react-native-community/cli@latest init HelloWorld --version 0.74

cd HelloWorld

# เปิดใน VS Code
code .

# รัน Metro bundler
npx react-native start

# เปิด Terminal ใหม่ แล้วรัน Android
npx react-native run-android

# หรือ iOS (macOS)
npx react-native run-ios
```

**สิ่งที่ควรเห็น:**
- Metro Bundler เปิดใน Terminal
- Emulator/Simulator เปิดขึ้น
- แอป "HelloWorld" แสดง Welcome Screen ของ React Native

### Workshop 2.3: ตั้งค่า VS Code สำหรับ Project

```bash
# สร้างไฟล์ config ใน project
mkdir .vscode
touch .vscode/settings.json
touch .vscode/launch.json
touch .prettierrc
touch .eslintrc.js
```

เพิ่ม launch.json สำหรับ Debug:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Android",
      "request": "launch",
      "type": "reactnative",
      "platform": "android"
    },
    {
      "name": "Debug iOS",
      "request": "launch",
      "type": "reactnative",
      "platform": "ios"
    },
    {
      "name": "Attach to packager",
      "request": "attach",
      "type": "reactnative"
    }
  ]
}
```

### Workshop 2.4: ทดสอบ Hot Reload

```bash
# เปิดแอปบน emulator แล้ว
# แก้ไขไฟล์ App.tsx
# เปลี่ยน text "Step One" เป็น "สวัสดี React Native!"
# บันทึกไฟล์ (Ctrl+S)
# ดูว่าแอปอัปเดตอัตโนมัติ
```

---

## แก้ปัญหาที่พบบ่อย

### ปัญหา: Android SDK not found

```bash
# ตรวจสอบ ANDROID_HOME
echo $ANDROID_HOME

# ถ้าว่างเปล่า ให้ตั้งค่า
# macOS - เพิ่มใน ~/.zshrc
echo 'export ANDROID_HOME=$HOME/Library/Android/sdk' >> ~/.zshrc
echo 'export PATH=$PATH:$ANDROID_HOME/platform-tools' >> ~/.zshrc
source ~/.zshrc

# Windows - ตั้งค่า Environment Variables ใน System Properties
```

### ปัญหา: Java version ไม่ถูกต้อง

```bash
# ตรวจสอบ Java version
java --version

# ถ้าไม่ใช่ JDK 17 ให้ติดตั้ง
# macOS
brew install --cask temurin@17

# ตั้งค่า JAVA_HOME
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
```

### ปัญหา: Emulator ช้า/ค้าง

```bash
# ตรวจสอบ Virtualization
# Windows: Task Manager > Performance > Virtualization: Enabled
# BIOS: เปิด VT-x (Intel) หรือ AMD-V (AMD)

# เพิ่ม RAM ให้ Emulator
# AVD Manager > Edit > Advanced Settings > RAM: 2048+ MB

# ใช้ Hardware Graphics
# AVD Manager > Edit > Graphics: Hardware - GLES 2.0
```

### ปัญหา: CocoaPods ติดตั้งไม่ได้ (macOS)

```bash
# ลอง Homebrew
brew install cocoapods

# ถ้า permission error
sudo gem install cocoapods

# ถ้า Ruby version เก่า
brew install rbenv
rbenv install 3.2.2
rbenv global 3.2.2
gem install cocoapods
```

### ปัญหา: Metro Bundler เริ่มไม่ได้

```bash
# Clear cache
npx react-native start --reset-cache

# ล้าง node_modules
rm -rf node_modules
npm install

# ล้าง Metro cache
npx react-native start --reset-cache
```

### ปัญหา: Android Build Failed

```bash
# Clean build
cd android
./gradlew clean
cd ..
npx react-native run-android

# หรือ clean + rebuild
./gradlew clean assembleDebug

# ถ้า Java heap space error
# เพิ่มใน android/gradle.properties:
# org.gradle.jvmargs=-Xmx4096m
```

### ปัญหา: iOS Pod Install Failed

```bash
cd ios

# ล้าง pods
rm -rf Pods
rm -rf ~/Library/Caches/CocoaPods
pod cache clean --all

# ติดตั้งใหม่
pod install

# ถ้ายังไม่ได้
pod install --repo-update
```

### ปัญหา: ENOSPC (Linux)

```bash
# เพิ่ม file watchers
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## สรุป Part 002

### Environment ที่ Setup แล้ว

```
✅ Node.js 20+ LTS
✅ npm/yarn
✅ Android Studio + Android SDK
✅ JDK 17
✅ Android Emulator (Pixel 7 API 34)
✅ (macOS) Xcode + Command Line Tools
✅ (macOS) CocoaPods
✅ VS Code + Extensions
✅ Prettier + ESLint config
```

### คำสั่งที่ใช้บ่อย

```bash
# สร้าง project ใหม่
npx @react-native-community/cli@latest init ProjectName

# รัน Metro bundler
npx react-native start

# รัน Android
npx react-native run-android

# รัน iOS
npx react-native run-ios

# ตรวจสอบ environment
npx react-native doctor

# Clear cache
npx react-native start --reset-cache
```

---

**ต่อไป → Part 003: สร้าง Project แรกด้วย React Native**
