# Part 030: Build และ Deploy เบื้องต้น

## บทนำ

การ build และ deploy app ไปยัง App Store และ Google Play Store เป็นขั้นตอนสุดท้ายที่สำคัญ ในบทนี้เราจะเรียนรู้กระบวนการ build ทั้ง iOS และ Android ตั้งแต่การเตรียม certificates จนถึงการส่ง app ขึ้น store

## สารบัญ

1. Build Modes: Debug vs Release
2. Android: Generate Keystore, Build APK/AAB
3. iOS: Certificates, Provisioning Profiles
4. Expo Build (EAS Build)
5. Submit to Stores
6. Workshop: Build Your First Release

---

## 1. Build Modes: Debug vs Release

### ความแตกต่างระหว่าง Debug และ Release

| Feature | Debug | Release |
|---------|-------|---------|
| JS Bundle | Unbundled (Metro) | Bundled (minified) |
| Performance | ช้ากว่า | เร็วกว่า |
| Error Messages | Detailed | Generic |
| Dev Tools | เปิดใช้งาน | ปิด |
| File Size | ใหญ่ | เล็ก |
| __DEV__ | true | false |
| Hermes Engine | optional | enabled |

```javascript
// ตรวจสอบ build mode ใน code
if (__DEV__) {
  console.log('Running in Debug mode');
  // Enable extra logging
  // Show developer hints
}

// Environment variables สำหรับ build modes
const API_URL = __DEV__
  ? 'https://api-dev.example.com'
  : 'https://api.example.com';

// ใช้กับ constants
export const CONFIG = {
  API_URL: __DEV__ ? 'http://localhost:3000' : 'https://api.example.com',
  LOG_LEVEL: __DEV__ ? 'debug' : 'error',
  ENABLE_ANALYTICS: !__DEV__,
  MAX_RETRY: __DEV__ ? 1 : 3,
};
```

### Environment Configuration

```bash
# ติดตั้ง react-native-config หรือ expo-constants
npx expo install expo-constants
```

```javascript
// app.config.js
export default {
  expo: {
    name: 'My App',
    slug: 'my-app',
    version: '1.0.0',
    extra: {
      apiUrl: process.env.API_URL || 'https://api.example.com',
      sentryDsn: process.env.SENTRY_DSN,
      environment: process.env.APP_ENV || 'production',
    },
  },
};

// ใช้งาน
import Constants from 'expo-constants';

const apiUrl = Constants.expoConfig?.extra?.apiUrl;
const environment = Constants.expoConfig?.extra?.environment;
```

### ตั้งค่า Environment Variables

```bash
# .env.development
API_URL=http://localhost:3000
APP_ENV=development
ENABLE_LOGS=true

# .env.staging
API_URL=https://api-staging.example.com
APP_ENV=staging
ENABLE_LOGS=true

# .env.production
API_URL=https://api.example.com
APP_ENV=production
ENABLE_LOGS=false
```

---

## 2. Android: Generate Keystore, Build APK/AAB

### ทำความเข้าใจ Android Signing

ทุก app ที่ upload ขึ้น Google Play ต้องได้รับการ sign ด้วย digital certificate (keystore)

```
Keystore Flow:
Developer Machine → Keystore → Sign APK/AAB → Google Play Store
```

### Step 1: สร้าง Keystore

```bash
# สร้าง keystore ด้วย keytool (มาพร้อม JDK)
keytool -genkeypair \
  -v \
  -storetype PKCS12 \
  -keystore my-app-release.keystore \
  -alias my-key-alias \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000

# กรอกข้อมูล:
# Enter keystore password: [สร้างรหัสผ่านแข็งแกร่ง]
# Re-enter new password: [ยืนยันรหัสผ่าน]
# What is your first and last name? [ชื่อนักพัฒนา]
# What is the name of your organizational unit? [หน่วยงาน]
# What is the name of your organization? [ชื่อบริษัท]
# What is the name of your City or Locality? [เมือง]
# What is the name of your State or Province? [จังหวัด]
# What is the two-letter country code? TH
```

