# Part 098: DevOps และ Mobile Deployment

## บทนำ

DevOps สำหรับ Mobile App ช่วยทำให้ process การ build, test, และ deploy เป็นอัตโนมัติ เราจะเรียนรู้ Fastlane automation, Multi-environment setup, Feature environments, Monitoring และสร้าง Complete DevOps Pipeline

## หัวข้อที่จะเรียน

1. Fastlane Automation
2. Multi-environment Setup
3. EAS Build (Expo)
4. Monitoring และ Crash Reporting
5. Workshop: Complete DevOps Pipeline

---

## 1. Fastlane Setup

### การติดตั้ง

```bash
# macOS
brew install fastlane

# Ruby gem
gem install fastlane

# เริ่ม setup ในโปรเจค
cd ios && fastlane init
cd android && fastlane init
```

### ios/fastlane/Fastfile

```ruby
default_platform(:ios)

platform :ios do
  before_all do
    ensure_git_branch(branch: 'main|develop|release/.*')
    ensure_git_status_clean
  end

  # === Development ===
  desc "Build development app"
  lane :dev do
    setup_ci if is_ci
    
    sync_code_signing(
      type: "development",
      app_identifier: "com.yourcompany.app.dev",
    )
    
    build_app(
      scheme: "App-Dev",
      configuration: "Debug",
      output_directory: "./builds/dev",
      output_name: "App-Dev.ipa",
    )
    
    slack(
      message: "✅ Dev build สำเร็จ!",
      channel: "#mobile-builds",
    ) if ENV['SLACK_URL']
  end

  # === Staging ===
  desc "Build and deploy to TestFlight (staging)"
  lane :staging do
    setup_ci if is_ci
    
    # อัปเดต version
    increment_build_number(
      build_number: latest_testflight_build_number + 1,
    )
    
    sync_code_signing(
      type: "appstore",
      app_identifier: "com.yourcompany.app.staging",
    )
    
    build_app(
      scheme: "App-Staging",
      configuration: "Release",
      export_method: "app-store",
    )
    
    upload_to_testflight(
      app_identifier: "com.yourcompany.app.staging",
      groups: ["QA Team", "Internal Testers"],
      notify_external_testers: false,
    )
    
    slack(
      message: "🚀 Staging build uploaded to TestFlight!\nBuild: #{lane_context[SharedValues::BUILD_NUMBER]}",
      channel: "#mobile-builds",
    ) if ENV['SLACK_URL']
  end

  # === Production ===
  desc "Build and submit to App Store"
  lane :production do
    setup_ci if is_ci
    
    # ตรวจสอบว่าอยู่บน main branch
    ensure_git_branch(branch: 'main')
    
    # Pull latest
    git_pull
    
    # อัปเดต version
    increment_version_number(
      version_number: ENV['VERSION'] || prompt(text: "Version number: "),
    )
    increment_build_number(
      build_number: latest_testflight_build_number + 1,
    )
    
    sync_code_signing(
      type: "appstore",
      app_identifier: "com.yourcompany.app",
    )
    
    build_app(
      scheme: "App",
      configuration: "Release",
      export_method: "app-store",
    )
    
    upload_to_app_store(
      force: true,
      reject_if_possible: true,
      submit_for_review: false, # ส่ง manual
      automatic_release: false,
      metadata_path: "./metadata",
      screenshots_path: "./screenshots",
    )
    
    # Tag release
    add_git_tag(
      tag: "v#{get_version_number}(#{get_build_number})",
    )
    push_git_tags
    
    slack(
      message: "🎉 App Store: Version #{get_version_number} submitted!",
      channel: "#mobile-builds",
    ) if ENV['SLACK_URL']
  end

  # === Screenshots ===
  desc "Generate screenshots"
  lane :screenshots do
    capture_ios_screenshots
    frame_screenshots(white: true)
    upload_to_app_store(
      skip_binary_upload: true,
      skip_metadata: true,
    )
  end

  # === Certificate Management ===
  desc "Sync certificates"
  lane :certs do
    sync_code_signing(
      type: "development",
      readonly: is_ci,
    )
    sync_code_signing(
      type: "appstore",
      readonly: is_ci,
    )
  end

  error do |lane, exception|
    slack(
      message: "❌ #{lane} failed: #{exception.message}",
      channel: "#mobile-builds",
      success: false,
    ) if ENV['SLACK_URL']
  end
end
```

