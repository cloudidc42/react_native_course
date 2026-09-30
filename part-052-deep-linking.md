# Part 052: Deep Linking ใน React Native

## ความเข้าใจเกี่ยวกับ Deep Linking

Deep Linking คือความสามารถในการเปิดแอปโดยตรงไปยังหน้าหรือ state ที่ต้องการ โดยใช้ URL หรือ link พิเศษ แทนที่จะเปิดแอปจากหน้าหลักเสมอ

### ประเภทของ Deep Links

```
1. Custom URL Scheme    - myapp://products/123
2. Universal Links (iOS) - https://www.myapp.com/products/123
3. App Links (Android)  - https://www.myapp.com/products/123
4. Branch.io / Firebase Dynamic Links - Smart links ที่ทำงานทั้งสองแพลตฟอร์ม
```

### ทำไมต้อง Deep Linking?

- **Better UX**: ผู้ใช้คลิก link แล้วเข้าไปยังหน้าที่ต้องการทันที
- **Marketing**: แคมเปญ email, SMS สามารถ link ไปยังหน้าสินค้าได้
- **Social Sharing**: Share link ที่เปิดแอปได้
- **Notifications**: Push notification ที่นำไปยังหน้าที่เกี่ยวข้อง

---

## Universal Links (iOS)

Universal Links ใช้ HTTPS URLs ที่ทำงานได้ทั้งใน web browser และในแอป

### ขั้นตอนการตั้งค่า Universal Links

#### 1. สร้าง apple-app-site-association file

ไฟล์นี้ต้องอยู่ที่ `https://yourdomain.com/.well-known/apple-app-site-association`

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.yourcompany.yourapp",
        "paths": [
          "/products/*",
          "/users/*",
          "/orders/*",
          "NOT /privacy",
          "NOT /terms"
        ]
      }
    ]
  }
}
```

#### 2. ตั้งค่า Xcode Entitlements

เพิ่มใน `YourApp.entitlements`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.developer.associated-domains</key>
    <array>
        <string>applinks:yourdomain.com</string>
        <string>applinks:www.yourdomain.com</string>
    </array>
</dict>
</plist>
```

#### 3. จัดการ URL ใน AppDelegate

```objc
// AppDelegate.mm
#import <React/RCTLinkingManager.h>

- (BOOL)application:(UIApplication *)application
   openURL:(NSURL *)url
   options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options
{
  return [RCTLinkingManager application:application openURL:url options:options];
}

- (BOOL)application:(UIApplication *)application 
   continueUserActivity:(NSUserActivity *)userActivity
   restorationHandler:(void(^)(NSArray<id<UIUserActivityRestoring>> * __nullable restorableObjects))restorationHandler
{
  return [RCTLinkingManager application:application
                   continueUserActivity:userActivity
                     restorationHandler:restorationHandler];
}
```

---

## App Links (Android)

App Links ใช้ Digital Asset Links file เพื่อยืนยัน domain ownership

### ขั้นตอนการตั้งค่า App Links

#### 1. สร้าง Digital Asset Links file

ไฟล์ต้องอยู่ที่ `https://yourdomain.com/.well-known/assetlinks.json`

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.yourcompany.yourapp",
      "sha256_cert_fingerprints": [
        "AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78"
      ]
    }
  }
]
```

#### 2. ตั้งค่า AndroidManifest.xml

```xml
<activity
  android:name=".MainActivity"
  android:launchMode="singleTask">
  
  <!-- Custom URL Scheme -->
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
  </intent-filter>
  
  <!-- App Links (HTTPS) -->
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data
      android:scheme="https"
      android:host="yourdomain.com"
      android:pathPrefix="/products" />
    <data
      android:scheme="https"
      android:host="yourdomain.com"
      android:pathPrefix="/users" />
  </intent-filter>
</activity>
```

---

## Custom URL Schemes

Custom URL Schemes เป็นวิธีที่ง่ายที่สุดในการทำ deep linking

### iOS - ตั้งค่า URL Scheme

เพิ่มใน `Info.plist`:
```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.yourcompany.yourapp</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

### Android - ตั้งค่าใน Manifest

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data
      android:scheme="myapp"
      android:host="open" />
