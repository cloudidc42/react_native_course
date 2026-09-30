# Part 064: E2E Testing ด้วย Detox ใน React Native

## บทนำ

End-to-End (E2E) Testing คือการทดสอบที่จำลองการใช้งานจริงของผู้ใช้ตั้งแต่ต้นจนจบ Detox เป็น framework ยอดนิยมสำหรับทำ E2E testing บน React Native ที่รองรับทั้ง iOS และ Android

---

## 1. ทำความเข้าใจ E2E Testing

### ความแตกต่างจาก Unit และ Integration Testing

```
Unit Test:     ทดสอบฟังก์ชันเดียว ใช้ mock ทั้งหมด
Integration:   ทดสอบหลาย components ร่วมกัน ใช้ mock บางส่วน
E2E Test:      ทดสอบทั้งแอปจริง บน device/simulator จริง
```

### เมื่อไรควรใช้ E2E Tests

```
- ทดสอบ user journeys สำคัญ (login, checkout, registration)
- ทดสอบ cross-screen interactions
- ทดสอบ native features (camera, GPS, biometrics)
- ทดสอบ deep links
- ทดสอบก่อน release
```

---

## 2. ติดตั้งและตั้งค่า Detox

### ติดตั้ง Dependencies

```bash
# ติดตั้ง Detox CLI globally
npm install -g detox-cli

# ติดตั้ง Detox ใน project
npm install --save-dev detox

# ติดตั้ง test runner
npm install --save-dev jest jest-circus

# ติดตั้ง additional dependencies
npm install --save-dev @types/detox
```

### ตั้งค่า .detoxrc.js

```javascript
/** @type {Detox.DetoxConfig} */
module.exports = {
  testRunner: {
    args: {
      '$0': 'jest',
      config: 'e2e/jest.config.js',
    },
    jest: {
      setupTimeout: 120000,
    },
  },
  apps: {
    'ios.release': {
      type: 'ios.app',
      binaryPath: 'ios/build/Build/Products/Release-iphonesimulator/YourApp.app',
      build: 'xcodebuild -workspace ios/YourApp.xcworkspace -scheme YourApp -configuration Release -sdk iphonesimulator -derivedDataPath ios/build',
    },
    'ios.debug': {
      type: 'ios.app',
      binaryPath: 'ios/build/Build/Products/Debug-iphonesimulator/YourApp.app',
      build: 'xcodebuild -workspace ios/YourApp.xcworkspace -scheme YourApp -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build',
    },
    'android.release': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/release/app-release.apk',
      build: 'cd android && ./gradlew assembleRelease assembleAndroidTest -DtestBuildType=release',
      reversePorts: [8081],
    },
    'android.debug': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/debug/app-debug.apk',
      build: 'cd android && ./gradlew assembleDebug assembleAndroidTest -DtestBuildType=debug',
      reversePorts: [8081],
    },
  },
  devices: {
    simulator: {
      type: 'ios.simulator',
      device: {
        type: 'iPhone 14',
        os: 'iOS 16.0',
      },
    },
    attached: {
      type: 'android.attached',
      device: {
        adbName: '.*',
      },
    },
    emulator: {
      type: 'android.emulator',
      device: {
        avdName: 'Pixel_4_API_30',
      },
    },
  },
  configurations: {
    'ios.sim.release': {
      device: 'simulator',
      app: 'ios.release',
    },
    'ios.sim.debug': {
      device: 'simulator',
      app: 'ios.debug',
    },
    'android.emu.release': {
      device: 'emulator',
      app: 'android.release',
    },
    'android.emu.debug': {
      device: 'emulator',
      app: 'android.debug',
    },
  },
};
```

### e2e/jest.config.js

```javascript
/** @type {import('@jest/types').Config.InitialOptions} */
module.exports = {
  rootDir: '..',
  testMatch: ['<rootDir>/e2e/**/*.e2e.{js,ts}'],
  testTimeout: 120000,
  maxWorkers: 1,
  globalSetup: 'detox/runners/jest/globalSetup',
  globalTeardown: 'detox/runners/jest/globalTeardown',
  reporters: ['detox/runners/jest/reporter'],
  testEnvironment: 'detox/runners/jest/testEnvironment',
  verbose: true,
};
```