**สำคัญ:** เก็บ keystore และรหัสผ่านอย่างปลอดภัย ถ้าสูญหายจะไม่สามารถ update app บน Play Store ได้!

### Step 2: ตั้งค่า Gradle

```bash
# สร้างไฟล์ ~/.gradle/gradle.properties (นอก project)
# หรือ android/gradle.properties

MY_RELEASE_STORE_FILE=my-app-release.keystore
MY_RELEASE_KEY_ALIAS=my-key-alias
MY_RELEASE_STORE_PASSWORD=your_keystore_password
MY_RELEASE_KEY_PASSWORD=your_key_password
```

```groovy
// android/app/build.gradle
android {
    ...
    signingConfigs {
        release {
            if (project.hasProperty('MY_RELEASE_STORE_FILE')) {
                storeFile file(MY_RELEASE_STORE_FILE)
                storePassword MY_RELEASE_STORE_PASSWORD
                keyAlias MY_RELEASE_KEY_ALIAS
                keyPassword MY_RELEASE_KEY_PASSWORD
            }
        }
    }
    buildTypes {
        release {
            ...
            signingConfig signingConfigs.release
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
        }
    }
}
```

### Step 3: Build APK (สำหรับทดสอบ)

```bash
# Navigate ไปที่ android directory
cd android

# Build debug APK
./gradlew assembleDebug
# หรือบน Windows: gradlew.bat assembleDebug

# Build release APK
./gradlew assembleRelease

# APK จะอยู่ที่:
# android/app/build/outputs/apk/release/app-release.apk

# Build Debug APK พร้อมติดตั้งบน device ทันที
./gradlew installDebug
```

### Step 4: Build AAB (สำหรับ Play Store)

```bash
# Build Android App Bundle (แนะนำสำหรับ Play Store)
cd android
./gradlew bundleRelease

# AAB จะอยู่ที่:
# android/app/build/outputs/bundle/release/app-release.aab
```

### ข้อมูลในไฟล์ app.json / build.gradle

```json
// app.json (Expo)
{
  "expo": {
    "android": {
      "package": "com.yourcompany.yourapp",
      "versionCode": 1,
      "permissions": [
        "CAMERA",
        "READ_EXTERNAL_STORAGE",
        "WRITE_EXTERNAL_STORAGE"
      ]
    }
  }
}
```

```groovy
// android/app/build.gradle
android {
    compileSdkVersion 34
    defaultConfig {
        applicationId "com.yourcompany.yourapp"
        minSdkVersion 23
        targetSdkVersion 34
        versionCode 1
        versionName "1.0.0"
    }
}
```

---

## 3. iOS: Certificates, Provisioning Profiles

iOS มีระบบ security ที่เข้มงวดมาก ต้องมี Apple Developer Account ก่อน

### Apple Developer Account

```
ราคา Apple Developer Program:
- Individual: $99/year
- Organization: $99/year (ต้อง D-U-N-S Number)
- Enterprise: $299/year
```

### Certificates ที่จำเป็น

```
iOS Certificates:
1. Development Certificate  → ทดสอบบน device ของตัวเอง
2. Distribution Certificate → ส่ง app ขึ้น App Store
```

### ขั้นตอนสร้าง Certificates (ใน Xcode)

```
1. Xcode → Preferences → Accounts
2. เพิ่ม Apple ID
3. เลือก Team
4. Manage Certificates → + → Apple Distribution

หรือผ่าน developer.apple.com:
1. Certificates, Identifiers & Profiles
2. Certificates → + (Add)
3. เลือก Apple Distribution
4. Upload CSR (Certificate Signing Request)
5. Download และ double-click เพื่อ install
```

### Provisioning Profile

