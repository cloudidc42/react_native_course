# Part 066: App Distribution ใน React Native

## บทนำ

App Distribution คือกระบวนการส่งมอบแอปให้กับ testers หรือผู้ใช้งาน ก่อน release สู่ public สำหรับ React Native มีหลายวิธีในการ distribute แอป ทั้ง TestFlight สำหรับ iOS, Google Play Internal Testing สำหรับ Android, และ third-party services เช่น Firebase App Distribution และ AppCenter

---

## 1. TestFlight (iOS)

### TestFlight คืออะไร?

TestFlight เป็น Apple's official beta testing platform สำหรับ iOS apps ช่วยให้นักพัฒนาส่ง beta builds ให้ผู้ทดสอบได้สูงสุด 10,000 คน

### ขั้นตอนการ Setup TestFlight

#### 1. สร้าง App ID ใน Apple Developer Portal

```
1. เข้า https://developer.apple.com
2. ไปที่ Certificates, Identifiers & Profiles
3. คลิก Identifiers → App IDs → เพิ่ม +
4. เลือก App
5. กรอก Description และ Bundle ID (เช่น com.yourcompany.yourapp)
6. เลือก capabilities ที่ต้องการ
7. คลิก Continue → Register
```

#### 2. สร้าง App ใน App Store Connect

```
1. เข้า https://appstoreconnect.apple.com
2. ไปที่ My Apps
3. คลิก + → New App
4. กรอก Platform: iOS
5. กรอก Name, Bundle ID, SKU
6. คลิก Create
```

#### 3. ตั้งค่า Signing ใน Xcode

```
1. เปิด Xcode → เปิด .xcworkspace
2. เลือก Target → Signing & Capabilities
3. ตั้งค่า Team, Bundle Identifier
4. ตรวจสอบ Automatically manage signing
```

### Upload App ไปยัง TestFlight ด้วย Xcode

```
1. เปลือก Product → Archive
2. รอ Archive เสร็จ
3. ใน Organizer → คลิก Distribute App
4. เลือก App Store Connect
5. เลือก Upload
6. กด Next จนถึง Upload
7. รอ Processing (ประมาณ 5-30 นาที)
```

### ใช้ Fastlane Upload ไปยัง TestFlight

```ruby
# ios/fastlane/Fastfile
lane :beta do
  # Update version number
  increment_version_number(
    version_number: "1.0.0"
  )
  
  # Update build number
  increment_build_number(
    build_number: latest_testflight_build_number + 1
  )
  
  # Build the app
  build_app(
    scheme: "YourApp",
    workspace: "YourApp.xcworkspace",
    export_method: "app-store"
  )
  
  # Upload to TestFlight
  upload_to_testflight(
    skip_waiting_for_build_processing: false,
    changelog: changelog_from_git_commits(
      merge_commit_filtering: "include_merges"
    )
  )
  
  # Send notification
  slack(
    message: "New TestFlight build uploaded! 🚀",
    channel: "#ios-releases",
    webhook_url: ENV["SLACK_WEBHOOK"]
  )
end
```

### จัดการ Testers ใน TestFlight

```
Internal Testers (สูงสุด 100 คน):
- ต้องมี Apple Developer account
- ได้รับ access ทันที ไม่ต้อง review
- ทดสอบได้ทุก build

External Testers (สูงสุด 10,000 คน):
- ใช้ email invite
- ต้องผ่าน Beta App Review ก่อน
- เพิ่ม groups เพื่อจัดการ
```

### ตั้งค่า TestFlight Groups ด้วย Fastlane

```ruby
lane :add_tester do
  pilot(
    add_bulk_testers: true,
    testers_file_path: "testers.csv",
    groups: ["Internal QA", "Beta Users"]
  )
end

# testers.csv format:
# First name,Last name,Email
# John,Doe,john@example.com
# Jane,Smith,jane@example.com
```

---

## 2. Google Play Console (Internal Testing)

### ขั้นตอนการ Setup Play Console

#### 1. สร้าง App ใน Google Play Console

```
1. เข้า https://play.google.com/console
2. คลิก Create app
3. กรอก App name, Default language, Type
4. ยอมรับ Developer Program Policies
5. คลิก Create app
```

#### 2. ตั้งค่า Internal Testing Track

```
1. ใน App Dashboard → Testing → Internal testing
2. คลิก Create new release
3. Upload AAB file
4. กรอก Release notes
5. คลิก Save → Review release → Start rollout
```

#### 3. เพิ่ม Testers

```
1. Internal testing → Testers tab
2. คลิก Create email list
3. กรอก Email addresses
4. คลิก Save changes
5. Copy opt-in URL ส่งให้ testers
```