### e2e/setup.ts

```typescript
import { cleanup, init } from 'detox';
import adapter from 'detox/runners/jest/adapter';

jest.setTimeout(120000);

// @ts-ignore
jasmine.getEnv().addReporter(adapter);

beforeAll(async () => {
  await init(require('../.detoxrc.js'));
});

afterAll(async () => {
  await adapter.afterAll();
  await cleanup();
});
```

---

## 3. การเพิ่ม testID ใน Components

### Best Practices สำหรับ testID

```typescript
// ควรเพิ่ม testID ใน elements สำคัญ
import React from 'react';
import { View, Text, TextInput, TouchableOpacity } from 'react-native';

const LoginScreen: React.FC = () => {
  return (
    <View testID="login-screen">
      <Text testID="login-title">Sign In</Text>
      
      <TextInput
        testID="email-input"
        placeholder="Email"
        accessibilityLabel="Email input"
      />
      
      <TextInput
        testID="password-input"
        placeholder="Password"
        secureTextEntry
        accessibilityLabel="Password input"
      />
      
      <TouchableOpacity testID="login-button">
        <Text>Sign In</Text>
      </TouchableOpacity>
      
      <TouchableOpacity testID="forgot-password-link">
        <Text>Forgot Password?</Text>
      </TouchableOpacity>
      
      <TouchableOpacity testID="register-link">
        <Text>Create Account</Text>
      </TouchableOpacity>
    </View>
  );
};
```

---

## 4. เขียน E2E Tests

### Test พื้นฐาน

**e2e/firstLaunch.e2e.ts**:
```typescript
import { by, device, element, expect } from 'detox';

describe('App Launch', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  afterAll(async () => {
    await device.terminateApp();
  });

  it('should show login screen on first launch', async () => {
    await expect(element(by.id('login-screen'))).toBeVisible();
  });

  it('should display login title', async () => {
    await expect(element(by.id('login-title'))).toBeVisible();
    await expect(element(by.text('Sign In'))).toBeVisible();
  });

  it('should show email and password inputs', async () => {
    await expect(element(by.id('email-input'))).toBeVisible();
    await expect(element(by.id('password-input'))).toBeVisible();
  });

  it('should have login button', async () => {
    await expect(element(by.id('login-button'))).toBeVisible();
  });
});
```

### Test Login Flow

**e2e/auth/login.e2e.ts**:
```typescript
import { by, device, element, expect, waitFor } from 'detox';

describe('Login Flow', () => {
  beforeEach(async () => {
    await device.launchApp({ newInstance: true });
  });

  afterEach(async () => {
    await device.terminateApp();
  });

  describe('Successful Login', () => {
    it('should login with valid credentials', async () => {
      // Type email
      await element(by.id('email-input')).tap();
      await element(by.id('email-input')).typeText('user@example.com');
      
      // Type password
      await element(by.id('password-input')).tap();
      await element(by.id('password-input')).typeText('Password123!');
      
      // Hide keyboard
      await element(by.id('password-input')).tapReturnKey();
      
      // Press login button
      await element(by.id('login-button')).tap();
      
      // Wait for home screen
      await waitFor(element(by.id('home-screen')))
        .toBeVisible()
        .withTimeout(5000);
    });

    it('should persist login state after app restart', async () => {
      // Login first
      await element(by.id('email-input')).typeText('user@example.com');
      await element(by.id('password-input')).typeText('Password123!');
      await element(by.id('login-button')).tap();
      
      await waitFor(element(by.id('home-screen'))).toBeVisible().withTimeout(5000);
      
      // Restart app (not new instance)
      await device.launchApp({ newInstance: false });
      
      // Should still show home screen
      await expect(element(by.id('home-screen'))).toBeVisible();
    });
  });

  describe('Failed Login', () => {
    it('should show error for invalid credentials', async () => {
      await element(by.id('email-input')).typeText('wrong@example.com');
      await element(by.id('password-input')).typeText('wrongpassword');
      await element(by.id('login-button')).tap();
      
      await waitFor(element(by.id('login-error-message')))
        .toBeVisible()
        .withTimeout(5000);
      
      await expect(element(by.text('Invalid email or password'))).toBeVisible();
    });

    it('should show validation errors for empty fields', async () => {
      await element(by.id('login-button')).tap();
      
      await expect(element(by.id('email-error'))).toBeVisible();
      await expect(element(by.id('password-error'))).toBeVisible();
    });

    it('should show error for invalid email format', async () => {
      await element(by.id('email-input')).typeText('not-an-email');
      await element(by.id('login-button')).tap();
      
      await expect(element(by.text('Please enter a valid email'))).toBeVisible();
    });
  });

  describe('Navigation from Login', () => {
    it('should navigate to forgot password screen', async () => {
      await element(by.id('forgot-password-link')).tap();
      
      await waitFor(element(by.id('forgot-password-screen')))
        .toBeVisible()
        .withTimeout(3000);
    });

    it('should navigate to register screen', async () => {
      await element(by.id('register-link')).tap();
      
      await waitFor(element(by.id('register-screen')))
        .toBeVisible()
        .withTimeout(3000);
    });
  });
});
```