```
Provisioning Profile เชื่อม:
- App ID (Bundle Identifier)
- Device UDIDs (Development only)
- Certificate

ประเภท:
1. Development    → ทดสอบบน device ที่ลงทะเบียน
2. Ad Hoc         → แจกจ่ายให้ testers ได้ถึง 100 devices
3. App Store      → ส่งขึ้น App Store
4. Enterprise     → สำหรับ in-house distribution
```

### Info.plist สำคัญ

```xml
<!-- ios/MyApp/Info.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <!-- Bundle Identifier -->
    <key>CFBundleIdentifier</key>
    <string>com.yourcompany.yourapp</string>
    
    <!-- Version -->
    <key>CFBundleShortVersionString</key>
    <string>1.0.0</string>
    
    <!-- Build Number -->
    <key>CFBundleVersion</key>
    <string>1</string>
    
    <!-- Permissions (ต้องมีคำอธิบาย) -->
    <key>NSCameraUsageDescription</key>
    <string>เราต้องการใช้กล้องเพื่อถ่ายรูปโปรไฟล์</string>
    
    <key>NSPhotoLibraryUsageDescription</key>
    <string>เราต้องการเข้าถึงรูปภาพของคุณ</string>
    
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>เราต้องการตำแหน่งของคุณเพื่อแสดงร้านอาหารใกล้เคียง</string>
    
    <!-- Face ID -->
    <key>NSFaceIDUsageDescription</key>
    <string>ใช้ Face ID เพื่อเข้าสู่ระบบได้เร็วขึ้น</string>
</dict>
</plist>
```

### Build ด้วย Xcode

```bash
# Clean build
cd ios && xcodebuild clean

# Build สำหรับ Simulator
cd ios && xcodebuild -scheme MyApp -sdk iphonesimulator -configuration Debug

# Build สำหรับ Device (Archive)
# 1. เปิด Xcode
# 2. เลือก Generic iOS Device
# 3. Product → Archive
# 4. Window → Organizer → Distribute App
```

---

## 4. Expo Build (EAS Build)

EAS (Expo Application Services) Build ทำให้การ build ง่ายขึ้นมาก

### ติดตั้ง EAS CLI

```bash
npm install -g eas-cli

# Login
eas login

# ตรวจสอบ account
eas whoami
```

### ตั้งค่า EAS Build

```bash
# สร้าง eas.json อัตโนมัติ
eas build:configure
```

```json
// eas.json
{
  "cli": {
    "version": ">= 7.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      },
      "ios": {
        "simulator": false
      }
    },
    "preview": {
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "staging": {
      "distribution": "internal",
      "android": {
        "buildType": "aab"
      },
      "env": {
        "APP_ENV": "staging"
      }
    },
    "production": {
      "distribution": "store",
      "android": {
        "buildType": "aab"
      },
      "env": {
        "APP_ENV": "production"
      }
    }
  },
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./google-service-account.json",
        "track": "internal"
      },
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "1234567890"
      }
    }
  }
}
```

### Build Commands

```bash
# Build สำหรับ Android
eas build --platform android --profile production

# Build สำหรับ iOS
eas build --platform ios --profile production

# Build ทั้ง 2 platforms พร้อมกัน
eas build --platform all --profile production

# Build development client
eas build --profile development

# ดู build status
eas build:list

# Cancel build
eas build:cancel
```

### Credentials Management

```bash
# EAS จัดการ credentials ให้อัตโนมัติ (แนะนำ)
eas credentials

# หรือ manual:
# Android - EAS จะถามว่าต้องการสร้าง keystore ใหม่หรือใช้ที่มีอยู่
# iOS - EAS จะสร้าง certificates และ provisioning profiles ให้

# ดู credentials ที่มีอยู่
eas credentials --platform android
eas credentials --platform ios
```

### app.config.js สำหรับ EAS