</intent-filter>
```

---

## React Navigation Deep Linking

React Navigation มี built-in support สำหรับ deep linking

### การตั้งค่า Linking

```typescript
import { NavigationContainer, LinkingOptions } from '@react-navigation/native';
import { Linking } from 'react-native';

// กำหนด URL Patterns
const linking: LinkingOptions<RootStackParamList> = {
  prefixes: [
    'myapp://',                    // Custom scheme
    'https://www.myapp.com',       // Universal/App Links
    'https://myapp.page.link',     // Firebase Dynamic Links
  ],
  config: {
    screens: {
      Home: {
        path: '/',
      },
      ProductDetail: {
        path: 'products/:productId',
        parse: {
          productId: (id: string) => parseInt(id, 10),
        },
        stringify: {
          productId: (id: number) => id.toString(),
        },
      },
      UserProfile: {
        path: 'users/:userId',
      },
      OrderDetail: {
        path: 'orders/:orderId',
      },
      // Nested navigators
      Tabs: {
        screens: {
          HomeTab: 'home',
          ProfileTab: 'profile',
          SettingsTab: 'settings',
        },
      },
      // ModalStack
      Modal: {
        screens: {
          ShareModal: 'share',
        },
      },
    },
  },
  // Custom getInitialURL
  async getInitialURL() {
    // ตรวจสอบว่าแอปถูกเปิดด้วย URL หรือไม่
    const url = await Linking.getInitialURL();
    if (url) return url;
    
    // ตรวจสอบ push notification
    // const notificationURL = await getNotificationURL();
    // if (notificationURL) return notificationURL;
    
    return null;
  },
  // Subscribe to URL changes
  subscribe(listener) {
    const subscription = Linking.addEventListener('url', ({ url }) => {
      listener(url);
    });
    
    return () => subscription.remove();
  },
};

// App Component
const App: React.FC = () => {
  return (
    <NavigationContainer linking={linking}>
      {/* your navigator */}
    </NavigationContainer>
  );
};
```

### Navigator และ Screen Types

```typescript
import React from 'react';
import { createStackNavigator } from '@react-navigation/stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

type RootStackParamList = {
  Home: undefined;
  ProductDetail: { productId: number };
  UserProfile: { userId: string };
  OrderDetail: { orderId: string };
  Tabs: undefined;
  Modal: undefined;
};

type TabParamList = {
  HomeTab: undefined;
  ProfileTab: undefined;
  SettingsTab: undefined;
};

const Stack = createStackNavigator<RootStackParamList>();
const Tab = createBottomTabNavigator<TabParamList>();

// ใช้งาน deep link parameter ใน screen
import { RouteProp } from '@react-navigation/native';

type ProductDetailScreenRouteProp = RouteProp<RootStackParamList, 'ProductDetail'>;

interface ProductDetailScreenProps {
  route: ProductDetailScreenRouteProp;
}

const ProductDetailScreen: React.FC<ProductDetailScreenProps> = ({ route }) => {
  const { productId } = route.params;
  
  return (
    <View>
      <Text>Product ID: {productId}</Text>
      {/* แสดงรายละเอียดสินค้า */}
    </View>
  );
};
```

---

## การจัดการ Deep Links แบบ Custom

```typescript
import React, { useEffect, useCallback } from 'react';
import { Linking, Alert } from 'react-native';
import { useNavigation } from '@react-navigation/native';