### Test Registration Flow

**e2e/auth/register.e2e.ts**:
```typescript
import { by, device, element, expect, waitFor } from 'detox';

describe('Registration Flow', () => {
  beforeEach(async () => {
    await device.launchApp({ newInstance: true });
    // Navigate to register screen
    await element(by.id('register-link')).tap();
    await waitFor(element(by.id('register-screen'))).toBeVisible().withTimeout(3000);
  });

  afterEach(async () => {
    await device.terminateApp();
  });

  const fillRegistrationForm = async (data: {
    firstName?: string;
    lastName?: string;
    email?: string;
    password?: string;
  }) => {
    if (data.firstName) {
      await element(by.id('first-name-input')).typeText(data.firstName);
    }
    if (data.lastName) {
      await element(by.id('last-name-input')).typeText(data.lastName);
    }
    if (data.email) {
      await element(by.id('email-input')).typeText(data.email);
    }
    if (data.password) {
      await element(by.id('password-input')).typeText(data.password);
      await element(by.id('confirm-password-input')).typeText(data.password);
    }
  };

  it('should complete registration successfully', async () => {
    await fillRegistrationForm({
      firstName: 'John',
      lastName: 'Doe',
      email: `test${Date.now()}@example.com`,
      password: 'SecurePass1!',
    });
    
    // Agree to terms
    await element(by.id('terms-checkbox')).tap();
    
    // Submit
    await element(by.id('submit-button')).tap();
    
    // Wait for success
    await waitFor(element(by.id('success-screen')))
      .toBeVisible()
      .withTimeout(10000);
    
    await expect(element(by.text('Account Created!'))).toBeVisible();
  });

  it('should scroll to see all form fields', async () => {
    // Scroll down to see bottom of form
    await element(by.id('register-form')).scroll(300, 'down');
    
    await expect(element(by.id('submit-button'))).toBeVisible();
  });
});
```

---

## 5. Test Scenarios ขั้นสูง

### Testing Scroll และ FlatList

**e2e/screens/productList.e2e.ts**:
```typescript
import { by, device, element, expect, waitFor } from 'detox';

describe('Product List Screen', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
    // Login first
    await element(by.id('email-input')).typeText('user@example.com');
    await element(by.id('password-input')).typeText('Password123!');
    await element(by.id('login-button')).tap();
    await waitFor(element(by.id('home-screen'))).toBeVisible().withTimeout(5000);
    // Navigate to products
    await element(by.id('products-tab')).tap();
  });

  afterAll(async () => {
    await device.terminateApp();
  });

  it('should show product list', async () => {
    await expect(element(by.id('product-list'))).toBeVisible();
  });

  it('should scroll through product list', async () => {
    await element(by.id('product-list')).scroll(500, 'down');
    await element(by.id('product-list')).scroll(500, 'down');
    // Scroll back up
    await element(by.id('product-list')).scroll(1000, 'up');
  });

  it('should navigate to product detail', async () => {
    await element(by.id('product-item-1')).tap();
    
    await waitFor(element(by.id('product-detail-screen')))
      .toBeVisible()
      .withTimeout(3000);
  });

  it('should pull to refresh', async () => {
    await element(by.id('product-list')).scroll(200, 'up');
    // Trigger pull to refresh
    await element(by.id('product-list')).swipe('down', 'fast', 0.8);
    
    // Wait for refresh to complete
    await waitFor(element(by.id('refresh-indicator')))
      .not.toBeVisible()
      .withTimeout(5000);
  });
});
```

