# Part 065: CI/CD Pipeline สำหรับ React Native

## บทนำ

CI/CD (Continuous Integration/Continuous Deployment) เป็นกระบวนการอัตโนมัติที่ช่วยให้ทีมพัฒนาสามารถ build, test และ deploy แอปได้อย่างรวดเร็วและน่าเชื่อถือ ในบทนี้เราจะเรียนรู้การสร้าง CI/CD pipeline สำหรับ React Native

---

## 1. ทำความเข้าใจ CI/CD

### CI (Continuous Integration)

```
- รัน tests อัตโนมัติทุกครั้งที่ push code
- ตรวจสอบ code quality (lint, type check)
- Build แอปอัตโนมัติ
- แจ้งเตือนทีมเมื่อมีปัญหา
```

### CD (Continuous Delivery/Deployment)

```
- Deploy app ไปยัง test environment อัตโนมัติ
- Upload ไป TestFlight / Play Console
- แจ้งเตือน QA team
- Release ไปยัง production เมื่อผ่าน approval
```

### ทำไมต้องมี CI/CD?

```
- ลดข้อผิดพลาดจาก manual processes
- เร็วขึ้น: build/deploy อัตโนมัติ
- ตรวจพบ bugs เร็วขึ้น
- เพิ่มความมั่นใจในการ release
- Audit trail ของทุก deployment
```

---

## 2. GitHub Actions

### พื้นฐาน GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '18'
  JAVA_VERSION: '11'

jobs:
  setup:
    name: Setup & Cache
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### Workflow สำหรับ Code Quality

```yaml
# .github/workflows/quality.yml
name: Code Quality

on:
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    name: ESLint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Run ESLint with annotations
        uses: ataylorme/eslint-annotate-action@v2
        if: failure()
        with:
          repo-token: "${{ secrets.GITHUB_TOKEN }}"
          report-json: "eslint-results.json"

  type-check:
    name: TypeScript Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run TypeScript check
        run: npx tsc --noEmit

  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test -- --coverage --ci
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: true
      
      - name: Comment PR with coverage
        uses: romeovs/lcov-reporter-action@v0.3.1
        if: github.event_name == 'pull_request'
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          lcov-file: ./coverage/lcov.info
```

### Workflow สำหรับ iOS Build