### Upload ด้วย Fastlane

```ruby
# android/fastlane/Fastfile
lane :internal do
  gradle(
    task: 'bundle',
    build_type: 'Release',
    project_dir: './'
  )
  
  upload_to_play_store(
    track: 'internal',
    aab: 'app/build/outputs/bundle/release/app-release.aab',
    json_key: 'play-store-key.json',
    release_status: 'completed',
    rollout: '1.0'
  )
  
  slack(
    message: "Android internal build uploaded! 🤖",
    webhook_url: ENV["SLACK_WEBHOOK"]
  )
end

lane :promote_to_beta do
  upload_to_play_store(
    track: 'internal',
    track_promote_to: 'beta',
    json_key: 'play-store-key.json'
  )
end

lane :promote_to_production do
  upload_to_play_store(
    track: 'beta',
    track_promote_to: 'production',
    json_key: 'play-store-key.json',
    rollout: '0.1'  # 10% rollout เริ่มต้น
  )
end
```

---

## 3. Firebase App Distribution

### ทำไมต้องใช้ Firebase App Distribution?

```
- รองรับทั้ง iOS และ Android
- ง่ายต่อการ invite testers
- ไม่ต้องผ่าน store review process
- Integration กับ Firebase ecosystem
- Tester feedback ใน app
```

### ตั้งค่า Firebase App Distribution

```bash
# ติดตั้ง Firebase CLI
npm install -g firebase-tools

# Login
firebase login

# รัน Firebase init
firebase init appdistribution
```

### Upload ด้วย Firebase CLI

```bash
# iOS
firebase appdistribution:distribute ios/build/YourApp.ipa \
  --app "1:123456789:ios:abcdef" \
  --groups "internal-testers,qa-team" \
  --release-notes "Bug fixes and improvements"

# Android
firebase appdistribution:distribute android/app/build/outputs/apk/release/app-release.apk \
  --app "1:123456789:android:abcdef" \
  --groups "internal-testers" \
  --release-notes "New features in this build"
```

### ใช้ Fastlane Plugin

```bash
# ติดตั้ง plugin
fastlane add_plugin firebase_app_distribution
```

```ruby
# Fastfile
lane :distribute do
  # Build app
  build_app(scheme: "YourApp")
  
  # Distribute via Firebase
  firebase_app_distribution(
    app: "1:123456789:ios:abcdef",
    groups: "internal-testers",
    release_notes: "Build #{Time.now.strftime('%Y%m%d%H%M')}",
    firebase_cli_path: "/usr/local/bin/firebase"
  )
end
```

### Firebase App Distribution ใน GitHub Actions

```yaml
# .github/workflows/distribute.yml
name: Distribute to Testers

on:
  push:
    branches: [develop]

jobs:
  distribute-ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup
        run: npm ci
      
      - name: Build iOS
        run: cd ios && bundle exec fastlane build_app
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
      
      - name: Distribute to Firebase
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_IOS_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          groups: internal-testers
          file: ios/output/YourApp.ipa
          releaseNotes: |
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
            ${{ github.event.head_commit.message }}

  distribute-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup
        run: npm ci
      
      - name: Build Android
        run: cd android && ./gradlew assembleRelease
      
      - name: Distribute to Firebase
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_ANDROID_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          groups: internal-testers
          file: android/app/build/outputs/apk/release/app-release.apk
          releaseNotes: |
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
```

### Firebase App Distribution SDK (In-App Tester UX)

```bash
# ติดตั้ง SDK
npm install @react-native-firebase/app @react-native-firebase/app-distribution
cd ios && pod install
```

```typescript
// App.tsx - สำหรับ non-production builds เท่านั้น
import React, { useEffect } from 'react';
import appdistribution from '@react-native-firebase/app-distribution';

const App: React.FC = () => {
  useEffect(() => {
    if (__DEV__ || process.env.ENVIRONMENT !== 'production') {
      checkForUpdates();
    }
  }, []);

  const checkForUpdates = async () => {
    try {
      const isNewBuildAvailable = await appdistribution().isTesterSignedIn();
      if (isNewBuildAvailable) {
        // แจ้ง tester เมื่อมี build ใหม่
        appdistribution().onUpdateAvailable(() => {
          // Handle update notification
        });
      }
    } catch (error) {
      console.log('App Distribution check failed:', error);
    }
  };

  return (
    // app content
    <></>
  );
};
```

---

## 4. AppCenter (Microsoft)

### ตั้งค่า AppCenter

```bash
# ติดตั้ง AppCenter CLI
npm install -g appcenter-cli

# Login
appcenter login
```

### Upload ด้วย AppCenter CLI