### android/fastlane/Fastfile

```ruby
default_platform(:android)

platform :android do
  # === Development ===
  desc "Build debug APK"
  lane :dev do
    gradle(
      task: "assemble",
      build_type: "Debug",
      project_dir: "./",
    )
    
    # ส่งไปยัง Firebase App Distribution
    firebase_app_distribution(
      app: ENV['FIREBASE_APP_ID_ANDROID_DEV'],
      groups: "developers",
      apk_path: "app/build/outputs/apk/debug/app-debug.apk",
    )
  end

  # === Staging ===
  desc "Build and upload to Firebase (staging)"
  lane :staging do
    # Build AAB (Android App Bundle)
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV['KEYSTORE_PATH'],
        "android.injected.signing.store.password" => ENV['KEYSTORE_PASSWORD'],
        "android.injected.signing.key.alias" => ENV['KEY_ALIAS'],
        "android.injected.signing.key.password" => ENV['KEY_PASSWORD'],
      },
    )
    
    firebase_app_distribution(
      app: ENV['FIREBASE_APP_ID_ANDROID_STAGING'],
      groups: "qa-team, internal-testers",
      aab_path: "app/build/outputs/bundle/release/app-release.aab",
    )
  end

  # === Production ===
  desc "Build and upload to Play Store"
  lane :production do
    # อัปเดต version code
    android_set_version_code(
      version_code: number_of_commits,
    )
    
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV['KEYSTORE_PATH'],
        "android.injected.signing.store.password" => ENV['KEYSTORE_PASSWORD'],
        "android.injected.signing.key.alias" => ENV['KEY_ALIAS'],
        "android.injected.signing.key.password" => ENV['KEY_PASSWORD'],
      },
    )
    
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      skip_upload_metadata: false,
      skip_upload_images: false,
    )
  end

  # === Internal Testing ===
  desc "Upload to Play Store internal testing"
  lane :internal do
    gradle(task: "bundle", build_type: "Release")
    upload_to_play_store(
      track: "internal",
      skip_upload_metadata: true,
    )
  end
end
```

---

## 2. Multi-Environment Setup

### Environment Configuration

```typescript
// src/config/environment.ts
import Config from 'react-native-config';

type Environment = 'development' | 'staging' | 'production';

interface EnvironmentConfig {
  ENV: Environment;
  API_URL: string;
  APP_NAME: string;
  BUNDLE_ID: string;
  SENTRY_DSN?: string;
  ANALYTICS_KEY?: string;
  ENABLE_LOGS: boolean;
  FEATURE_FLAGS: {
    newCheckout: boolean;
    socialLogin: boolean;
    darkMode: boolean;
  };
}

const configs: Record<Environment, EnvironmentConfig> = {
  development: {
    ENV: 'development',
    API_URL: 'http://localhost:3000',
    APP_NAME: 'MyApp Dev',
    BUNDLE_ID: 'com.yourcompany.app.dev',
    ENABLE_LOGS: true,
    FEATURE_FLAGS: {
      newCheckout: true,
      socialLogin: true,
      darkMode: true,
    },
  },
  staging: {
    ENV: 'staging',
    API_URL: 'https://api-staging.yourapp.com',
    APP_NAME: 'MyApp Staging',
    BUNDLE_ID: 'com.yourcompany.app.staging',
    SENTRY_DSN: 'https://staging@sentry.io/...',
    ENABLE_LOGS: true,
    FEATURE_FLAGS: {
      newCheckout: true,
      socialLogin: true,
      darkMode: false,
    },
  },
  production: {
    ENV: 'production',
    API_URL: 'https://api.yourapp.com',
    APP_NAME: 'MyApp',
    BUNDLE_ID: 'com.yourcompany.app',
    SENTRY_DSN: 'https://production@sentry.io/...',
    ANALYTICS_KEY: 'GA-XXXXX',
    ENABLE_LOGS: false,
    FEATURE_FLAGS: {
      newCheckout: false,
      socialLogin: true,
      darkMode: false,
    },
  },
};

const currentEnv = (Config.ENV as Environment) || 'development';

export const env = configs[currentEnv];

export const isProduction = () => env.ENV === 'production';
export const isDevelopment = () => env.ENV === 'development';
export const isStaging = () => env.ENV === 'staging';
```