const useDeedLinking = () => {
  const navigation = useNavigation();

  const handleURL = useCallback((url: string) => {
    console.log('Received URL:', url);

    try {
      // Parse URL
      const urlObj = new URL(url);
      const { hostname, pathname, searchParams } = urlObj;

      // Handle different paths
      if (pathname.startsWith('/products/')) {
        const productId = pathname.replace('/products/', '');
        navigation.navigate('ProductDetail', { productId: parseInt(productId) });
      } else if (pathname.startsWith('/users/')) {
        const userId = pathname.replace('/users/', '');
        navigation.navigate('UserProfile', { userId });
      } else if (pathname.startsWith('/orders/')) {
        const orderId = pathname.replace('/orders/', '');
        navigation.navigate('OrderDetail', { orderId });
      } else if (pathname === '/promo') {
        const code = searchParams.get('code');
        Alert.alert('Promo Code', `คุณได้รับรหัสส่วนลด: ${code}`);
      } else {
        navigation.navigate('Home');
      }
    } catch (error) {
      console.error('Failed to parse URL:', error);
    }
  }, [navigation]);

  useEffect(() => {
    // จัดการ URL เมื่อแอปถูกเปิดด้วย link (cold start)
    Linking.getInitialURL().then(url => {
      if (url) handleURL(url);
    });

    // จัดการ URL เมื่อแอปทำงานอยู่แล้ว (hot start)
    const subscription = Linking.addEventListener('url', event => {
      handleURL(event.url);
    });

    return () => subscription.remove();
  }, [handleURL]);
};
```

---

## Branch.io Integration

Branch.io เป็น service ที่ช่วยจัดการ deep links ได้ทั้งสองแพลตฟอร์ม

### การติดตั้ง

```bash
npm install react-native-branch
# iOS
cd ios && pod install
```

### การตั้งค่า Branch.io

```typescript
import branch, { BranchEvent, BranchParams } from 'react-native-branch';

// ใน App.tsx
const App = () => {
  useEffect(() => {
    // Subscribe to Branch deep links
    const unsubscribe = branch.subscribe(({ error, params, uri }) => {
      if (error) {
        console.error('Branch error:', error);
        return;
      }
      
      if (params && params['+clicked_branch_link']) {
        // ผู้ใช้คลิก Branch link
        handleBranchLink(params);
      }
    });

    return () => unsubscribe();
  }, []);

  const handleBranchLink = (params: BranchParams) => {
    const { productId, screen, campaign } = params;
    
    // Track analytics
    branch.logEvent(BranchEvent.ViewItem, {
      campaign,
    });
    
    // Navigate based on params
    if (productId) {
      // navigation.navigate('ProductDetail', { productId });
    }
  };

  return <NavigationContainer>{/* ... */}</NavigationContainer>;
};
```

### สร้าง Branch Link

```typescript
import branch from 'react-native-branch';

const createDeepLink = async (productId: string, productName: string) => {
  const buo = await branch.createBranchUniversalObject(`product/${productId}`, {
    locallyIndex: true,
    title: productName,
    contentDescription: `ดูสินค้า ${productName}`,
    contentImageUrl: `https://example.com/products/${productId}/image.jpg`,
    contentMetadata: {
      customMetadata: {
        productId,
        screen: 'ProductDetail',
      },
    },
  });

  const linkProperties = {
    feature: 'sharing',
    channel: 'app',
  };

  const controlParams = {
    $desktop_url: `https://www.myapp.com/products/${productId}`,
    $fallback_url: `https://www.myapp.com/products/${productId}`,
  };

  const { url } = await buo.generateShortUrl(linkProperties, controlParams);
  return url;
};
```

---

## Workshop: Deep Link Handling

```typescript
import React, { useEffect, useState, useCallback } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Linking,
  Alert,
  ScrollView,
  FlatList,
} from 'react-native';
import { NavigationContainer, useNavigation, RouteProp } from '@react-navigation/native';
import { createNativeStackNavigator, NativeStackNavigationProp } from '@react-navigation/native-stack';

// Types
type RootStackParamList = {
  Home: undefined;
  Product: { id: string; name?: string };
  Category: { slug: string };
  Promo: { code: string };
};

type HomeScreenNavigationProp = NativeStackNavigationProp<RootStackParamList, 'Home'>;
type ProductScreenRouteProp = RouteProp<RootStackParamList, 'Product'>;
type CategoryScreenRouteProp = RouteProp<RootStackParamList, 'Category'>;
type PromoScreenRouteProp = RouteProp<RootStackParamList, 'Promo'>;

const Stack = createNativeStackNavigator<RootStackParamList>();

// Mock products data
const products = [
  { id: '1', name: 'iPhone 15 Pro', price: 45900, category: 'electronics' },
  { id: '2', name: 'MacBook Air M3', price: 42900, category: 'computers' },
  { id: '3', name: 'AirPods Pro', price: 8990, category: 'audio' },
];