### Testing Modal และ Alert

**e2e/screens/deleteConfirmation.e2e.ts**:
```typescript
import { by, device, element, expect, waitFor } from 'detox';

describe('Delete Confirmation', () => {
  it('should show confirmation before deleting', async () => {
    // Press delete button
    await element(by.id('delete-item-button')).tap();
    
    // Check alert is shown
    await expect(element(by.text('Are you sure?'))).toBeVisible();
    await expect(element(by.text('This action cannot be undone'))).toBeVisible();
  });

  it('should cancel deletion', async () => {
    await element(by.id('delete-item-button')).tap();
    
    // Press cancel in alert
    await element(by.text('Cancel')).tap();
    
    // Item should still be visible
    await expect(element(by.id('item-1'))).toBeVisible();
  });

  it('should confirm deletion', async () => {
    await element(by.id('delete-item-button')).tap();
    
    // Press confirm in alert
    await element(by.text('Delete')).tap();
    
    // Item should be removed
    await waitFor(element(by.id('item-1')))
      .not.toBeVisible()
      .withTimeout(3000);
  });
});
```

### Testing Navigation Tabs

**e2e/navigation/tabNavigation.e2e.ts**:
```typescript
import { by, device, element, expect, waitFor } from 'detox';

describe('Tab Navigation', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
    // Login
    await element(by.id('email-input')).typeText('user@example.com');
    await element(by.id('password-input')).typeText('Password123!');
    await element(by.id('login-button')).tap();
    await waitFor(element(by.id('home-screen'))).toBeVisible().withTimeout(5000);
  });

  it('should show all tabs', async () => {
    await expect(element(by.id('home-tab'))).toBeVisible();
    await expect(element(by.id('search-tab'))).toBeVisible();
    await expect(element(by.id('cart-tab'))).toBeVisible();
    await expect(element(by.id('profile-tab'))).toBeVisible();
  });

  it('should navigate to Search tab', async () => {
    await element(by.id('search-tab')).tap();
    await expect(element(by.id('search-screen'))).toBeVisible();
  });

  it('should navigate to Cart tab', async () => {
    await element(by.id('cart-tab')).tap();
    await expect(element(by.id('cart-screen'))).toBeVisible();
  });

  it('should navigate back to Home tab', async () => {
    await element(by.id('home-tab')).tap();
    await expect(element(by.id('home-screen'))).toBeVisible();
  });
});
```

---

## 6. Helpers และ Page Objects

### Page Object Pattern

**e2e/pages/LoginPage.ts**:
```typescript
import { by, element, expect, waitFor } from 'detox';

class LoginPage {
  private get emailInput() {
    return element(by.id('email-input'));
  }

  private get passwordInput() {
    return element(by.id('password-input'));
  }

  private get loginButton() {
    return element(by.id('login-button'));
  }

  private get errorMessage() {
    return element(by.id('login-error-message'));
  }

  async waitForScreen() {
    await waitFor(element(by.id('login-screen')))
      .toBeVisible()
      .withTimeout(5000);
  }

  async enterEmail(email: string) {
    await this.emailInput.tap();
    await this.emailInput.typeText(email);
  }

  async enterPassword(password: string) {
    await this.passwordInput.tap();
    await this.passwordInput.typeText(password);
  }

  async tapLoginButton() {
    await this.loginButton.tap();
  }

  async login(email: string, password: string) {
    await this.enterEmail(email);
    await this.enterPassword(password);
    await this.tapLoginButton();
  }

  async expectErrorMessage(message: string) {
    await expect(element(by.text(message))).toBeVisible();
  }

  async isVisible() {
    try {
      await expect(element(by.id('login-screen'))).toBeVisible();
      return true;
    } catch {
      return false;
    }
  }
}

export default new LoginPage();
```