```bash
# Upload iOS
appcenter distribute release \
  -a YourOrg/YourApp-iOS \
  -f ios/build/YourApp.ipa \
  -g "Internal Testers" \
  --release-notes "Version 1.0.0 - New features"

# Upload Android
appcenter distribute release \
  -a YourOrg/YourApp-Android \
  -f android/app/build/outputs/apk/release/app-release.apk \
  -g "QA Team" \
  --release-notes "Build for QA review"
```

### AppCenter SDK

```bash
npm install appcenter appcenter-analytics appcenter-crashes
cd ios && pod install
```

```typescript
// index.js หรือ App.tsx
import AppCenter from 'appcenter';
import Analytics from 'appcenter-analytics';
import Crashes from 'appcenter-crashes';

// เปิด AppCenter
AppCenter.setLogLevel(AppCenter.LogLevel.VERBOSE);
Analytics.setEnabled(true);
Crashes.setEnabled(true);
```

---

## 5. Changelog Management

### สร้าง Changelog อัตโนมัติ

**scripts/generateChangelog.js**:
```javascript
const { execSync } = require('child_process');
const fs = require('fs');

function generateChangelog() {
  // ดึง commits ตั้งแต่ last tag
  const lastTag = execSync('git describe --tags --abbrev=0').toString().trim();
  const commits = execSync(`git log ${lastTag}..HEAD --pretty=format:"%h|%s|%an"`).toString();
  
  const lines = commits.split('\n').filter(line => line.trim());
  
  const categories = {
    feat: [],
    fix: [],
    docs: [],
    style: [],
    refactor: [],
    perf: [],
    test: [],
    chore: [],
    other: [],
  };
  
  lines.forEach(line => {
    const [hash, subject, author] = line.split('|');
    const match = subject.match(/^(\w+)(\(.+?\))?:\s*(.+)$/);
    
    if (match) {
      const [, type, scope, message] = match;
      if (categories[type]) {
        categories[type].push({ hash, message, scope, author });
      } else {
        categories.other.push({ hash, message: subject, scope: null, author });
      }
    } else {
      categories.other.push({ hash, message: subject, scope: null, author });
    }
  });
  
  let changelog = `# Changelog\n\n## [Unreleased] - ${new Date().toISOString().split('T')[0]}\n\n`;
  
  const categoryNames = {
    feat: '### New Features',
    fix: '### Bug Fixes',
    perf: '### Performance',
    refactor: '### Refactoring',
    docs: '### Documentation',
    test: '### Tests',
    chore: '### Chores',
    other: '### Other Changes',
  };
  
  Object.entries(categoryNames).forEach(([key, title]) => {
    if (categories[key].length > 0) {
      changelog += `${title}\n`;
      categories[key].forEach(({ hash, message, scope }) => {
        const scopeStr = scope ? `**${scope.replace(/[()]/g, '')}**: ` : '';
        changelog += `- ${scopeStr}${message} ([${hash}])\n`;
      });
      changelog += '\n';
    }
  });
  
  return changelog;
}

const changelog = generateChangelog();
fs.writeFileSync('CHANGELOG.md', changelog);
console.log('Changelog generated!');
console.log(changelog);
```

### Conventional Commits

```bash
# ติดตั้ง commitizen
npm install --save-dev commitizen cz-conventional-changelog

# ตั้งค่าใน package.json
{
  "scripts": {
    "commit": "git-cz"
  },
  "config": {
    "commitizen": {
      "path": "./node_modules/cz-conventional-changelog"
    }
  }
}
```

```
# Format ของ Conventional Commits:
type(scope): description

# Examples:
feat(auth): add biometric login support
fix(profile): resolve avatar upload issue
docs(readme): update installation guide
style(button): fix button alignment on iOS
refactor(api): simplify error handling
perf(list): optimize FlatList rendering
test(auth): add unit tests for login hook
chore(deps): update React Native to 0.72
```

### Standard-version สำหรับ Versioning

```bash
npm install --save-dev standard-version

# ใน package.json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major"
  }
}

# รัน release
npm run release
```

---

## Workshop: Distribution Setup

### สร้าง Distribution Script

**scripts/distribute.sh**:
```bash
#!/bin/bash
set -e

# Configuration
ENVIRONMENT=${1:-"staging"}
PLATFORM=${2:-"both"}

echo "Starting distribution for $ENVIRONMENT on $PLATFORM"

# Build number จาก timestamp
BUILD_NUMBER=$(date +%Y%m%d%H%M)

# Generate release notes
RELEASE_NOTES=$(git log --oneline -10 --pretty=format:"- %s")

