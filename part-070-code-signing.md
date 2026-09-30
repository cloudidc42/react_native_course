# Part 070: Code Signing สำหรับ React Native

## บทนำ

Code Signing เป็นกระบวนการสำคัญในการ publish แอป iOS และ Android ที่ยืนยันตัวตนของนักพัฒนาและรับรองว่า code ไม่ถูกแก้ไขหลังจาก sign แล้ว ในบทนี้เราจะเรียนรู้การตั้งค่า code signing อย่างสมบูรณ์

---

## 1. ทำความเข้าใจ Code Signing

### ทำไม Code Signing ถึงสำคัญ?

```
Security:
- ป้องกัน code ถูกแก้ไขโดยคนอื่น
- ยืนยันว่า app มาจากนักพัฒนาที่ไว้วางใจ
- iOS ไม่อนุญาต install app ที่ไม่ signed

Distribution:
- จำเป็นสำหรับ App Store submission
- จำเป็นสำหรับ TestFlight
- จำเป็นสำหรับ Google Play

Trust:
- ผู้ใช้มั่นใจว่า app ปลอดภัย
- Apple/Google ตรวจสอบ developer identity
```

---

## 2. iOS Code Signing

### ส่วนประกอบของ iOS Code Signing

```
1. Certificate (ใบรับรอง):
   - Development Certificate: สำหรับ test บน device จริง
   - Distribution Certificate: สำหรับ App Store / Ad Hoc
   - เก็บใน Keychain บนเครื่อง Mac

2. App ID:
   - ระบุ app อย่างเฉพาะเจาะจง (Bundle Identifier)
   - เช่น com.yourcompany.yourapp

3. Provisioning Profile:
   - รวม Certificate + App ID + Devices
   - ประเภท: Development, Ad Hoc, App Store

4. Entitlements:
   - Capabilities ที่ app ใช้
   - Push Notifications, Sign in with Apple, etc.
```

### สร้าง iOS Distribution Certificate

```bash
# สร้าง Certificate Signing Request (CSR)
# 1. เปิด Keychain Access บน Mac
# 2. Certificate Assistant → Request a Certificate From a Certificate Authority
# 3. กรอก Email, Common Name
# 4. บันทึกเป็น CertificateSigningRequest.certSigningRequest

# Upload CSR ใน Apple Developer Portal:
# 1. certificates.developer.apple.com
# 2. + (Add Certificate)
# 3. iOS Distribution (App Store and Ad Hoc)
# 4. Upload CSR file
# 5. Download .cer file
# 6. Double-click เพื่อ install ใน Keychain
```

### Export Certificate เป็น .p12

```bash
# 1. เปิด Keychain Access
# 2. ค้นหา "iPhone Distribution"
# 3. Right-click → Export
# 4. เลือก Personal Information Exchange (.p12)
# 5. ตั้ง password
# 6. บันทึก file

# ตรวจสอบ certificate ด้วย command line:
security find-identity -v -p codesigning
```

### สร้าง Provisioning Profile

```
App Store Distribution Profile:
1. developer.apple.com → Profiles → +
2. เลือก App Store
3. เลือก App ID
4. เลือก Distribution Certificate
5. ตั้งชื่อ (เช่น "YourApp App Store")
6. Download → copy ไปที่ ~/Library/MobileDevice/Provisioning Profiles/
```

### ตั้งค่าใน Xcode

```
1. เปิด YourApp.xcworkspace
2. เลือก Project → Target: YourApp
3. Tab: Signing & Capabilities

Development:
- Team: เลือก Apple Developer Team
- Bundle Identifier: com.yourcompany.yourapp
- Automatically manage signing: ✓ (แนะนำสำหรับ dev)

Distribution (manual signing):
- Automatically manage signing: ✗
- Signing Certificate: iPhone Distribution: ...
- Provisioning Profile: เลือก profile ที่ download มา
```

---

## 3. Android Keystore

### สร้าง Android Keystore

```bash
# สร้าง keystore ใหม่ด้วย keytool
keytool -genkey -v \
  -keystore android/app/release.keystore \
  -alias your_app_alias \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000

# จะถามข้อมูล:
# Enter keystore password: [กำหนด password]
# Re-enter new password: [กำหนด password อีกครั้ง]
# First and Last Name: Your Name
# Organizational Unit: Development
# Organization: Your Company
# City or Locality: Bangkok
# State or Province: Bangkok
# Country Code: TH
```