```yaml
# .github/workflows/ios-build.yml
name: iOS Build

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  build-ios:
    name: Build iOS
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Node dependencies
        run: npm ci
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true
      
      - name: Install Fastlane
        run: gem install fastlane
      
      - name: Cache Pods
        uses: actions/cache@v3
        with:
          path: ios/Pods
          key: ${{ runner.os }}-pods-${{ hashFiles('**/Podfile.lock') }}
      
      - name: Install CocoaPods
        run: |
          cd ios
          pod install --repo-update
      
      - name: Setup Code Signing
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
        run: |
          cd ios
          bundle exec fastlane sync_certificates
      
      - name: Build iOS App
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
        run: |
          cd ios
          bundle exec fastlane build_app
      
      - name: Upload to TestFlight
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
        run: |
          cd ios
          bundle exec fastlane upload_testflight
      
      - name: Upload IPA artifact
        uses: actions/upload-artifact@v3
        with:
          name: ios-release
          path: ios/output/*.ipa

  notify:
    name: Notify Team
    needs: build-ios
    runs-on: ubuntu-latest
    if: always()
    
    steps:
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'iOS build ${{ job.status }}'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Workflow สำหรับ Android Build

```yaml
# .github/workflows/android-build.yml
name: Android Build

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-android:
    name: Build Android
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Node dependencies
        run: npm ci
      
      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
      
      - name: Setup Android Keystore
        env:
          KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
        run: |
          echo $KEYSTORE_BASE64 | base64 -d > android/app/release.keystore
      
      - name: Build Android Release APK
        env:
          KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
        run: |
          cd android
          ./gradlew bundleRelease
      
      - name: Sign APK
        uses: r0adkll/sign-android-release@v1
        id: sign_app
        with:
          releaseDirectory: android/app/build/outputs/bundle/release
          signingKeyBase64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
          alias: ${{ secrets.ANDROID_KEY_ALIAS }}
          keyStorePassword: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          keyPassword: ${{ secrets.ANDROID_KEY_PASSWORD }}
      
      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.GOOGLE_PLAY_SERVICE_ACCOUNT }}
          packageName: com.yourapp
          releaseFiles: android/app/build/outputs/bundle/release/*.aab
          track: internal
          status: completed
      
      - name: Upload AAB artifact
        uses: actions/upload-artifact@v3
        with:
          name: android-release
          path: android/app/build/outputs/bundle/release/*.aab
```

---

## 3. Fastlane

### การตั้งค่า Fastlane สำหรับ iOS

**ios/Gemfile**:
```ruby
source "https://rubygems.org"

gem "fastlane"
gem "cocoapods"
```

**ios/fastlane/Fastfile**:
```ruby
# iOS Fastfile
default_platform(:ios)

BUNDLE_ID = "com.yourcompany.yourapp"
APPLE_TEAM_ID = ENV["APPLE_TEAM_ID"]
MATCH_GIT_URL = ENV["MATCH_GIT_URL"]

platform :ios do
  
  before_all do
    # ตรวจสอบ environment
    ensure_env_vars(
      env_vars: ['MATCH_PASSWORD', 'APP_STORE_CONNECT_API_KEY_ID']
    )
  end

  desc "Sync certificates and provisioning profiles"
  lane :sync_certificates do
    match(
      type: "appstore",
      app_identifier: BUNDLE_ID,
      git_url: MATCH_GIT_URL,
      readonly: is_ci,
      verbose: false
    )
  end

  desc "Build the iOS app"
  lane :build_app do
    # อัพเดท version number จาก package.json
    version = JSON.parse(File.read("../../package.json"))["version"]
    increment_version_number(version_number: version)
    increment_build_number(build_number: ENV["GITHUB_RUN_NUMBER"] || "1")

    sync_certificates

    gym(
      scheme: "YourApp",
      workspace: "YourApp.xcworkspace",
      configuration: "Release",
      export_method: "app-store",
      output_directory: "./output",
      output_name: "YourApp.ipa",
      clean: true
    )
  end

  desc "Upload to TestFlight"
  lane :upload_testflight do
    api_key = app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_API_KEY"]
    )

    pilot(
      api_key: api_key,
      ipa: "./output/YourApp.ipa",
      changelog: "Latest changes from CI",
      distribute_external: false,
      skip_waiting_for_build_processing: true
    )
  end

  desc "Deploy to App Store"
  lane :deploy_production do
    build_app
    
    api_key = app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_API_KEY"]
    )

    deliver(
      api_key: api_key,
      ipa: "./output/YourApp.ipa",
      submit_for_review: false,
      automatic_release: false,
      force: true,
      metadata_path: "./fastlane/metadata",
      screenshots_path: "./fastlane/screenshots"
    )
  end

  desc "Run tests"
  lane :test do
    run_tests(
      workspace: "YourApp.xcworkspace",
      scheme: "YourApp",
      clean: true,
      devices: ["iPhone 14"]
    )
  end

  error do |lane, exception|
    # แจ้งเตือนเมื่อเกิด error
    slack(
      message: "Lane #{lane} failed: #{exception.message}",
      success: false,
      webhook_url: ENV["SLACK_WEBHOOK"]
    ) if ENV["SLACK_WEBHOOK"]
  end
end
```

### Fastlane สำหรับ Android

**android/fastlane/Fastfile**:
```ruby
# Android Fastfile
default_platform(:android)

PACKAGE_NAME = "com.yourcompany.yourapp"

platform :android do
  
  desc "Build release APK"
  lane :build_release do
    version = JSON.parse(File.read("../../package.json"))["version"]
    
    gradle(
      task: "bundle",
      build_type: "Release",
      project_dir: "./",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"],
      }
    )
  end

  desc "Upload to Play Store (Internal testing)"
  lane :upload_internal do
    build_release

    upload_to_play_store(
      track: "internal",
      package_name: PACKAGE_NAME,
      aab: "../app/build/outputs/bundle/release/app-release.aab",
      json_key: ENV["GOOGLE_PLAY_SERVICE_ACCOUNT_KEY"],
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true
    )
  end

  desc "Promote to production"
  lane :promote_to_production do
    upload_to_play_store(
      package_name: PACKAGE_NAME,
      track: "internal",
      track_promote_to: "production",
      json_key: ENV["GOOGLE_PLAY_SERVICE_ACCOUNT_KEY"]
    )
  end

  desc "Run tests"
  lane :test do
    gradle(
      task: "test",
      project_dir: "./"
    )
  end
end
```

---

## 4. Bitrise

### ตัวอย่าง bitrise.yml

```yaml
# bitrise.yml
format_version: '13'
default_step_lib_source: https://github.com/bitrise-io/bitrise-steplib.git
project_type: react-native

app:
  envs:
    - NODE_VERSION: '18'
    - ANDROID_COMPILE_SDK: '33'

workflows:
  
  test:
    description: Run tests and lint
    steps:
      - activate-ssh-key@4: {}
      - git-clone@6: {}
      - nvm@1:
          inputs:
            - node_version: $NODE_VERSION
      - npm@1:
          inputs:
            - command: install
      - npm@1:
          inputs:
            - command: run lint
      - npm@1:
          inputs:
            - command: test -- --ci --coverage

  ios-staging:
    description: Build and deploy iOS staging
    before_run:
      - test
    steps:
      - activate-ssh-key@4: {}
      - git-clone@6: {}
      - nvm@1:
          inputs:
            - node_version: $NODE_VERSION
      - npm@1:
          inputs:
            - command: install
      - cocoapods-install@2: {}
      - certificate-and-profile-installer@1: {}
      - xcode-archive@4:
          inputs:
            - scheme: YourApp
            - configuration: Release
            - distribution_method: app-store
      - deploy-to-bitrise-io@2: {}
      - testflight-deploy@1:
          inputs:
            - apple_id: $APPLE_ID
            - app_password: $APPLE_APP_SPECIFIC_PASSWORD

  android-staging:
    description: Build and deploy Android staging
    before_run:
      - test
    steps:
      - activate-ssh-key@4: {}
      - git-clone@6: {}
      - nvm@1:
          inputs:
            - node_version: $NODE_VERSION
      - npm@1:
          inputs:
            - command: install
      - android-build@1:
          inputs:
            - project_location: android
            - build_type: bundle
      - sign-apk@1:
          inputs:
            - android_app: android/app/build/outputs/bundle/release/app-release.aab
      - google-play@1:
          inputs:
            - package_name: com.yourapp
            - track: internal
```

---

## 5. Environment Variables และ Secrets

### การจัดการ Secrets ใน GitHub Actions

```yaml
# วิธีใช้ secrets ใน workflow
env:
  API_KEY: ${{ secrets.API_KEY }}
  
steps:
  - name: Create .env file
    run: |
      echo "API_URL=${{ secrets.API_URL }}" >> .env
      echo "API_KEY=${{ secrets.API_KEY }}" >> .env
      echo "ENVIRONMENT=production" >> .env
```

### .env Files สำหรับแต่ละ Environment

```bash
# .env.development
API_URL=http://localhost:3000
DEBUG=true
ANALYTICS_ENABLED=false

# .env.staging
API_URL=https://api-staging.example.com
DEBUG=false
ANALYTICS_ENABLED=true

# .env.production
API_URL=https://api.example.com
DEBUG=false
ANALYTICS_ENABLED=true
SENTRY_DSN=https://xxxxx@sentry.io/xxxxx
```

### ใช้ react-native-config

```bash
npm install react-native-config
```

```typescript
// ในโค้ด
import Config from 'react-native-config';

const apiUrl = Config.API_URL;
const isDebug = Config.DEBUG === 'true';
```

---

## 6. Version Management

### Auto-increment Version

```javascript
// scripts/bumpVersion.js
const fs = require('fs');
const path = require('path');

const bumpType = process.argv[2] || 'patch'; // major, minor, patch

// Update package.json
const packagePath = path.join(__dirname, '../package.json');
const package = JSON.parse(fs.readFileSync(packagePath, 'utf8'));
const [major, minor, patch] = package.version.split('.').map(Number);

let newVersion;
switch (bumpType) {
  case 'major':
    newVersion = `${major + 1}.0.0`;
    break;
  case 'minor':
    newVersion = `${major}.${minor + 1}.0`;
    break;
  default:
    newVersion = `${major}.${minor}.${patch + 1}`;
}

package.version = newVersion;
fs.writeFileSync(packagePath, JSON.stringify(package, null, 2));

console.log(`Version bumped to ${newVersion}`);
```

---

## 7. Pipeline Monitoring

### Slack Notifications

```yaml
# ใน GitHub Actions
- name: Notify Slack on success
  if: success()
  uses: 8398a7/action-slack@v3
  with:
    status: custom
    custom_payload: |
      {
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "✅ *Build Successful*\n*App:* ${{ github.repository }}\n*Branch:* ${{ github.ref_name }}\n*Triggered by:* ${{ github.actor }}"
            }
          }
        ]
      }
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}

- name: Notify Slack on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: custom
    custom_payload: |
      {
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "❌ *Build Failed*\n*App:* ${{ github.repository }}\n*Branch:* ${{ github.ref_name }}\n*Triggered by:* ${{ github.actor }}\n*View logs:* ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }
          }
        ]
      }
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Workshop: Complete CI/CD Pipeline

### Full Pipeline Workflow

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [created]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Stage 1: Validation
  validate:
    name: Validate Code
    runs-on: ubuntu-latest
    outputs:
      should-deploy: ${{ steps.check.outputs.should-deploy }}
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      
      - id: check
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "should-deploy=true" >> $GITHUB_OUTPUT
          else
            echo "should-deploy=false" >> $GITHUB_OUTPUT
          fi

  # Stage 2: Test
  test:
    name: Run Tests
    needs: validate
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --ci --coverage --maxWorkers=2
      
      - uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  # Stage 3a: iOS Build
  build-ios:
    name: Build iOS
    needs: test
    runs-on: macos-latest
    if: needs.validate.outputs.should-deploy == 'true'
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: cd ios && pod install
      - name: Build
        run: cd ios && bundle exec fastlane build_app
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
      - uses: actions/upload-artifact@v3
        with:
          name: ios-ipa
          path: ios/output/*.ipa

  # Stage 3b: Android Build
  build-android:
    name: Build Android
    needs: test
    runs-on: ubuntu-latest
    if: needs.validate.outputs.should-deploy == 'true'
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - name: Build
        run: cd android && ./gradlew bundleRelease
        env:
          KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
      - uses: actions/upload-artifact@v3
        with:
          name: android-aab
          path: android/app/build/outputs/bundle/release/*.aab

  # Stage 4: Deploy to Staging
  deploy-staging:
    name: Deploy to Staging
    needs: [build-ios, build-android]
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: ios-ipa
          path: artifacts/ios
      - uses: actions/download-artifact@v3
        with:
          name: android-aab
          path: artifacts/android
      
      - name: Upload iOS to TestFlight
        run: echo "Uploading to TestFlight..."
        # bundle exec fastlane upload_testflight
      
      - name: Upload Android to Play Console
        run: echo "Uploading to Play Console Internal..."
        # bundle exec fastlane upload_internal
      
      - name: Notify team
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              "text": "🚀 New build deployed to staging!\nVersion: ${{ github.sha }}"
            }
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  # Stage 5: Deploy to Production (manual approval)
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    if: github.event_name == 'release'
    
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to production
        run: echo "Deploying to production..."
```

---

## Tips และ Best Practices

### 1. Matrix Testing

```yaml
# ทดสอบบน Node.js หลาย versions
strategy:
  matrix:
    node: [16, 18, 20]

steps:
  - uses: actions/setup-node@v3
    with:
      node-version: ${{ matrix.node }}
```

### 2. Caching Dependencies

```yaml
# Cache Node modules
- uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### 3. Skip CI เมื่อไม่จำเป็น

```bash
# ใน commit message เพื่อ skip CI
git commit -m "Update README [skip ci]"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **GitHub Actions**: Workflows สำหรับ CI/CD
2. **Bitrise**: Mobile CI/CD platform
3. **Fastlane**: อัตโนมัติ iOS/Android builds
4. **Automated Testing**: รัน tests ใน CI
5. **Automated Deployment**: Deploy ไปยัง TestFlight/Play Console
6. **Environment Management**: จัดการ secrets และ environments