```javascript
// app.config.js
export default ({ config }) => ({
  ...config,
  name: 'My App',
  slug: 'my-app',
  version: '1.0.0',
  orientation: 'portrait',
  icon: './assets/icon.png',
  splash: {
    image: './assets/splash.png',
    resizeMode: 'contain',
    backgroundColor: '#ffffff',
  },
  ios: {
    bundleIdentifier: 'com.yourcompany.myapp',
    buildNumber: '1',
    supportsTablet: false,
    infoPlist: {
      NSCameraUsageDescription: 'ใช้กล้องเพื่อถ่ายรูป',
      NSPhotoLibraryUsageDescription: 'เข้าถึงรูปภาพ',
    },
  },
  android: {
    package: 'com.yourcompany.myapp',
    versionCode: 1,
    adaptiveIcon: {
      foregroundImage: './assets/adaptive-icon.png',
      backgroundColor: '#FFFFFF',
    },
    permissions: ['CAMERA', 'READ_EXTERNAL_STORAGE'],
  },
  extra: {
    apiUrl: process.env.API_URL,
    eas: {
      projectId: 'your-eas-project-id',
    },
  },
});
```

### OTA Updates ด้วย EAS Update

```bash
# ติดตั้ง expo-updates
npx expo install expo-updates

# ส่ง update โดยไม่ต้อง rebuild
eas update --branch production --message "Fix login bug"

# ส่ง update ไปยัง branch staging
eas update --branch staging --message "Test new feature"
```

```javascript
// ใน app.config.js
export default {
  expo: {
    updates: {
      url: 'https://u.expo.dev/your-project-id',
    },
    runtimeVersion: {
      policy: 'appVersion',
    },
  },
};
```

---

## 5. Submit to Stores

### Google Play Store

**ขั้นตอน:**

```
1. สร้าง Google Play Console Account
   - ค่าธรรมเนียม: $25 (one-time)
   - เว็บไซต์: play.google.com/console

2. สร้าง App ใน Console
   - Create Application
   - กรอก Default Language, Title

3. เตรียม Assets
   - Icon: 512x512 PNG
   - Feature Graphic: 1024x500 JPG/PNG
   - Screenshots: >=2 ภาพต่อ device type
   - Short Description: <=80 ตัวอักษร
   - Full Description: <=4000 ตัวอักษร

4. ตั้งค่า Privacy Policy URL (จำเป็น!)

5. Content Rating
   - กรอก questionnaire
   - จะได้ rating โดยอัตโนมัติ

6. Upload AAB/APK
   - Production → Releases → Create release
   - อัพโหลด .aab file

7. Submit for Review
   - ปกติใช้เวลา 1-3 วัน
```

### ส่งด้วย EAS Submit (Android)

```bash
# Submit AAB ที่ build ล่าสุดขึ้น Play Store
eas submit --platform android

# หรือระบุ build
eas submit --platform android --id [build-id]

# ต้องมี Google Service Account JSON:
# 1. Google Play Console → Setup → API Access
# 2. Link to Google Cloud Project
# 3. Create Service Account
# 4. ดาวน์โหลด JSON key
```

### Apple App Store

**ขั้นตอน:**

```
1. Apple Developer Account ($99/year)
   - developer.apple.com

2. App Store Connect
   - appstoreconnect.apple.com
   - My Apps → + → New App

3. เตรียม Assets
   - App Icon: 1024x1024 PNG (ไม่มีมุมโค้ง, ไม่มี alpha)
   - Screenshots:
     * iPhone 6.7": 1290x2796 px (iPhone 15 Pro Max)
     * iPhone 6.5": 1242x2688 px (iPhone 11 Pro Max)
     * iPad 12.9": 2048x2732 px (optional)
   - App Preview Videos (optional)
   - Description: <=4000 ตัวอักษร
   - Keywords: <=100 ตัวอักษร
   - Support URL
   - Privacy Policy URL

4. Pricing and Availability
   - ราคา (Free หรือ Paid)
   - Countries ที่จำหน่าย

5. Submit for Review
   - ปกติใช้เวลา 1-3 วัน (อาจนานกว่า)
   - Apple review เข้มกว่า Google
```

### ส่งด้วย EAS Submit (iOS)