### ตรวจสอบ Keystore

```bash
# ดูข้อมูล keystore
keytool -list -v -keystore android/app/release.keystore

# ดู certificate fingerprint (สำหรับ Firebase/Google Services)
keytool -list -v \
  -keystore android/app/release.keystore \
  -alias your_app_alias
```

### ตั้งค่า Signing ใน Android

**android/gradle.properties**:
```properties
# Keystore configuration (เพิ่มใน .gitignore!)
MYAPP_RELEASE_STORE_FILE=release.keystore
MYAPP_RELEASE_KEY_ALIAS=your_app_alias
MYAPP_RELEASE_STORE_PASSWORD=your_keystore_password
MYAPP_RELEASE_KEY_PASSWORD=your_key_password
```

**android/app/build.gradle**:
```groovy
android {
    ...
    defaultConfig { ... }
    
    signingConfigs {
        debug {
            storeFile file('debug.keystore')
            storePassword 'android'
            keyAlias 'androiddebugkey'
            keyPassword 'android'
        }
        
        release {
            if (project.hasProperty('MYAPP_RELEASE_STORE_FILE')) {
                storeFile file(MYAPP_RELEASE_STORE_FILE)
                storePassword MYAPP_RELEASE_STORE_PASSWORD
                keyAlias MYAPP_RELEASE_KEY_ALIAS
                keyPassword MYAPP_RELEASE_KEY_PASSWORD
            }
        }
    }
    
    buildTypes {
        debug {
            signingConfig signingConfigs.debug
        }
        
        release {
            signingConfig signingConfigs.release
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

### ใช้ Environment Variables แทน gradle.properties

```groovy
// android/app/build.gradle - ใช้ Environment Variables สำหรับ CI/CD
signingConfigs {
    release {
        storeFile file(System.getenv("KEYSTORE_PATH") ?: 
            project.property("MYAPP_RELEASE_STORE_FILE"))
        storePassword System.getenv("KEYSTORE_PASSWORD") ?: 
            project.property("MYAPP_RELEASE_STORE_PASSWORD")
        keyAlias System.getenv("KEY_ALIAS") ?: 
            project.property("MYAPP_RELEASE_KEY_ALIAS")
        keyPassword System.getenv("KEY_PASSWORD") ?: 
            project.property("MYAPP_RELEASE_KEY_PASSWORD")
    }
}
```

---

## 4. Fastlane Match (iOS)

### Fastlane Match คืออะไร?

```
Match = Fastlane tool สำหรับ sync certificates และ profiles
- เก็บ certificates/profiles ใน Git repo หรือ S3
- ทุกคนในทีมใช้ certificates เดียวกัน
- รองรับ CI/CD โดยไม่ต้องมี Mac
- Encrypt ด้วย password
```

### Setup Fastlane Match

```bash
# ติดตั้ง Fastlane
gem install fastlane

# Initialize Match
cd ios
fastlane match init

# จะถามว่า storage type:
# 1. git (แนะนำ)
# 2. google_cloud
# 3. s3
# 4. gitlab_secure_files

# หลังจากนั้น:
# - สร้าง private git repo สำหรับเก็บ certificates
# - กำหนด URL ของ repo
```

### ตั้งค่า Matchfile

```ruby
# ios/fastlane/Matchfile
git_url("git@github.com:yourorg/certificates.git")
git_branch("main")

storage_mode("git")

type("development") # default type

app_identifier(["com.yourcompany.yourapp"])
username("your@email.com")