// Home Screen
const HomeScreen: React.FC = () => {
  const navigation = useNavigation<HomeScreenNavigationProp>();
  const [linkHistory, setLinkHistory] = useState<string[]>([]);

  useEffect(() => {
    // ติดตาม deep links
    const subscription = Linking.addEventListener('url', ({ url }) => {
      setLinkHistory(prev => [url, ...prev.slice(0, 4)]);
    });

    Linking.getInitialURL().then(url => {
      if (url) setLinkHistory([url]);
    });

    return () => subscription.remove();
  }, []);

  const testLinks = [
    { label: 'เปิดสินค้า iPhone', url: 'myapp://product/1' },
    { label: 'เปิดหมวด Electronics', url: 'myapp://category/electronics' },
    { label: 'ใช้ Promo Code', url: 'myapp://promo?code=SAVE20' },
  ];

  return (
    <ScrollView style={styles.screen}>
      <Text style={styles.screenTitle}>หน้าหลัก</Text>
      
      <Text style={styles.sectionTitle}>ทดสอบ Deep Links</Text>
      {testLinks.map((link, index) => (
        <TouchableOpacity
          key={index}
          style={styles.testLinkButton}
          onPress={() => Linking.openURL(link.url)}
        >
          <Text style={styles.testLinkText}>{link.label}</Text>
          <Text style={styles.testLinkUrl}>{link.url}</Text>
        </TouchableOpacity>
      ))}

      <Text style={styles.sectionTitle}>สินค้าทั้งหมด</Text>
      {products.map(product => (
        <TouchableOpacity
          key={product.id}
          style={styles.productCard}
          onPress={() => navigation.navigate('Product', { id: product.id, name: product.name })}
        >
          <View>
            <Text style={styles.productName}>{product.name}</Text>
            <Text style={styles.productPrice}>฿{product.price.toLocaleString()}</Text>
          </View>
          <Text style={styles.arrow}>›</Text>
        </TouchableOpacity>
      ))}

      {linkHistory.length > 0 && (
        <>
          <Text style={styles.sectionTitle}>ประวัติ Deep Links</Text>
          {linkHistory.map((link, index) => (
            <Text key={index} style={styles.linkHistoryItem}>{link}</Text>
          ))}
        </>
      )}
    </ScrollView>
  );
};

// Product Screen
const ProductScreen: React.FC<{ route: ProductScreenRouteProp }> = ({ route }) => {
  const navigation = useNavigation<HomeScreenNavigationProp>();
  const { id, name } = route.params;
  
  const product = products.find(p => p.id === id);

  const shareProduct = async () => {
    try {
      const shareUrl = `myapp://product/${id}`;
      await Linking.openURL(shareUrl);
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถ share ได้');
    }
  };

  if (!product) {
    return (
      <View style={styles.screen}>
        <Text style={styles.errorText}>ไม่พบสินค้า ID: {id}</Text>
      </View>
    );
  }

  return (
    <ScrollView style={styles.screen}>
      <Text style={styles.screenTitle}>{product.name}</Text>
      <Text style={styles.productDetailPrice}>฿{product.price.toLocaleString()}</Text>
      
      <View style={styles.deepLinkInfo}>
        <Text style={styles.deepLinkLabel}>Deep Link สินค้านี้:</Text>
        <Text style={styles.deepLinkUrl}>myapp://product/{id}</Text>
      </View>

      <TouchableOpacity style={styles.shareButton} onPress={shareProduct}>
        <Text style={styles.shareButtonText}>Share สินค้า</Text>
      </TouchableOpacity>
      
      <TouchableOpacity 
        style={styles.categoryButton}
        onPress={() => navigation.navigate('Category', { slug: product.category })}
      >
        <Text style={styles.categoryButtonText}>
          ดูสินค้าหมวด {product.category}
        </Text>
      </TouchableOpacity>
    </ScrollView>
  );
};