```bash
# Submit ขึ้น App Store
eas submit --platform ios

# ต้องการ:
# - Apple ID
# - App-specific password (appleid.apple.com)
# - App Store Connect App ID

# หรือใช้ App Store Connect API Key:
eas submit --platform ios --apple-api-key-path ./AuthKey_XXXXX.p8
```

### Metadata Automation ด้วย Fastlane

```bash
# ติดตั้ง Fastlane
gem install fastlane

# ในโปรเจกต์
fastlane init
```

```ruby
# fastlane/Fastfile

# Android
lane :deploy_android do
  gradle(
    task: 'bundle',
    build_type: 'Release',
    project_dir: 'android/'
  )
  upload_to_play_store(
    track: 'internal',
    aab: 'android/app/build/outputs/bundle/release/app-release.aab',
    skip_upload_screenshots: true,
    skip_upload_images: true
  )
end

# iOS
lane :deploy_ios do
  build_app(
    scheme: 'MyApp',
    workspace: 'ios/MyApp.xcworkspace',
    configuration: 'Release'
  )
  upload_to_app_store(
    skip_screenshots: true,
    skip_metadata: true
  )
end

# OTA Update (Expo)
lane :update_app do
  sh("eas update --branch production --message 'Hot fix'")
end
```

---

## 6. Workshop: Build Your First Release

### Checklist ก่อน Build

```markdown
## Pre-Release Checklist

### Code
- [ ] ทดสอบบน iOS device จริง
- [ ] ทดสอบบน Android device จริง
- [ ] ทดสอบบน multiple screen sizes
- [ ] ไม่มี console.log ที่ sensitive data
- [ ] ตั้งค่า ENV variables สำหรับ production
- [ ] Version number อัพเดทแล้ว
- [ ] Build number อัพเดทแล้ว

### Assets
- [ ] App icon ขนาดถูกต้อง (1024x1024 iOS, 512x512 Android)
- [ ] Splash screen ขนาดถูกต้อง
- [ ] Screenshots พร้อมแล้ว

### App Store Requirements
- [ ] Privacy Policy URL
- [ ] Support URL
- [ ] App Description
- [ ] Keywords
- [ ] Category เลือกแล้ว

### Security
- [ ] Keystore เก็บปลอดภัย
- [ ] API keys ไม่อยู่ใน code
- [ ] Certificates ยังไม่หมดอายุ
```

### ตัวอย่าง Build Script

```javascript
// scripts/prepare-release.js
const fs = require('fs');
const path = require('path');

const packageJson = require('../package.json');
const appJson = require('../app.json');

// อ่าน version จาก package.json
const version = packageJson.version;
const [major, minor, patch] = version.split('.').map(Number);

// คำนวณ build number
const buildNumber = major * 10000 + minor * 100 + patch;

console.log(`
📦 Preparing Release Build
==========================
Version: ${version}
Build Number: ${buildNumber}

✅ Checklist:
`);

// ตรวจสอบไฟล์สำคัญ
const requiredFiles = [
  'assets/icon.png',
  'assets/splash.png',
  'assets/adaptive-icon.png',
];

requiredFiles.forEach((file) => {
  const exists = fs.existsSync(path.join(__dirname, '..', file));
  console.log(`  ${exists ? '✓' : '✗'} ${file}`);
});

// ตรวจสอบ env variables
const requiredEnvVars = ['API_URL'];
requiredEnvVars.forEach((envVar) => {
  const hasValue = !!process.env[envVar];
  console.log(`  ${hasValue ? '✓' : '✗'} ${envVar}`);
});

console.log('\n📝 Version Info:');
console.log(`  app.json version: ${appJson.expo.version}`);
console.log(`  iOS build: ${appJson.expo.ios?.buildNumber}`);
console.log(`  Android versionCode: ${appJson.expo.android?.versionCode}`);
```

### app.json ที่สมบูรณ์สำหรับ Production