# Timeout settings
git_full_name("CI Bot")
git_user_email("ci@yourcompany.com")
```

### ตั้งค่า Fastfile สำหรับ Match

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  
  desc "Sync development certificates"
  lane :sync_dev do
    match(
      type: "development",
      readonly: is_ci,
      force_for_new_devices: !is_ci
    )
  end
  
  desc "Sync App Store certificates"
  lane :sync_release do
    match(
      type: "appstore",
      readonly: is_ci
    )
  end
  
  desc "Sync Ad Hoc certificates"
  lane :sync_adhoc do
    match(
      type: "adhoc",
      readonly: is_ci,
      force_for_new_devices: !is_ci
    )
  end
  
  desc "Nuke all certificates and regenerate"
  lane :nuke_and_regenerate do
    # ลบ certificates เก่าทั้งหมด
    match_nuke(type: "appstore")
    match_nuke(type: "development")
    
    # Regenerate ใหม่
    sync_release
    sync_dev
  end
  
  desc "Add new device to provisioning profiles"
  lane :register_device do |options|
    device_name = options[:name] || prompt(text: "Device name: ")
    device_udid = options[:udid] || prompt(text: "Device UDID: ")
    
    register_devices(
      devices: { device_name => device_udid }
    )
    
    # Update all provisioning profiles
    match(
      type: "development",
      force_for_new_devices: true
    )
    match(
      type: "adhoc",
      force_for_new_devices: true
    )
  end
end
```

### ใช้ Match ใน CI/CD

```yaml
# .github/workflows/ios-build.yml
- name: Setup Match
  env:
    MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
    MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_AUTH }}
  run: |
    cd ios
    bundle exec fastlane match appstore --readonly true

# หรือในกรณีใช้ SSH key:
- name: Setup SSH for Match
  uses: webfactory/ssh-agent@v0.7.0
  with:
    ssh-private-key: ${{ secrets.MATCH_SSH_KEY }}
```

---

## 5. Multiple Environments

### ตั้งค่า Multiple Bundle IDs

```
Development:   com.yourapp.dev
Staging:       com.yourapp.staging
Production:    com.yourapp
```

### iOS - Multiple Schemes

**ios/fastlane/Fastfile**:
```ruby
lane :build_staging do
  match(
    type: "appstore",
    app_identifier: "com.yourapp.staging"
  )
  
  build_app(
    scheme: "YourApp-Staging",
    configuration: "Staging",
    export_options: {
      method: "app-store",
      provisioningProfiles: {
        "com.yourapp.staging" => "match AppStore com.yourapp.staging"
      }
    }
  )
end

lane :build_production do
  match(
    type: "appstore",
    app_identifier: "com.yourapp"
  )
  
  build_app(
    scheme: "YourApp",
    configuration: "Release",
    export_options: {
      method: "app-store",
      provisioningProfiles: {
        "com.yourapp" => "match AppStore com.yourapp"
      }
    }
  )
end
```

### Android - Product Flavors

**android/app/build.gradle**:
```groovy
android {
    flavorDimensions "environment"
    
    productFlavors {
        development {
            dimension "environment"
            applicationId "com.yourapp.dev"
            versionNameSuffix "-dev"
            resValue "string", "app_name", "YourApp Dev"
        }
        
        staging {
            dimension "environment"
            applicationId "com.yourapp.staging"
            versionNameSuffix "-staging"
            resValue "string", "app_name", "YourApp Staging"
        }
        
        production {
            dimension "environment"
            applicationId "com.yourapp"
            resValue "string", "app_name", "YourApp"
        }
    }
    
    signingConfigs {
        developmentDebug {
            // debug keystore
        }
        staging {
            storeFile file("staging.keystore")
            storePassword System.getenv("STAGING_KEYSTORE_PASSWORD")
            keyAlias System.getenv("STAGING_KEY_ALIAS")
            keyPassword System.getenv("STAGING_KEY_PASSWORD")
        }
        production {
            storeFile file("release.keystore")
            storePassword System.getenv("PROD_KEYSTORE_PASSWORD")
            keyAlias System.getenv("PROD_KEY_ALIAS")
            keyPassword System.getenv("PROD_KEY_PASSWORD")
        }
    }
    
    buildTypes {
        release {
            productFlavors {
                development { signingConfig signingConfigs.developmentDebug }
                staging { signingConfig signingConfigs.staging }
                production { signingConfig signingConfigs.production }
            }
        }
    }
}
```

---

## 6. Security Best Practices

### อย่าเก็บ Credentials ใน Git

```bash
# .gitignore
android/app/release.keystore
android/app/staging.keystore
android/gradle.properties
*.p12
*.mobileprovision
fastlane/.env
*.env.local
secrets/
```

### เก็บ Keystore ใน CI/CD Secrets

```bash
# Base64 encode keystore
base64 -i android/app/release.keystore | pbcopy

# เพิ่มใน GitHub Secrets:
# ANDROID_KEYSTORE_BASE64 = (paste base64 content)
```