**e2e/pages/HomePage.ts**:
```typescript
import { by, element, expect, waitFor } from 'detox';

class HomePage {
  async waitForScreen() {
    await waitFor(element(by.id('home-screen')))
      .toBeVisible()
      .withTimeout(5000);
  }

  async isVisible() {
    try {
      await expect(element(by.id('home-screen'))).toBeVisible();
      return true;
    } catch {
      return false;
    }
  }

  async tapLogout() {
    await element(by.id('profile-tab')).tap();
    await element(by.id('logout-button')).tap();
  }

  async navigateToProducts() {
    await element(by.id('products-tab')).tap();
    await waitFor(element(by.id('product-list'))).toBeVisible().withTimeout(3000);
  }
}

export default new HomePage();
```

### ใช้ Page Objects ใน Tests

**e2e/flows/loginFlow.e2e.ts**:
```typescript
import { device } from 'detox';
import LoginPage from '../pages/LoginPage';
import HomePage from '../pages/HomePage';

describe('Login Flow with Page Objects', () => {
  beforeEach(async () => {
    await device.launchApp({ newInstance: true });
    await LoginPage.waitForScreen();
  });

  afterEach(async () => {
    await device.terminateApp();
  });

  it('should successfully login', async () => {
    await LoginPage.login('user@example.com', 'Password123!');
    await HomePage.waitForScreen();
    
    const homeVisible = await HomePage.isVisible();
    expect(homeVisible).toBe(true);
  });

  it('should fail with wrong credentials', async () => {
    await LoginPage.login('wrong@email.com', 'wrongpass');
    await LoginPage.expectErrorMessage('Invalid email or password');
  });
});
```

---

## 7. CI Integration

### GitHub Actions สำหรับ Detox

**.github/workflows/e2e-tests.yml**:
```yaml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  e2e-ios:
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install pods
        run: cd ios && pod install
      
      - name: Install Detox CLI
        run: npm install -g detox-cli
      
      - name: Build iOS app
        run: detox build --configuration ios.sim.release
      
      - name: Run E2E Tests
        run: detox test --configuration ios.sim.release --cleanup
      
      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: detox-screenshots
          path: artifacts/

  e2e-android:
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
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Detox CLI
        run: npm install -g detox-cli
      
      - name: Enable KVM
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm
      
      - name: Build Android app
        run: detox build --configuration android.emu.release
      
      - name: Start Android emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 30
          script: detox test --configuration android.emu.release --cleanup
      
      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: detox-screenshots-android
          path: artifacts/
```

---

## 8. Test Artifacts และ Screenshots

### ตั้งค่าการเก็บ Screenshots

```javascript
// .detoxrc.js
module.exports = {
  // ... other config
  artifacts: {
    rootDir: '.artifacts',
    plugins: {
      screenshot: {
        shouldTakeAutomaticSnapshots: true,
        keepOnlyFailedTestsArtifacts: true,
        takeWhen: {
          testStart: false,
          testDone: true,
        },
      },
      video: {
        enabled: true,
        keepOnlyFailedTestsArtifacts: true,
      },
      log: {
        enabled: true,
        keepOnlyFailedTestsArtifacts: false,
      },
    },
  },
};
```

### Manual Screenshots ใน Tests

```typescript
import { device } from 'detox';

it('should show product correctly', async () => {
  // Take screenshot ก่อนทำ action
  await device.takeScreenshot('before-product-detail');
  
  await element(by.id('product-1')).tap();
  
  // Take screenshot หลังทำ action
  await device.takeScreenshot('after-product-detail');
});
```

---

## Workshop: Full E2E Test Suite

### Test Shopping Flow

