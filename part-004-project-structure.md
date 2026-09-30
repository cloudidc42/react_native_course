# Part 004: ทำความเข้าใจโครงสร้าง Project

## สารบัญ
1. [ภาพรวมโครงสร้าง Project](#ภาพรวม)
2. [ไฟล์ Configuration หลัก](#configuration-files)
3. [package.json อย่างละเอียด](#package-json)
4. [metro.config.js](#metro-config)
5. [babel.config.js](#babel-config)
6. [tsconfig.json](#tsconfig)
7. [android/ folder](#android-folder)
8. [ios/ folder](#ios-folder)
9. [src/ Structure Best Practices](#src-structure)
10. [Workshop: จัดโครงสร้าง Project](#workshop)

---

## ภาพรวมโครงสร้าง Project

```
MyApp/
├── __tests__/                    # Unit tests
│   └── App.test.tsx
│
├── android/                      # Android Native Code
│   ├── app/
│   │   ├── src/
│   │   │   ├── debug/
│   │   │   │   └── AndroidManifest.xml
│   │   │   └── main/
│   │   │       ├── AndroidManifest.xml
│   │   │       ├── java/com/myapp/
│   │   │       │   ├── MainActivity.kt
│   │   │       │   └── MainApplication.kt
│   │   │       └── res/
│   │   │           ├── drawable/
│   │   │           ├── mipmap-*/  (App icons)
│   │   │           └── values/
│   │   └── build.gradle
│   ├── gradle/
│   │   └── wrapper/
│   │       ├── gradle-wrapper.jar
│   │       └── gradle-wrapper.properties
│   ├── .gitignore
│   ├── build.gradle
│   ├── gradle.properties
│   └── settings.gradle
│
├── ios/                          # iOS Native Code
│   ├── MyApp/
│   │   ├── AppDelegate.swift
│   │   ├── Info.plist
│   │   ├── LaunchScreen.storyboard
│   │   └── main.m
│   ├── MyApp.xcodeproj/
│   │   └── project.pbxproj
│   ├── MyApp.xcworkspace/
│   │   └── contents.xcworkspacedata
│   └── Podfile
│
├── src/                          # Source code (สร้างเอง)
│   ├── assets/
│   │   ├── images/
│   │   ├── fonts/
│   │   └── icons/
│   ├── components/
│   │   ├── common/
│   │   └── screens/
│   ├── navigation/
│   ├── screens/
│   ├── hooks/
│   ├── services/
│   ├── store/
│   ├── types/
│   └── utils/
│
├── .eslintrc.js                  # ESLint config
├── .gitignore                    # Git ignore rules
├── .prettierrc.js                # Prettier config
├── .watchmanconfig               # Watchman config
├── app.json                      # App name/config
├── babel.config.js               # Babel transpiler config
├── index.js                      # Entry point
├── jest.config.js                # Jest test config
├── metro.config.js               # Metro bundler config
├── package.json                  # Dependencies
├── package-lock.json             # Lock file
├── react-native.config.js        # RN config (optional)
└── tsconfig.json                 # TypeScript config
```

---

## Configuration Files

### .gitignore

ไฟล์นี้บอก Git ว่าอะไรไม่ควร commit

```gitignore
# Facebook อาจสร้าง .gitignore นี้ให้แล้ว
# แต่ควรรู้ว่า ignore อะไร

# Dependencies
node_modules/

# Build outputs
android/app/build/
ios/build/
ios/DerivedData/
ios/Pods/

# CocoaPods
ios/*.xcworkspace
/ios/Pods

# Watchman
.watchman-cookie-*
.watchmanconfig

# Android
*.keystore
!debug.keystore

# iOS
*.ipa
*.dSYM.zip
*.dSYM

# Environment variables
.env
.env.local
.env.*.local

# VS Code
.vscode/*
!.vscode/settings.json
!.vscode/launch.json
!.vscode/extensions.json

# Metro
.metro-health-check*

# Logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# OS
.DS_Store
Thumbs.db

# TypeScript
*.tsbuildinfo
```

### .watchmanconfig

Watchman คือเครื่องมือของ Facebook สำหรับ watch files

```json
{
  "ignore_dirs": ["__tests__", "coverage", "node_modules"]
}
```

---

## package.json

```json
{
  "name": "MyApp",
  "version": "0.0.1",
  "private": true,
  
  "scripts": {
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "lint": "eslint .",
    "start": "react-native start",
    "test": "jest",
    "typecheck": "tsc --noEmit",
    
    // คำสั่งเพิ่มเติมที่ควรเพิ่ม
    "clean": "react-native clean",
    "clean:android": "cd android && ./gradlew clean",
    "clean:ios": "cd ios && rm -rf build Pods && pod install",
    "build:android": "cd android && ./gradlew assembleRelease",
    "format": "prettier --write \"src/**/*.{ts,tsx,js,jsx}\"",
    "prepare": "husky install"
  },
  
  "dependencies": {
    // React และ React Native - Core
    "react": "18.3.1",
    "react-native": "0.74.0"
    
    // ตัวอย่าง dependencies ที่มักเพิ่ม
    // "@react-navigation/native": "^6.x.x",
    // "@react-navigation/native-stack": "^6.x.x",
    // "react-native-safe-area-context": "^4.x.x",
    // "react-native-screens": "^3.x.x",
    // "axios": "^1.x.x",
    // "react-native-mmkv": "^2.x.x",
  },
  
  "devDependencies": {
    // TypeScript
    "@tsconfig/react-native": "^3.0.0",
    "typescript": "5.0.4",
    
    // Testing
    "@babel/core": "^7.20.0",
    "@babel/preset-env": "^7.20.0",
    "@babel/runtime": "^7.20.0",
    "@react-native/babel-preset": "0.74.83",
    "@react-native/eslint-config": "0.74.83",
    "@react-native/metro-config": "0.74.83",
    "@react-native/typescript-config": "0.74.83",
    "@types/react": "^18.2.6",
    "@types/react-test-renderer": "^18.0.0",
    "babel-jest": "^29.6.3",
    "eslint": "^8.19.0",
    "jest": "^29.6.3",
    "prettier": "2.8.8",
    "react-test-renderer": "18.3.1"
  },
  
  "engines": {
    "node": ">=18"
  },
  
  "jest": {
    "preset": "react-native"
  }
}
```

### Semantic Versioning

```
version: "1.2.3"
         │ │ │
         │ │ └── Patch: Bug fixes
         │ └──── Minor: New features (backward compatible)
         └────── Major: Breaking changes

^ (caret) - อนุญาต minor + patch updates
  "^1.2.3" → อนุญาต 1.x.x แต่ไม่ใช่ 2.0.0

~ (tilde) - อนุญาต patch updates เท่านั้น
  "~1.2.3" → อนุญาต 1.2.x แต่ไม่ใช่ 1.3.0

ไม่มี prefix - exact version
  "1.2.3" → ต้องเป็น 1.2.3 เท่านั้น
```

### Commands ใน package.json

```bash
# เพิ่ม script ใหม่
# ใน package.json > "scripts":
{
  "scripts": {
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "start": "react-native start",
    "test": "jest",
    "lint": "eslint .",
    
    // เพิ่มเองได้
    "android:release": "cd android && ./gradlew assembleRelease",
    "clean": "react-native clean",
    "pods": "cd ios && pod install",
    "reset": "watchman watch-del-all && rm -rf node_modules && npm install"
  }
}

// รัน script
npm run android
npm run ios
npm run lint
npm run test
```

---

## metro.config.js

Metro คือ JavaScript bundler สำหรับ React Native

```javascript
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

/**
 * Metro configuration
 * https://reactnative.dev/docs/metro
 *
 * @type {import('metro-config').MetroConfig}
 */
const config = {};

module.exports = mergeConfig(getDefaultConfig(__dirname), config);
```

### การ customize Metro

```javascript
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const defaultConfig = getDefaultConfig(__dirname);
const { assetExts, sourceExts } = defaultConfig.resolver;

const config = {
  // ตั้งค่า resolver
  resolver: {
    // เพิ่ม file extensions ที่รองรับ
    assetExts: [...assetExts, 'lottie', 'glb', 'gltf'],
    sourceExts: [...sourceExts, 'svg'],
    
    // Path aliases (แทน relative paths)
    alias: {
      '@': './src',
      '@components': './src/components',
      '@screens': './src/screens',
      '@assets': './src/assets',
    },
  },
  
  // ตั้งค่า transformer
  transformer: {
    // ใช้ SVG transformer
    babelTransformerPath: require.resolve('react-native-svg-transformer'),
    
    // Enable TypeScript
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: true,
      },
    }),
  },
  
  // Server settings
  server: {
    port: 8081,
    enhanceMiddleware: (middleware) => {
      return middleware;
    },
  },
  
  // Watcher settings
  watchFolders: [
    // เพิ่ม folder ที่ต้องการ watch
    // path.resolve(__dirname, '../shared'),
  ],
  
  // Cache settings
  cacheVersion: '1.0',
  
  // Max workers
  maxWorkers: 4,
};

module.exports = mergeConfig(defaultConfig, config);
```

### Metro เพิ่ม SVG Support

```bash
# ติดตั้ง packages
npm install react-native-svg
npm install react-native-svg-transformer --save-dev
```

```javascript
// metro.config.js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const defaultConfig = getDefaultConfig(__dirname);
const { assetExts, sourceExts } = defaultConfig.resolver;

const config = {
  transformer: {
    babelTransformerPath: require.resolve('react-native-svg-transformer'),
  },
  resolver: {
    assetExts: assetExts.filter(ext => ext !== 'svg'),
    sourceExts: [...sourceExts, 'svg'],
  },
};

module.exports = mergeConfig(defaultConfig, config);
```

```tsx
// ใช้งาน SVG
import Logo from './assets/logo.svg';

const App = () => (
  <Logo width={200} height={100} fill="#007AFF" />
);
```

---

## babel.config.js

Babel แปลง JavaScript ใหม่ให้ทำงานได้กับทุก environment

```javascript
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  
  plugins: [
    // Plugin สำหรับ module aliases
    [
      'module-resolver',
      {
        root: ['./src'],
        extensions: ['.ios.js', '.android.js', '.js', '.ts', '.tsx', '.json'],
        alias: {
          '@': './src',
          '@components': './src/components',
          '@screens': './src/screens',
          '@hooks': './src/hooks',
          '@services': './src/services',
          '@store': './src/store',
          '@types': './src/types',
          '@utils': './src/utils',
          '@assets': './src/assets',
        },
      },
    ],
    
    // Plugin สำหรับ Reanimated (ถ้าใช้)
    'react-native-reanimated/plugin',
    
    // Plugin สำหรับ optional chaining (อาจมีแล้วใน preset)
    '@babel/plugin-proposal-optional-chaining',
    
    // Plugin สำหรับ nullish coalescing
    '@babel/plugin-proposal-nullish-coalescing-operator',
  ],
};
```

**ติดตั้ง module-resolver:**

```bash
npm install --save-dev babel-plugin-module-resolver

# แก้ tsconfig.json เพื่อให้ TypeScript รู้จัก aliases
```

```json
// tsconfig.json - เพิ่ม paths
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@screens/*": ["src/screens/*"]
    }
  }
}
```

---

## tsconfig.json

TypeScript Configuration

```json
{
  "extends": "@react-native/typescript-config/tsconfig.json",
  "compilerOptions": {
    // Target
    "target": "esnext",
    "module": "esnext",
    "lib": ["esnext"],
    
    // Module resolution
    "moduleResolution": "bundler",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@screens/*": ["src/screens/*"],
      "@hooks/*": ["src/hooks/*"],
      "@services/*": ["src/services/*"],
      "@store/*": ["src/store/*"],
      "@types/*": ["src/types/*"],
      "@utils/*": ["src/utils/*"],
      "@assets/*": ["src/assets/*"]
    },
    
    // Strict type checking
    "strict": true,
    "strictNullChecks": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    
    // JSX
    "jsx": "react-native",
    
    // Other
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true
  },
  "include": [
    "src",
    "App.tsx",
    "index.js",
    "metro.config.js",
    "babel.config.js"
  ],
  "exclude": [
    "node_modules",
    "android",
    "ios",
    "__tests__"
  ]
}
```

---

## android/ folder

### โครงสร้าง Android

```
android/
├── app/
│   ├── src/
│   │   ├── debug/
│   │   │   └── AndroidManifest.xml   # Debug-specific settings
│   │   └── main/
│   │       ├── AndroidManifest.xml   # App permissions, activities
│   │       ├── java/com/myapp/
│   │       │   ├── MainActivity.kt   # Main activity
│   │       │   └── MainApplication.kt # Application class
│   │       └── res/
│   │           ├── drawable/         # XML drawables
│   │           ├── drawable-night/   # Dark mode drawables
│   │           ├── mipmap-hdpi/      # App icons 72x72
│   │           ├── mipmap-mdpi/      # App icons 48x48
│   │           ├── mipmap-xhdpi/     # App icons 96x96
│   │           ├── mipmap-xxhdpi/    # App icons 144x144
│   │           ├── mipmap-xxxhdpi/   # App icons 192x192
│   │           └── values/
│   │               ├── strings.xml   # String resources
│   │               └── styles.xml    # Style resources
│   └── build.gradle                  # App-level build config
│
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties  # Gradle version
│
├── build.gradle                       # Project-level build config
├── gradle.properties                  # Gradle properties
└── settings.gradle                    # Project settings
```

### AndroidManifest.xml

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

  <!-- Permissions ที่ต้องขอ -->
  <uses-permission android:name="android.permission.INTERNET" />
  <uses-permission android:name="android.permission.CAMERA" />
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
  <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
  <uses-permission android:name="android.permission.VIBRATE" />
  <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

  <application
    android:name=".MainApplication"
    android:label="@string/app_name"
    android:icon="@mipmap/ic_launcher"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:allowBackup="false"
    android:theme="@style/AppTheme"
    android:usesCleartextTraffic="true">  <!-- อนุญาต HTTP (dev) -->
    
    <activity
      android:name=".MainActivity"
      android:label="@string/app_name"
      android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|screenSize|smallestScreenSize|uiMode"
      android:launchMode="singleTask"
      android:windowSoftInputMode="adjustResize"
      android:exported="true">
      
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
      
      <!-- Deep links -->
      <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" android:host="*" />
      </intent-filter>
    </activity>
  </application>
</manifest>
```

### build.gradle (app level)

```groovy
// android/app/build.gradle
apply plugin: "com.android.application"
apply plugin: "org.jetbrains.kotlin.android"
apply plugin: "com.facebook.react"

android {
  ndkVersion rootProject.ext.ndkVersion
  buildToolsVersion rootProject.ext.buildToolsVersion
  compileSdk rootProject.ext.compileSdkVersion

  namespace "com.myapp"
  
  defaultConfig {
    applicationId "com.myapp"        // Package name (ต้อง unique)
    minSdkVersion rootProject.ext.minSdkVersion
    targetSdkVersion rootProject.ext.targetSdkVersion
    versionCode 1                    // Build number (ต้องเพิ่มทุก release)
    versionName "1.0"               // Version string
  }

  signingConfigs {
    debug {
      storeFile file('debug.keystore')
      storePassword 'android'
      keyAlias 'androiddebugkey'
      keyPassword 'android'
    }
    // release config จะเพิ่มทีหลัง
  }

  buildTypes {
    debug {
      signingConfig signingConfigs.debug
    }
    release {
      minifyEnabled enableProguardInReleaseBuilds
      proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
      signingConfig signingConfigs.release
    }
  }
}

dependencies {
  implementation("com.facebook.react:react-android")
  implementation("com.facebook.react:hermes-android")
}

apply from: file("../../node_modules/@react-native-community/cli-platform-android/native_modules.gradle")
applyNativeModulesAppBuildGradle(project)
```

### gradle.properties

```properties
# android/gradle.properties

# Android SDK versions
android.useAndroidX=true
android.enableJetifier=true

# React Native
reactNativeArchitectures=armeabi-v7a,arm64-v8a,x86,x86_64
newArchEnabled=true
hermesEnabled=true

# Performance
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m
org.gradle.daemon=true
org.gradle.parallel=true
org.gradle.configureondemand=true
org.gradle.caching=true
```

---

## ios/ folder

### โครงสร้าง iOS

```
ios/
├── MyApp/
│   ├── AppDelegate.swift          # App entry point
│   ├── Info.plist                 # App settings/permissions
│   ├── LaunchScreen.storyboard    # Launch screen UI
│   ├── Images.xcassets/           # App icons & images
│   │   ├── AppIcon.appiconset/
│   │   └── Contents.json
│   └── main.m                     # (legacy) C entry point
│
├── MyApp.xcodeproj/
│   └── project.pbxproj            # Xcode project file
│
├── MyApp.xcworkspace/             # Workspace (ใช้ CocoaPods)
│   └── contents.xcworkspacedata
│
├── Podfile                        # CocoaPods dependencies
└── Podfile.lock                   # Lock file สำหรับ pods
```

### Info.plist

```xml
<!-- ios/MyApp/Info.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
<plist version="1.0">
<dict>
  <!-- App Information -->
  <key>CFBundleIdentifier</key>
  <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
  
  <key>CFBundleDisplayName</key>
  <string>My App</string>
  
  <key>CFBundleShortVersionString</key>
  <string>1.0</string>
  
  <key>CFBundleVersion</key>
  <string>1</string>
  
  <!-- Network Settings -->
  <key>NSAppTransportSecurity</key>
  <dict>
    <!-- อนุญาต HTTP ในการพัฒนา (ลบใน production) -->
    <key>NSAllowsArbitraryLoads</key>
    <true/>
  </dict>
  
  <!-- Permissions -->
  <key>NSCameraUsageDescription</key>
  <string>แอปต้องการเข้าถึงกล้องเพื่อถ่ายภาพ</string>
  
  <key>NSPhotoLibraryUsageDescription</key>
  <string>แอปต้องการเข้าถึง Photo Library</string>
  
  <key>NSLocationWhenInUseUsageDescription</key>
  <string>แอปต้องการตำแหน่งเพื่อแสดงข้อมูลใกล้เคียง</string>
  
  <key>NSMicrophoneUsageDescription</key>
  <string>แอปต้องการเข้าถึงไมโครโฟน</string>
  
  <!-- URL Schemes (Deep Links) -->
  <key>CFBundleURLTypes</key>
  <array>
    <dict>
      <key>CFBundleURLSchemes</key>
      <array>
        <string>myapp</string>
      </array>
    </dict>
  </array>
  
  <!-- Supported Orientations -->
  <key>UISupportedInterfaceOrientations</key>
  <array>
    <string>UIInterfaceOrientationPortrait</string>
    <!-- Uncomment for landscape support -->
    <!-- <string>UIInterfaceOrientationLandscapeLeft</string> -->
    <!-- <string>UIInterfaceOrientationLandscapeRight</string> -->
  </array>
</dict>
</plist>
```

### Podfile

```ruby
# ios/Podfile
require_relative '../node_modules/react-native/scripts/react_native_pods'
require_relative '../node_modules/@react-native-community/cli-platform-ios/native_modules'

platform :ios, min_ios_version_supported
prepare_react_native_project!

linkage = ENV['USE_FRAMEWORKS']
if linkage != nil
  Pod::UI.puts "Configuring Pod with #{linkage}ally linked Frameworks".green
  use_frameworks! :linkage => linkage.to_sym
end

target 'MyApp' do
  config = use_native_modules!

  use_react_native!(
    :path => config[:reactNativePath],
    :app_path => "#{Pod::Config.instance.installation_root}/.."
  )

  # เพิ่ม pods ที่ต้องการ
  # pod 'react-native-camera', :path => '../node_modules/react-native-camera'

  post_install do |installer|
    react_native_post_install(installer)
    
    # แก้ปัญหา build settings
    installer.pods_project.targets.each do |target|
      target.build_configurations.each do |config|
        config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '13.4'
      end
    end
  end
end
```

---

## src/ Structure Best Practices

### โครงสร้างแนะนำ

```
src/
├── assets/                       # Static files
│   ├── fonts/                    # Custom fonts (.ttf, .otf)
│   │   ├── Prompt-Regular.ttf
│   │   ├── Prompt-Bold.ttf
│   │   └── Prompt-Medium.ttf
│   ├── images/                   # Images (.png, .jpg, .webp)
│   │   ├── logo.png
│   │   ├── placeholder.png
│   │   └── icons/
│   └── animations/               # Lottie files (.json)
│       ├── loading.json
│       └── success.json
│
├── components/                   # Reusable components
│   ├── common/                   # Shared across whole app
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.styles.ts
│   │   │   ├── Button.test.tsx
│   │   │   └── index.ts         # Re-export
│   │   ├── Card/
│   │   ├── Input/
│   │   ├── LoadingSpinner/
│   │   ├── Modal/
│   │   └── Typography/
│   └── index.ts                  # Export all common components
│
├── hooks/                        # Custom React Hooks
│   ├── useAuth.ts
│   ├── useTheme.ts
│   ├── useLocation.ts
│   ├── useNotification.ts
│   └── index.ts
│
├── navigation/                   # React Navigation
│   ├── RootNavigator.tsx         # Root navigator
│   ├── AuthNavigator.tsx         # Auth flow
│   ├── MainNavigator.tsx         # Main app flow
│   ├── types.ts                  # Navigation types
│   └── index.ts
│
├── screens/                      # Screen components
│   ├── Auth/
│   │   ├── LoginScreen.tsx
│   │   ├── RegisterScreen.tsx
│   │   └── ForgotPasswordScreen.tsx
│   ├── Home/
│   │   ├── HomeScreen.tsx
│   │   └── components/          # Screen-specific components
│   │       └── ProductCard.tsx
│   ├── Profile/
│   │   └── ProfileScreen.tsx
│   └── Settings/
│       └── SettingsScreen.tsx
│
├── services/                     # API calls, external services
│   ├── api/
│   │   ├── client.ts            # Axios instance
│   │   ├── auth.ts              # Auth endpoints
│   │   ├── products.ts          # Product endpoints
│   │   └── index.ts
│   ├── firebase/
│   │   ├── auth.ts
│   │   ├── firestore.ts
│   │   └── index.ts
│   └── storage.ts               # AsyncStorage/MMKV
│
├── store/                        # State management
│   ├── index.ts                 # Store setup
│   ├── slices/                  # Redux Toolkit slices
│   │   ├── authSlice.ts
│   │   ├── cartSlice.ts
│   │   └── uiSlice.ts
│   └── hooks.ts                 # Typed hooks
│
├── theme/                        # Design system
│   ├── colors.ts                # Color palette
│   ├── typography.ts            # Font styles
│   ├── spacing.ts               # Spacing scale
│   ├── shadows.ts               # Shadow styles
│   └── index.ts
│
├── types/                        # TypeScript types
│   ├── api.ts                   # API response types
│   ├── navigation.ts            # Navigation types
│   ├── user.ts                  # User types
│   └── index.ts
│
└── utils/                        # Helper functions
    ├── date.ts                  # Date formatting
    ├── currency.ts              # Currency formatting
    ├── validation.ts            # Form validation
    ├── storage.ts               # Storage helpers
    └── index.ts
```

### ตัวอย่าง: Theme System

```typescript
// src/theme/colors.ts
export const colors = {
  // Brand colors
  primary: '#007AFF',
  primaryDark: '#0056B3',
  primaryLight: '#4DA6FF',
  
  secondary: '#FF9500',
  secondaryDark: '#CC7700',
  secondaryLight: '#FFB347',
  
  // Status colors
  success: '#34C759',
  warning: '#FF9500',
  error: '#FF3B30',
  info: '#007AFF',
  
  // Neutral colors
  white: '#FFFFFF',
  black: '#000000',
  
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
  
  // Dark mode
  background: '#FFFFFF',
  backgroundDark: '#000000',
  surface: '#F9FAFB',
  surfaceDark: '#1C1C1E',
  text: '#111827',
  textDark: '#FFFFFF',
  textSecondary: '#6B7280',
  textSecondaryDark: '#EBEBF5',
};

export type ColorKeys = keyof typeof colors;
```

```typescript
// src/theme/spacing.ts
export const spacing = {
  xs: 4,
  sm: 8,
  md: 12,
  base: 16,
  lg: 20,
  xl: 24,
  '2xl': 32,
  '3xl': 40,
  '4xl': 48,
  '5xl': 64,
};
```

```typescript
// src/theme/typography.ts
import { StyleSheet } from 'react-native';

export const typography = StyleSheet.create({
  h1: {
    fontSize: 32,
    fontWeight: '700',
    lineHeight: 40,
  },
  h2: {
    fontSize: 28,
    fontWeight: '700',
    lineHeight: 36,
  },
  h3: {
    fontSize: 24,
    fontWeight: '600',
    lineHeight: 32,
  },
  h4: {
    fontSize: 20,
    fontWeight: '600',
    lineHeight: 28,
  },
  body1: {
    fontSize: 16,
    fontWeight: '400',
    lineHeight: 24,
  },
  body2: {
    fontSize: 14,
    fontWeight: '400',
    lineHeight: 20,
  },
  caption: {
    fontSize: 12,
    fontWeight: '400',
    lineHeight: 16,
  },
  button: {
    fontSize: 16,
    fontWeight: '600',
    lineHeight: 24,
  },
});
```

### ตัวอย่าง: Common Button Component

```tsx
// src/components/common/Button/Button.tsx
import React from 'react';
import {
  TouchableOpacity,
  Text,
  ActivityIndicator,
  StyleSheet,
  ViewStyle,
  TextStyle,
} from 'react-native';
import { colors } from '../../../theme/colors';
import { spacing } from '../../../theme/spacing';

type ButtonVariant = 'primary' | 'secondary' | 'outline' | 'ghost';
type ButtonSize = 'sm' | 'md' | 'lg';

interface ButtonProps {
  label: string;
  onPress: () => void;
  variant?: ButtonVariant;
  size?: ButtonSize;
  disabled?: boolean;
  loading?: boolean;
  fullWidth?: boolean;
  style?: ViewStyle;
  textStyle?: TextStyle;
}

const Button: React.FC<ButtonProps> = ({
  label,
  onPress,
  variant = 'primary',
  size = 'md',
  disabled = false,
  loading = false,
  fullWidth = false,
  style,
  textStyle,
}) => {
  const buttonStyle = [
    styles.base,
    styles[variant],
    styles[size],
    fullWidth && styles.fullWidth,
    (disabled || loading) && styles.disabled,
    style,
  ];

  const labelStyle = [
    styles.label,
    styles[`${variant}Label` as keyof typeof styles],
    styles[`${size}Label` as keyof typeof styles],
    textStyle,
  ];

  return (
    <TouchableOpacity
      style={buttonStyle}
      onPress={onPress}
      disabled={disabled || loading}
      activeOpacity={0.8}
    >
      {loading ? (
        <ActivityIndicator
          color={variant === 'primary' ? '#fff' : colors.primary}
          size="small"
        />
      ) : (
        <Text style={labelStyle}>{label}</Text>
      )}
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  base: {
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
    flexDirection: 'row',
  },
  
  // Variants
  primary: {
    backgroundColor: colors.primary,
  },
  secondary: {
    backgroundColor: colors.secondary,
  },
  outline: {
    backgroundColor: 'transparent',
    borderWidth: 1.5,
    borderColor: colors.primary,
  },
  ghost: {
    backgroundColor: 'transparent',
  },
  
  // Sizes
  sm: {
    paddingHorizontal: spacing.md,
    paddingVertical: spacing.xs,
  },
  md: {
    paddingHorizontal: spacing.base,
    paddingVertical: spacing.sm,
  },
  lg: {
    paddingHorizontal: spacing.xl,
    paddingVertical: spacing.md,
  },
  
  // Full Width
  fullWidth: {
    width: '100%',
  },
  
  // Disabled
  disabled: {
    opacity: 0.5,
  },
  
  // Labels
  label: {
    fontWeight: '600',
  },
  primaryLabel: {
    color: '#fff',
  },
  secondaryLabel: {
    color: '#fff',
  },
  outlineLabel: {
    color: colors.primary,
  },
  ghostLabel: {
    color: colors.primary,
  },
  smLabel: {
    fontSize: 14,
  },
  mdLabel: {
    fontSize: 16,
  },
  lgLabel: {
    fontSize: 18,
  },
});

export default Button;
```

```typescript
// src/components/common/Button/index.ts
export { default } from './Button';
export type { ButtonProps } from './Button';
```

---

## Workshop

### Workshop 4.1: จัดโครงสร้าง Project ที่มีอยู่

จัดโครงสร้าง HelloWorldApp ให้ถูกต้อง:

```bash
cd HelloWorldApp

# สร้าง src structure
mkdir -p src/assets/{fonts,images,animations}
mkdir -p src/components/{common,screens}
mkdir -p src/hooks
mkdir -p src/navigation
mkdir -p src/screens/{Auth,Home,Profile}
mkdir -p src/services/api
mkdir -p src/store/slices
mkdir -p src/theme
mkdir -p src/types
mkdir -p src/utils

# สร้างไฟล์ index.ts สำหรับแต่ละ folder
touch src/components/index.ts
touch src/hooks/index.ts
touch src/theme/index.ts
touch src/types/index.ts
touch src/utils/index.ts
```

### Workshop 4.2: สร้าง Theme System

สร้าง Theme ให้กับ project:

```typescript
// src/theme/index.ts
export * from './colors';
export * from './spacing';
export * from './typography';
export * from './shadows';
```

```typescript
// src/theme/shadows.ts
import { Platform } from 'react-native';

export const shadows = {
  sm: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 1 },
      shadowOpacity: 0.05,
      shadowRadius: 2,
    },
    android: {
      elevation: 2,
    },
  }),
  md: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.1,
      shadowRadius: 4,
    },
    android: {
      elevation: 4,
    },
  }),
  lg: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 4 },
      shadowOpacity: 0.15,
      shadowRadius: 8,
    },
    android: {
      elevation: 8,
    },
  }),
};
```

### Workshop 4.3: Path Aliases Setup

ตั้งค่า Path Aliases เพื่อ import สะดวกขึ้น:

```bash
npm install --save-dev babel-plugin-module-resolver
```

```javascript
// babel.config.js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    [
      'module-resolver',
      {
        root: ['./src'],
        alias: {
          '@components': './src/components',
          '@screens': './src/screens',
          '@hooks': './src/hooks',
          '@theme': './src/theme',
          '@utils': './src/utils',
          '@assets': './src/assets',
          '@navigation': './src/navigation',
          '@services': './src/services',
          '@store': './src/store',
          '@types': './src/types',
        },
      },
    ],
  ],
};
```

```json
// tsconfig.json - เพิ่ม paths
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@components/*": ["src/components/*"],
      "@screens/*": ["src/screens/*"],
      "@hooks/*": ["src/hooks/*"],
      "@theme/*": ["src/theme/*"],
      "@utils/*": ["src/utils/*"],
      "@assets/*": ["src/assets/*"],
      "@navigation/*": ["src/navigation/*"],
      "@services/*": ["src/services/*"],
      "@store/*": ["src/store/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

ทดสอบ import:

```tsx
// App.tsx - ใช้ path aliases
import Button from '@components/common/Button';
import { colors } from '@theme/colors';
import { spacing } from '@theme/spacing';

// แทนที่จะเป็น
import Button from './src/components/common/Button';
import { colors } from './src/theme/colors';
```

### Workshop 4.4: แบบฝึกหัด

1. สร้าง `src/theme/colors.ts` พร้อม color palette ที่สมบูรณ์
2. สร้าง `src/components/common/Card/Card.tsx` component
3. สร้าง `src/utils/date.ts` พร้อม functions:
   - `formatDate(date: Date): string` - แสดงวันที่ภาษาไทย
   - `timeAgo(date: Date): string` - "5 นาทีที่แล้ว"
4. เขียน unit test สำหรับ date utils

```typescript
// src/utils/date.ts ตัวอย่าง
export const formatDate = (date: Date): string => {
  const options: Intl.DateTimeFormatOptions = {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    locale: 'th-TH',
  };
  return new Intl.DateTimeFormat('th-TH', options).format(date);
};

export const timeAgo = (date: Date): string => {
  const now = new Date();
  const diffMs = now.getTime() - date.getTime();
  const diffSec = Math.floor(diffMs / 1000);
  const diffMin = Math.floor(diffSec / 60);
  const diffHour = Math.floor(diffMin / 60);
  const diffDay = Math.floor(diffHour / 24);

  if (diffSec < 60) return 'เมื่อกี้';
  if (diffMin < 60) return `${diffMin} นาทีที่แล้ว`;
  if (diffHour < 24) return `${diffHour} ชั่วโมงที่แล้ว`;
  if (diffDay < 7) return `${diffDay} วันที่แล้ว`;
  return formatDate(date);
};
```

---

## Tips และ Best Practices

### 1. Barrel Exports (index.ts)

```typescript
// src/components/common/index.ts
export { default as Button } from './Button';
export { default as Card } from './Card';
export { default as Input } from './Input';
export { default as LoadingSpinner } from './LoadingSpinner';

// การใช้งาน
import { Button, Card, Input } from '@components/common';
// แทนที่จะ
import Button from '@components/common/Button';
import Card from '@components/common/Card';
```

### 2. Feature-based Structure (ทางเลือก)

สำหรับแอปที่ใหญ่มาก อาจใช้โครงสร้างแบบ Feature:

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── screens/
│   │   ├── services/
│   │   ├── store/
│   │   └── types/
│   ├── cart/
│   └── products/
└── shared/
    ├── components/
    ├── hooks/
    └── utils/
```

### 3. ตั้งชื่อไฟล์ให้สม่ำเสมอ

```
PascalCase สำหรับ Components:
Button.tsx, ProductCard.tsx, UserProfile.tsx

camelCase สำหรับ Utilities/Hooks:
useAuth.ts, formatDate.ts, apiClient.ts

SCREAMING_SNAKE สำหรับ Constants:
API_URL.ts, APP_CONSTANTS.ts
```

---

## สรุป Part 004

### ได้เรียนรู้

1. **โครงสร้างไฟล์** - ทุก folder และ file มีความหมาย
2. **package.json** - dependencies, scripts, versioning
3. **metro.config.js** - bundler configuration, aliases
4. **babel.config.js** - transpiler configuration
5. **tsconfig.json** - TypeScript settings
6. **android/** - native Android code
7. **ios/** - native iOS code
8. **src/ structure** - best practices สำหรับจัดโค้ด

---

**ต่อไป → Part 005: JSX และ Components เบื้องต้น**