if [ "$PLATFORM" = "ios" ] || [ "$PLATFORM" = "both" ]; then
  echo "Building iOS..."
  
  cd ios
  
  # Update build number
  agvtool new-version -all $BUILD_NUMBER
  
  # Build
  bundle exec fastlane build_app
  
  if [ "$ENVIRONMENT" = "staging" ]; then
    # Upload to TestFlight
    bundle exec fastlane upload_testflight
    
    # Upload to Firebase App Distribution
    firebase appdistribution:distribute output/YourApp.ipa \
      --app "$FIREBASE_IOS_APP_ID" \
      --groups "internal-testers,qa-team" \
      --release-notes "$RELEASE_NOTES"
  else
    bundle exec fastlane deploy_production
  fi
  
  cd ..
fi

if [ "$PLATFORM" = "android" ] || [ "$PLATFORM" = "both" ]; then
  echo "Building Android..."
  
  cd android
  
  # Update version code
  sed -i "s/versionCode [0-9]*/versionCode $BUILD_NUMBER/" app/build.gradle
  
  # Build
  ./gradlew bundleRelease
  
  if [ "$ENVIRONMENT" = "staging" ]; then
    # Upload to Play Console
    bundle exec fastlane upload_internal
    
    # Upload to Firebase App Distribution
    firebase appdistribution:distribute app/build/outputs/apk/release/app-release.apk \
      --app "$FIREBASE_ANDROID_APP_ID" \
      --groups "internal-testers" \
      --release-notes "$RELEASE_NOTES"
  else
    bundle exec fastlane promote_to_production
  fi
  
  cd ..
fi

echo "Distribution completed for $ENVIRONMENT!"

# Send Slack notification
curl -X POST -H 'Content-type: application/json' \
  --data "{
    \"text\": \"🚀 New build distributed to $ENVIRONMENT!\",
    \"attachments\": [{
      \"color\": \"good\",
      \"fields\": [
        {\"title\": \"Platform\", \"value\": \"$PLATFORM\"},
        {\"title\": \"Build Number\", \"value\": \"$BUILD_NUMBER\"},
        {\"title\": \"Release Notes\", \"value\": \"$RELEASE_NOTES\"}
      ]
    }]
  }" \
  $SLACK_WEBHOOK
```

### Fastfile ที่รองรับหลาย Environment

```ruby
# fastlane/Fastfile
default_platform(:ios)

ENVIRONMENTS = {
  development: {
    app_identifier: "com.yourapp.dev",
    firebase_app_id: ENV["FIREBASE_DEV_APP_ID"]
  },
  staging: {
    app_identifier: "com.yourapp.staging",
    firebase_app_id: ENV["FIREBASE_STAGING_APP_ID"]
  },
  production: {
    app_identifier: "com.yourapp",
    firebase_app_id: ENV["FIREBASE_PROD_APP_ID"]
  }
}

platform :ios do
  desc "Distribute to specified environment"
  lane :distribute do |options|
    env = (options[:env] || "staging").to_sym
    config = ENVIRONMENTS[env]
    
    raise "Invalid environment: #{options[:env]}" unless config
    
    increment_build_number
    
    match(
      type: env == :production ? "appstore" : "adhoc",
      app_identifier: config[:app_identifier]
    )
    
    build_app(
      configuration: env == :production ? "Release" : "Staging",
      export_method: env == :production ? "app-store" : "ad-hoc"
    )
    
    if env == :production
      upload_to_testflight
    else
      firebase_app_distribution(
        app: config[:firebase_app_id],
        groups: "internal-testers"
      )
    end
  end
end
```

---

## Tips และ Best Practices

### 1. Semantic Versioning

```
MAJOR.MINOR.PATCH
1.0.0 - Initial release
1.1.0 - New features
1.1.1 - Bug fix
2.0.0 - Breaking changes
```

### 2. Build Number Strategy

```
iOS Build Number: Year+Month+Day+Hour+Minute (yyyyMMddHHmm)
Android Version Code: Integer ที่ต้องเพิ่มขึ้นเรื่อยๆ

# Auto-generate
BUILD_NUMBER=$(date +%Y%m%d%H%M)
```

### 3. Testing Matrix

```
iOS:
- iPhone 13 mini (iOS 15)
- iPhone 14 (iOS 16)
- iPad Pro (iOS 16)

Android:
- Samsung Galaxy S21 (Android 12)
- Google Pixel 6 (Android 13)
- OnePlus 9 (Android 12)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **TestFlight**: การ distribute iOS apps สำหรับ beta testing
2. **Play Console**: Internal testing track สำหรับ Android
3. **Firebase App Distribution**: Cross-platform distribution
4. **AppCenter**: Microsoft's distribution platform
5. **Changelog Management**: จัดการ release notes อัตโนมัติ
6. **Distribution Automation**: Scripts และ Fastlane lanes