**e2e/flows/shoppingFlow.e2e.ts**:
```typescript
import { by, device, element, expect, waitFor } from 'detox';
import LoginPage from '../pages/LoginPage';
import HomePage from '../pages/HomePage';

describe('Complete Shopping Flow', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
    await LoginPage.waitForScreen();
    await LoginPage.login('shopper@example.com', 'Password123!');
    await HomePage.waitForScreen();
  });

  afterAll(async () => {
    await device.terminateApp();
  });

  it('step 1: browse products', async () => {
    await element(by.id('products-tab')).tap();
    await expect(element(by.id('product-list'))).toBeVisible();
  });

  it('step 2: view product detail', async () => {
    await element(by.id('product-item-0')).tap();
    await waitFor(element(by.id('product-detail-screen')))
      .toBeVisible()
      .withTimeout(3000);
    
    await expect(element(by.id('product-name'))).toBeVisible();
    await expect(element(by.id('product-price'))).toBeVisible();
    await expect(element(by.id('add-to-cart-button'))).toBeVisible();
  });

  it('step 3: add to cart', async () => {
    await element(by.id('add-to-cart-button')).tap();
    
    // Check cart badge updated
    await waitFor(element(by.id('cart-badge')))
      .toBeVisible()
      .withTimeout(2000);
    
    await expect(element(by.text('1'))).toBeVisible();
  });

  it('step 4: view cart', async () => {
    await element(by.id('cart-tab')).tap();
    await expect(element(by.id('cart-screen'))).toBeVisible();
    await expect(element(by.id('cart-item-0'))).toBeVisible();
  });

  it('step 5: proceed to checkout', async () => {
    await element(by.id('checkout-button')).tap();
    await waitFor(element(by.id('checkout-screen')))
      .toBeVisible()
      .withTimeout(3000);
  });

  it('step 6: fill payment details', async () => {
    await element(by.id('card-number-input')).typeText('4242424242424242');
    await element(by.id('expiry-input')).typeText('12/25');
    await element(by.id('cvv-input')).typeText('123');
    await element(by.id('cardholder-name-input')).typeText('John Doe');
  });

  it('step 7: place order', async () => {
    await element(by.id('place-order-button')).tap();
    
    await waitFor(element(by.id('order-confirmation-screen')))
      .toBeVisible()
      .withTimeout(10000);
    
    await expect(element(by.text('Order Placed!'))).toBeVisible();
    await expect(element(by.id('order-number'))).toBeVisible();
  });

  it('step 8: view order in history', async () => {
    await element(by.id('view-orders-button')).tap();
    await waitFor(element(by.id('orders-screen'))).toBeVisible().withTimeout(3000);
    
    // First order should be the one just placed
    await expect(element(by.id('order-item-0'))).toBeVisible();
  });
});
```

---

## Tips และ Best Practices

### 1. ใช้ Data-testid อย่างสม่ำเสมอ

```typescript
// ใน components
<TouchableOpacity testID={`product-item-${item.id}`}>
```

### 2. ใช้ waitFor แทน setTimeout

```typescript
// ไม่ดี
await new Promise(resolve => setTimeout(resolve, 3000));

// ดี
await waitFor(element(by.id('result')))
  .toBeVisible()
  .withTimeout(5000);
```

### 3. Reset State ระหว่าง Tests

```typescript
beforeEach(async () => {
  // ล้าง AsyncStorage
  await device.launchApp({
    newInstance: true,
    delete: true,
  });
});
```

### 4. แยก Critical Flow Tests

```typescript
// เทส flows สำคัญที่สุดก่อน
describe('Critical: Authentication', () => {});
describe('Critical: Checkout', () => {});
describe('Optional: Settings', () => {});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Detox Setup**: ติดตั้งและตั้งค่า Detox
2. **Writing E2E Tests**: เขียน tests จำลองผู้ใช้จริง
3. **Page Object Pattern**: จัดระเบียบ test code
4. **Test Scenarios**: ทดสอบ flows ต่างๆ
5. **CI Integration**: รัน E2E tests ใน GitHub Actions
6. **Best Practices**: เทคนิคสำหรับ E2E tests ที่ดี