```json
{
  "expo": {
    "name": "My Awesome App",
    "slug": "my-awesome-app",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "userInterfaceStyle": "automatic",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "assetBundlePatterns": [
      "**/*"
    ],
    "ios": {
      "supportsTablet": false,
      "bundleIdentifier": "com.yourcompany.myawesomeapp",
      "buildNumber": "1",
      "infoPlist": {
        "NSCameraUsageDescription": "เราต้องการกล้องเพื่อถ่ายรูปโปรไฟล์และสินค้า",
        "NSPhotoLibraryUsageDescription": "เราต้องการเข้าถึงรูปภาพเพื่อเลือกรูปโปรไฟล์",
        "NSLocationWhenInUseUsageDescription": "เราต้องการตำแหน่งของคุณเพื่อแสดงร้านค้าใกล้เคียง",
        "NSFaceIDUsageDescription": "ใช้ Face ID เพื่อเข้าสู่ระบบได้รวดเร็วยิ่งขึ้น",
        "ITSAppUsesNonExemptEncryption": false
      }
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#FFFFFF"
      },
      "package": "com.yourcompany.myawesomeapp",
      "versionCode": 1,
      "permissions": [
        "android.permission.CAMERA",
        "android.permission.READ_EXTERNAL_STORAGE"
      ],
      "googleServicesFile": "./google-services.json"
    },
    "plugins": [
      "expo-router",
      [
        "expo-camera",
        {
          "cameraPermission": "เราต้องการกล้องเพื่อถ่ายรูปสินค้า"
        }
      ]
    ],
    "extra": {
      "eas": {
        "projectId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      }
    }
  }
}
```

### Deploy Pipeline ด้วย GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches:
      - main
    tags:
      - 'v*'

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install EAS CLI
        run: npm install -g eas-cli

      - name: Build Android
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
        run: |
          eas build --platform android \
            --profile production \
            --non-interactive \
            --no-wait

  build-ios:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install EAS CLI
        run: npm install -g eas-cli

      - name: Build iOS
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
        run: |
          eas build --platform ios \
            --profile production \
            --non-interactive \
            --no-wait

  deploy:
    needs: [build-android, build-ios]
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - name: Submit to stores
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
        run: |
          eas submit --platform all \
            --profile production \
            --non-interactive
```

### Version Management Script

```javascript
// scripts/bump-version.js
const fs = require('fs');
const { execSync } = require('child_process');

const bumpType = process.argv[2]; // 'major' | 'minor' | 'patch'

if (!['major', 'minor', 'patch'].includes(bumpType)) {
  console.error('Usage: node scripts/bump-version.js [major|minor|patch]');
  process.exit(1);
}

// อ่าน package.json
const packagePath = './package.json';
const pkg = JSON.parse(fs.readFileSync(packagePath, 'utf8'));
const [major, minor, patch] = pkg.version.split('.').map(Number);

// เพิ่ม version
let newVersion;
if (bumpType === 'major') newVersion = `${major + 1}.0.0`;
else if (bumpType === 'minor') newVersion = `${major}.${minor + 1}.0`;
else newVersion = `${major}.${minor}.${patch + 1}`;

const [newMajor, newMinor, newPatch] = newVersion.split('.').map(Number);
const newBuildNumber = newMajor * 10000 + newMinor * 100 + newPatch;

// อัพเดท package.json
pkg.version = newVersion;
fs.writeFileSync(packagePath, JSON.stringify(pkg, null, 2) + '\n');

// อัพเดท app.json
const appJsonPath = './app.json';
const appJson = JSON.parse(fs.readFileSync(appJsonPath, 'utf8'));
appJson.expo.version = newVersion;
appJson.expo.ios.buildNumber = String(newBuildNumber);
appJson.expo.android.versionCode = newBuildNumber;
fs.writeFileSync(appJsonPath, JSON.stringify(appJson, null, 2) + '\n');

console.log(`✅ Version bumped: ${pkg.version} → ${newVersion}`);
console.log(`   Build Number: ${newBuildNumber}`);