### .env files

```bash
# .env.development
ENV=development
API_URL=http://localhost:3000
APP_VARIANT=development

# .env.staging
ENV=staging
API_URL=https://api-staging.yourapp.com
APP_VARIANT=staging

# .env.production
ENV=production
API_URL=https://api.yourapp.com
APP_VARIANT=production
```

---

## 3. GitHub Actions CI/CD

### .github/workflows/ci.yml

```yaml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: TypeScript check
        run: npm run type-check
      
      - name: Lint
        run: npm run lint
      
      - name: Unit tests
        run: npm test -- --coverage --ci
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build-android-staging:
    name: Build Android Staging
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/develop'
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
          node-version: 18
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true
          working-directory: android
      
      - name: Decode Keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE }}" | base64 --decode > android/app/keystore.jks
      
      - name: Run Fastlane staging
        working-directory: android
        env:
          KEYSTORE_PATH: app/keystore.jks
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          FIREBASE_APP_ID_ANDROID_STAGING: ${{ secrets.FIREBASE_APP_ID_ANDROID_STAGING }}
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
        run: bundle exec fastlane staging

  build-ios-staging:
    name: Build iOS Staging
    runs-on: macos-latest
    needs: test
    if: github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true
          working-directory: ios
      
      - name: Run Fastlane staging
        working-directory: ios
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
          APP_STORE_CONNECT_API_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_KEY }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          FIREBASE_APP_ID_IOS_STAGING: ${{ secrets.FIREBASE_APP_ID_IOS_STAGING }}
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
        run: bundle exec fastlane staging

  production-release:
    name: Production Release
    runs-on: macos-latest
    needs: test
    if: github.ref == 'refs/heads/main' && startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/checkout@v3
      # ... production release steps
```

---

## 4. Monitoring และ Crash Reporting

### Sentry Integration

```typescript
// src/utils/monitoring.ts
import * as Sentry from '@sentry/react-native';
import { env, isProduction } from '../config/environment';

export const initMonitoring = () => {
  if (!env.SENTRY_DSN) return;
  
  Sentry.init({
    dsn: env.SENTRY_DSN,
    environment: env.ENV,
    tracesSampleRate: isProduction() ? 0.1 : 1.0,
    profilesSampleRate: isProduction() ? 0.1 : 1.0,
    enabled: !!env.SENTRY_DSN,
    
    beforeSend(event) {
      // Filter sensitive data
      if (event.request?.headers) {
        delete event.request.headers['Authorization'];
      }
      return event;
    },
    
    integrations: [
      new Sentry.ReactNativeTracing({
        routingInstrumentation: new Sentry.ReactNavigationInstrumentation(),
        tracingOrigins: ['localhost', env.API_URL.replace('https://', '')],
      }),
    ],
  });
};

export const captureError = (
  error: Error,
  context?: Record<string, any>
) => {
  Sentry.withScope((scope) => {
    if (context) {
      scope.setContext('additional', context);
    }
    Sentry.captureException(error);
  });
};

export const setUserContext = (user: {
  id: string;
  email: string;
  role: string;
}) => {
  Sentry.setUser({
    id: user.id,
    email: user.email,
    username: user.role,
  });
};

export const clearUserContext = () => {
  Sentry.setUser(null);
};

export const logBreadcrumb = (
  message: string,
  data?: Record<string, any>
) => {
  Sentry.addBreadcrumb({
    message,
    data,
    level: 'info',
    timestamp: Date.now() / 1000,
  });
};
```