```yaml
# ใน GitHub Actions
- name: Decode Keystore
  run: |
    echo ${{ secrets.ANDROID_KEYSTORE_BASE64 }} | base64 -d > android/app/release.keystore
  
- name: Build Release
  env:
    KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
    KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
    KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
  run: cd android && ./gradlew bundleRelease
  
- name: Remove Keystore
  if: always()
  run: rm -f android/app/release.keystore
```

### Rotate Certificates

```
เมื่อไรควร Rotate:
- มีคนออกจากทีมที่มี access
- Certificate ใกล้หมดอายุ (1 ปีสำหรับ iOS)
- Keystore ถูก compromise
- ทุก 1-2 ปี (best practice)

iOS:
1. Revoke certificate เก่าใน Apple Developer Portal
2. Nuke Match: fastlane match nuke
3. Regenerate ใหม่: fastlane match init

Android:
1. สร้าง keystore ใหม่
2. Upload app ที่ signed ด้วย keystore เก่า + certificate ใหม่
3. หรือใช้ Play App Signing (Google จัดการ)
```

---

## Workshop: Setup Code Signing

### Complete Setup Script

**scripts/setup-signing.sh**:
```bash
#!/bin/bash
set -e

echo "🔐 Setting up Code Signing..."

# Check prerequisites
command -v fastlane >/dev/null 2>&1 || { echo "Fastlane not installed. Run: gem install fastlane"; exit 1; }
command -v keytool >/dev/null 2>&1 || { echo "Java not installed"; exit 1; }

# iOS Setup
echo "\n📱 iOS Setup"
echo "Syncing certificates with Match..."
cd ios
bundle exec fastlane sync_release
bundle exec fastlane sync_dev
cd ..

# Android Setup
echo "\n🤖 Android Setup"
if [ ! -f android/app/release.keystore ]; then
  echo "Creating Android release keystore..."
  keytool -genkey -v \
    -keystore android/app/release.keystore \
    -alias release \
    -keyalg RSA \
    -keysize 2048 \
    -validity 10000 \
    -storepass "$ANDROID_KEYSTORE_PASSWORD" \
    -keypass "$ANDROID_KEY_PASSWORD" \
    -dname "CN=Your Name, OU=Development, O=Your Company, L=Bangkok, ST=Bangkok, C=TH"
  echo "✅ Keystore created"
else
  echo "✅ Keystore already exists"
fi

echo "\n✅ Code signing setup complete!"
echo "\nNext steps:"
echo "1. Add keystore password to gradle.properties or .env"
echo "2. Add secrets to CI/CD"
echo "3. Test by building release version"
```

### Validation Script

```bash
# scripts/validate-signing.sh
#!/bin/bash

echo "🔍 Validating Code Signing Setup..."

# iOS Validation
echo "\n📱 iOS Certificates:"
security find-identity -v -p codesigning | grep "iPhone" || echo "⚠️  No iOS signing certificates found"

echo "\n📱 Provisioning Profiles:"
ls ~/Library/MobileDevice/Provisioning\ Profiles/*.mobileprovision 2>/dev/null | \
  while read profile; do
    name=$(security cms -D -i "$profile" | grep -A1 '<key>Name</key>' | tail -1 | sed 's/<[^>]*>//g' | xargs)
    echo "  ✓ $name"
  done || echo "⚠️  No provisioning profiles found"

# Android Validation
echo "\n🤖 Android Keystores:"
if [ -f android/app/release.keystore ]; then
  echo "  ✓ Release keystore found"
  keytool -list -keystore android/app/release.keystore -storepass "${ANDROID_KEYSTORE_PASSWORD:-android}" 2>/dev/null | head -5
else
  echo "  ⚠️  Release keystore not found"
fi

echo "\n✅ Validation complete"
```

---

## 7. App Store Connect API Key

### สร้าง API Key

```
1. เข้า App Store Connect → Users and Access
2. ไปที่ Keys tab
3. คลิก + เพื่อสร้าง key ใหม่
4. ตั้งชื่อ key
5. เลือก Access: App Manager หรือ Developer
6. Download .p8 file (download ได้ครั้งเดียวเท่านั้น)
7. จด Key ID และ Issuer ID
```