// Git commit และ tag
execSync('git add package.json app.json');
execSync(`git commit -m "chore: bump version to ${newVersion}"`);
execSync(`git tag v${newVersion}`);

console.log(`✅ Git commit and tag created: v${newVersion}`);
console.log('   Run: git push && git push --tags');
```

```bash
# ใช้งาน
node scripts/bump-version.js patch   # 1.0.0 → 1.0.1
node scripts/bump-version.js minor   # 1.0.0 → 1.1.0
node scripts/bump-version.js major   # 1.0.0 → 2.0.0
```

---

## Tips และ Best Practices

### 1. Version Naming Convention

```
App Version: MAJOR.MINOR.PATCH
  MAJOR: Breaking changes หรือ redesign ใหญ่
  MINOR: New features
  PATCH: Bug fixes

Build Number: Auto-calculated หรือ auto-increment
  iOS Build Number: String (e.g., "1", "100")
  Android versionCode: Integer (e.g., 1, 100)
```

### 2. Keystore Security

```bash
# ✅ เก็บ keystore ใน secure location
# ✅ Backup หลายที่ (encrypted)
# ✅ ไม่ commit keystore ใน git
# ✅ ใช้ environment variables สำหรับ passwords

# .gitignore
*.keystore
*.jks
google-services.json
GoogleService-Info.plist
.env*
```

### 3. TestFlight / Internal Testing

```bash
# iOS: TestFlight
# 1. Upload build ไป App Store Connect
# 2. TestFlight → + Build
# 3. เพิ่ม External Testers (ส่ง email invite)

# Android: Internal Testing
# 1. Play Console → Testing → Internal Testing
# 2. Create release → Upload AAB
# 3. Add testers (email list)

# ด้วย EAS
eas submit --platform ios --profile preview    # TestFlight
eas submit --platform android --profile preview  # Internal Testing
```

### 4. Release Notes

```bash
# iOS App Store Connect - What's New
# ต้องกรอก per language

# Android Google Play Console - Release Notes
# ต้องกรอก per language

# ตัวอย่าง release notes ภาษาไทย
"สิ่งใหม่ใน Version 1.1.0:
• เพิ่มฟีเจอร์การชำระเงินด้วย QR Code
• ปรับปรุงความเร็วในการโหลดหน้าแรก
• แก้ไขปัญหาการแจ้งเตือนบน iOS 17
• UI ใหม่สำหรับหน้าโปรไฟล์"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- ความแตกต่างระหว่าง Debug และ Release build
- Android: สร้าง Keystore และ Build APK/AAB
- iOS: Certificates, Provisioning Profiles, และ Build
- EAS Build สำหรับการ build ที่ง่ายและ automated
- การ Submit app ขึ้น Play Store และ App Store
- Workshop: Build Release พร้อม checklist และ automation scripts

---

## สรุปทั้งหมด Parts 021-030

เราได้เรียนรู้ครบทุกหัวข้อแล้ว:

| Part | หัวข้อ |
|------|--------|
| 021 | Modal, Alert, ActionSheet |
| 022 | ScrollView, KeyboardAvoidingView, SafeAreaView |
| 023 | Touchable Components และ Pressable |
| 024 | Icons และ Vector Icons |
| 025 | Fonts และ Typography |
| 026 | Colors และ Themes |
| 027 | Platform-specific Code |
| 028 | Debugging เบื้องต้น |
| 029 | Testing เบื้องต้น |
| 030 | Build และ Deploy เบื้องต้น |

ขั้นตอนต่อไปหลังจากนี้:
1. พัฒนา app จริงโดยใช้ความรู้ทั้งหมดที่ได้เรียน
2. ศึกษา Navigation ขั้นสูง (React Navigation v6)
3. เรียนรู้ State Management (Redux Toolkit, Zustand)
4. ทำความเข้าใจ Performance Optimization
5. เพิ่ม CI/CD pipeline
6. ศึกษา Native Modules สำหรับ functionality พิเศษ

สู้ต่อไปครับ! 🚀