// Category Screen
const CategoryScreen: React.FC<{ route: CategoryScreenRouteProp }> = ({ route }) => {
  const { slug } = route.params;
  const categoryProducts = products.filter(p => p.category === slug);

  return (
    <ScrollView style={styles.screen}>
      <Text style={styles.screenTitle}>หมวดหมู่: {slug}</Text>
      <Text style={styles.deepLinkInfo2}>
        Deep Link: myapp://category/{slug}
      </Text>
      {categoryProducts.length > 0 ? (
        categoryProducts.map(product => (
          <View key={product.id} style={styles.productCard}>
            <Text style={styles.productName}>{product.name}</Text>
            <Text style={styles.productPrice}>฿{product.price.toLocaleString()}</Text>
          </View>
        ))
      ) : (
        <Text style={styles.emptyText}>ไม่มีสินค้าในหมวดนี้</Text>
      )}
    </ScrollView>
  );
};

// Promo Screen
const PromoScreen: React.FC<{ route: PromoScreenRouteProp }> = ({ route }) => {
  const { code } = route.params;
  const [applied, setApplied] = useState(false);

  const discounts: Record<string, number> = {
    'SAVE20': 20,
    'SAVE50': 50,
    'WELCOME': 15,
  };

  const discount = discounts[code];

  return (
    <View style={styles.screen}>
      <Text style={styles.screenTitle}>Promo Code</Text>
      
      <View style={styles.promoCard}>
        <Text style={styles.promoCode}>{code}</Text>
        {discount ? (
          <>
            <Text style={styles.promoDiscount}>ลด {discount}%</Text>
            {!applied ? (
              <TouchableOpacity
                style={styles.applyButton}
                onPress={() => setApplied(true)}
              >
                <Text style={styles.applyButtonText}>ใช้รหัส</Text>
              </TouchableOpacity>
            ) : (
              <Text style={styles.appliedText}>✓ ใช้รหัสแล้ว</Text>
            )}
          </>
        ) : (
          <Text style={styles.invalidPromo}>รหัสไม่ถูกต้องหรือหมดอายุ</Text>
        )}
      </View>
    </View>
  );
};

// Linking configuration
const linking = {
  prefixes: ['myapp://', 'https://myapp.example.com'],
  config: {
    screens: {
      Home: '',
      Product: {
        path: 'product/:id',
      },
      Category: {
        path: 'category/:slug',
      },
      Promo: {
        path: 'promo',
        parse: {
          code: (code: string) => code.toUpperCase(),
        },
      },
    },
  },
};