### Performance Monitoring

```typescript
// src/utils/performance.ts
import * as Sentry from '@sentry/react-native';

export class PerformanceMonitor {
  private transactions = new Map<string, ReturnType<typeof Sentry.startTransaction>>();

  startTransaction(name: string, operation: string) {
    const transaction = Sentry.startTransaction({ name, op: operation });
    this.transactions.set(name, transaction);
    return transaction;
  }

  finishTransaction(name: string) {
    const transaction = this.transactions.get(name);
    if (transaction) {
      transaction.finish();
      this.transactions.delete(name);
    }
  }

  measureAsync<T>(name: string, operation: string, fn: () => Promise<T>): Promise<T> {
    const transaction = this.startTransaction(name, operation);
    return fn().finally(() => transaction.finish());
  }
}

export const monitor = new PerformanceMonitor();
```

---

## 5. Workshop: Complete DevOps Pipeline

### Deployment Strategy

```
Developer → Push → GitHub Actions → 
  ├── Test (unit, e2e)
  ├── Lint & Type Check
  ├── Build (staging)
  ├── Deploy to Firebase App Distribution
  └── Notify Slack

QA Approval → 
  ├── Build (production)
  ├── Upload to App Store / Play Store
  └── Submit for Review
```

### Version Management Script

```typescript
// scripts/bump-version.ts
import { readFileSync, writeFileSync } from 'fs';
import { execSync } from 'child_process';

type BumpType = 'major' | 'minor' | 'patch';

const bumpVersion = (type: BumpType) => {
  // Read current version
  const pkg = JSON.parse(readFileSync('package.json', 'utf-8'));
  const [major, minor, patch] = pkg.version.split('.').map(Number);
  
  let newVersion: string;
  switch (type) {
    case 'major': newVersion = `${major + 1}.0.0`; break;
    case 'minor': newVersion = `${major}.${minor + 1}.0`; break;
    case 'patch': newVersion = `${major}.${minor}.${patch + 1}`; break;
  }
  
  // Update package.json
  pkg.version = newVersion;
  writeFileSync('package.json', JSON.stringify(pkg, null, 2));
  
  // Update iOS version
  execSync(`/usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString ${newVersion}" ios/App/Info.plist`);
  
  // Update Android version
  const buildGradle = readFileSync('android/app/build.gradle', 'utf-8');
  const updatedGradle = buildGradle
    .replace(/versionName ".*"/, `versionName "${newVersion}"`)
    .replace(/versionCode \d+/, `versionCode ${Date.now()}`);
  writeFileSync('android/app/build.gradle', updatedGradle);
  
  // Git commit and tag
  execSync(`git add .`);
  execSync(`git commit -m "chore: bump version to ${newVersion}"`);
  execSync(`git tag v${newVersion}`);
  
  console.log(`✅ Version bumped to ${newVersion}`);
};

const type = process.argv[2] as BumpType || 'patch';
bumpVersion(type);
```

---

## Workshop Exercises

1. **Setup Fastlane** สำหรับ project จริง
2. **GitHub Actions** pipeline ครบทุก environments
3. **Sentry Integration** และทดสอบ crash reporting
4. **OTA Updates** ด้วย CodePush หรือ EAS Update

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Fastlane** - automation สำหรับ iOS และ Android builds
2. **Multi-environment** - development, staging, production configs
3. **GitHub Actions** - CI/CD pipeline อัตโนมัติ
4. **Monitoring** - Sentry สำหรับ crash reporting และ performance
5. **Version Management** - ระบบจัดการ versions