### ใช้ API Key ใน Fastlane

```ruby
# Fastfile
lane :deploy do
  api_key = app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
    key_content: ENV["APP_STORE_CONNECT_KEY_CONTENT"],
    # หรือใช้ path:
    # key_filepath: "fastlane/AuthKey_XXXXX.p8"
    duration: 1200, # 20 minutes
    in_house: false
  )
  
  pilot(
    api_key: api_key,
    ipa: "output/YourApp.ipa"
  )
end
```

```yaml
# GitHub Actions - เก็บ .p8 content เป็น base64
- name: Decode API Key
  run: |
    echo ${{ secrets.APP_STORE_CONNECT_API_KEY }} | base64 -d > fastlane/AuthKey.p8
```

---

## 8. Play App Signing (Android)

### ทำไมต้องใช้ Play App Signing?

```
ประโยชน์:
- Google จัดเก็บ signing key ที่ปลอดภัย
- ถ้า keystore หาย สามารถ recover ได้
- รองรับ Android App Bundle (.aab)
- จำเป็นสำหรับ instant apps

วิธีการ:
- Upload key ไปยัง Google Play (ครั้งเดียว)
- Google sign app ก่อน deliver ให้ users
- Developer ยังต้องมี upload key
```

### Setup Play App Signing

```
1. Play Console → YourApp → Setup → App signing
2. คลิก "Export and upload a key from Java keystore"
3. Run command ที่ Google แนะนำ:

java -jar pepk.jar \
  --keystore=android/app/release.keystore \
  --alias=your_alias \
  --output=output.zip \
  --include-cert \
  --encryptionkey=GOOGLE_PUBLIC_KEY

4. Upload output.zip ไปยัง Play Console
5. Google จะเก็บ app signing key
6. สร้าง upload key ใหม่สำหรับ CI/CD
```

---

## Tips และ Best Practices

### 1. Certificate Backup

```bash
# Export certificates สำหรับ backup
# iOS:
# - Keychain Access → Export iOS Distribution certificate เป็น .p12
# - เก็บในที่ปลอดภัย (encrypted storage)

# Android:
# - Copy keystore ไว้หลายที่
cp android/app/release.keystore ~/Documents/backup/
# อย่าลืม password!
```

### 2. Document Signing Information

```markdown
# Signing Information (เก็บใน password manager)

## iOS
- Apple Developer Account: developer@company.com
- Team ID: XXXXXXXXXX
- Match Repo: git@github.com:org/certs.git
- Match Password: (stored in 1Password: "Match Password")

## Android
- Keystore Location: android/app/release.keystore
- Keystore Password: (stored in 1Password: "Android Keystore")
- Key Alias: release
- Key Password: (stored in 1Password: "Android Key")

## CI/CD Secrets
- GitHub Secrets: ANDROID_KEYSTORE_BASE64, ANDROID_KEYSTORE_PASSWORD, ...
- Updated: 2024-01-01
- Expires: 2034-01-01 (Android keystore)
```

### 3. Testing Signing Setup

```bash
# ทดสอบ iOS signing
cd ios && bundle exec fastlane build_app --dry-run

# ทดสอบ Android signing
cd android && ./gradlew assembleRelease

# ตรวจสอบ APK signing
apksigner verify --print-certs android/app/build/outputs/apk/release/app-release.apk

# ตรวจสอบ AAB signing
bundletool validate --bundle=android/app/build/outputs/bundle/release/app-release.aab
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **iOS Certificates**: สร้างและจัดการ certificates
2. **Provisioning Profiles**: ตั้งค่าสำหรับ distribution
3. **Android Keystore**: สร้างและจัดการ keystores
4. **Fastlane Match**: Sync certificates ในทีม
5. **Multiple Environments**: จัดการ signing สำหรับหลาย environments
6. **Security Best Practices**: รักษาความปลอดภัยของ credentials
7. **Play App Signing**: Google's managed signing service

### แบบฝึกหัด

1. สร้าง iOS development certificate และ provisioning profile
2. สร้าง Android release keystore
3. ตั้งค่า Fastlane Match กับ private Git repo
4. Setup CI/CD pipeline ที่ใช้ secrets จัดการ signing
5. สร้าง script สำหรับ validate signing setup