// Main App
const DeepLinkApp: React.FC = () => {
  return (
    <NavigationContainer linking={linking}>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen
          name="Home"
          component={HomeScreen}
          options={{ title: 'Deep Link Demo' }}
        />
        <Stack.Screen
          name="Product"
          component={ProductScreen}
          options={({ route }) => ({ title: route.params.name || 'สินค้า' })}
        />
        <Stack.Screen
          name="Category"
          component={CategoryScreen}
          options={({ route }) => ({ title: route.params.slug })}
        />
        <Stack.Screen
          name="Promo"
          component={PromoScreen}
          options={{ title: 'Promo Code' }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

const styles = StyleSheet.create({
  screen: {
    flex: 1,
    padding: 20,
    backgroundColor: '#f5f5f5',
  },
  screenTitle: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 20,
  },
  sectionTitle: {
    fontSize: 18,
    fontWeight: '600',
    color: '#444',
    marginTop: 20,
    marginBottom: 12,
  },
  testLinkButton: {
    backgroundColor: 'white',
    padding: 14,
    borderRadius: 10,
    marginBottom: 10,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
  },
  testLinkText: {
    fontSize: 15,
    fontWeight: '600',
    color: '#333',
  },
  testLinkUrl: {
    fontSize: 12,
    color: '#2196F3',
    marginTop: 3,
  },
  productCard: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: 'white',
    padding: 14,
    borderRadius: 10,
    marginBottom: 10,
    elevation: 2,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
  },
  productName: {
    fontSize: 15,
    fontWeight: '600',
    color: '#333',
  },
  productPrice: {
    fontSize: 13,
    color: '#2196F3',
    marginTop: 3,
  },
  arrow: {
    fontSize: 20,
    color: '#999',
  },
  linkHistoryItem: {
    fontSize: 12,
    color: '#666',
    backgroundColor: '#f0f0f0',
    padding: 8,
    borderRadius: 6,
    marginBottom: 5,
    fontFamily: 'monospace',
  },
  productDetailPrice: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#2196F3',
    marginBottom: 20,
  },
  deepLinkInfo: {
    backgroundColor: '#E3F2FD',
    padding: 12,
    borderRadius: 8,
    marginBottom: 15,
  },
  deepLinkLabel: {
    fontSize: 12,
    color: '#1565C0',
    marginBottom: 4,
  },
  deepLinkUrl: {
    fontSize: 13,
    color: '#1976D2',
    fontFamily: 'monospace',
  },
  deepLinkInfo2: {
    fontSize: 13,
    color: '#1976D2',
    fontFamily: 'monospace',
    marginBottom: 15,
  },
  shareButton: {
    backgroundColor: '#4CAF50',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
    marginBottom: 10,
  },
  shareButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: 'bold',
  },
  categoryButton: {
    backgroundColor: '#9C27B0',
    padding: 14,
    borderRadius: 10,
    alignItems: 'center',
  },
  categoryButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: 'bold',
  },
  errorText: {
    color: 'red',
    fontSize: 16,
    textAlign: 'center',
  },
  emptyText: {
    textAlign: 'center',
    color: '#999',
    fontSize: 16,
    marginTop: 30,
  },
  promoCard: {
    backgroundColor: 'white',
    padding: 30,
    borderRadius: 15,
    alignItems: 'center',
    elevation: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.2,
    shadowRadius: 4,
  },
  promoCode: {
    fontSize: 32,
    fontWeight: 'bold',
    color: '#333',
    letterSpacing: 3,
    marginBottom: 10,
  },
  promoDiscount: {
    fontSize: 48,
    fontWeight: 'bold',
    color: '#4CAF50',
    marginBottom: 20,
  },
  applyButton: {
    backgroundColor: '#4CAF50',
    paddingHorizontal: 30,
    paddingVertical: 12,
    borderRadius: 25,
  },
  applyButtonText: {
    color: 'white',
    fontSize: 18,
    fontWeight: 'bold',
  },
  appliedText: {
    color: '#4CAF50',
    fontSize: 20,
    fontWeight: 'bold',
  },
  invalidPromo: {
    color: '#F44336',
    fontSize: 16,
  },
});

export default DeepLinkApp;
```

---

## Tips และ Best Practices

### 1. ทดสอบ Deep Links
```bash
# iOS Simulator
xcrun simctl openurl booted myapp://products/123

# Android Emulator
adb shell am start -W -a android.intent.action.VIEW -d "myapp://products/123"
```

### 2. จัดการ Authentication
```typescript
const handleDeepLink = async (url: string) => {
  const isAuthenticated = await checkAuth();
  
  if (!isAuthenticated) {
    // บันทึก pending URL
    await AsyncStorage.setItem('@pending_deep_link', url);
    navigation.navigate('Login');
    return;
  }
  
  // ประมวลผล deep link
  processDeepLink(url);
};

// หลัง login สำเร็จ
const handleLoginSuccess = async () => {
  const pendingUrl = await AsyncStorage.getItem('@pending_deep_link');
  if (pendingUrl) {
    await AsyncStorage.removeItem('@pending_deep_link');
    processDeepLink(pendingUrl);
  }
};
```

### 3. Fallback สำหรับ Invalid Links
```typescript
const processDeepLink = (url: string) => {
  try {
    // ประมวลผล URL
    const route = parseURL(url);
    navigation.navigate(route.screen, route.params);
  } catch {
    // Fallback ไปหน้าหลักถ้า URL ไม่ถูกต้อง
    navigation.navigate('Home');
    Alert.alert('ลิงก์ไม่ถูกต้อง', 'ขออภัย ลิงก์ที่คุณใช้ไม่สามารถเปิดได้');
  }
};
```

---

## สรุป

Deep Linking เป็นฟีเจอร์สำคัญที่ช่วยให้ UX ดีขึ้น:
- **Universal Links / App Links**: ดูเป็น professional มากกว่า Custom URL Schemes
- **React Navigation**: มี built-in support ที่ใช้งานง่าย
- **Branch.io**: เหมาะสำหรับ marketing campaigns ที่ต้องการ analytics
- **ทดสอบ**: ทดสอบบน device จริงเสมอ เพราะ simulator/emulator อาจให้ผลต่าง
