# Part 012: Stack Navigator

## สารบัญ
1. createNativeStackNavigator
2. Screen Options
3. Navigation Params
4. Navigation Methods
5. Nested Navigators
6. Workshop: Multi-screen App

---

## 1. createNativeStackNavigator

Stack Navigator จัดการหน้าแบบ "stack" คือ ซ้อนกันเหมือนกองไพ่ หน้าใหม่วางทับหน้าเดิม และกดปุ่ม Back เพื่อนำหน้าบนสุดออก

### การสร้าง Stack Navigator

```javascript
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

// สร้าง Stack instance
const Stack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator
        initialRouteName="Home"  // หน้าแรกที่จะแสดง
        screenOptions={{         // options ที่ใช้กับทุก screen
          headerShown: true,
          headerStyle: { backgroundColor: '#6200EE' },
          headerTintColor: '#FFF',
        }}
      >
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Detail" component={DetailScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### Stack vs NativeStack

```javascript
// Stack (JavaScript-based) - เก่า, customizable มากกว่า
import { createStackNavigator } from '@react-navigation/stack';

// NativeStack (Native-based) - ใหม่, performance ดีกว่า
import { createNativeStackNavigator } from '@react-navigation/native-stack';
```

**แนะนำให้ใช้ NativeStack** เพราะ:
- ใช้ native navigation components ของ iOS และ Android
- Animation smoother
- Performance ดีกว่า
- Memory ใช้น้อยกว่า

---

## 2. Screen Options

### Header Options ทั้งหมด

```javascript
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={{
    // ===== Header Title =====
    title: 'รายละเอียด',
    headerTitle: 'รายละเอียด',  // เหมือน title
    
    // Custom title component
    headerTitle: ({ children, style }) => (
      <Text style={[style, { color: 'gold', fontSize: 20 }]}>
        {children}
      </Text>
    ),
    
    // ===== Header Style =====
    headerStyle: {
      backgroundColor: '#6200EE',
    },
    headerTintColor: '#fff',  // สีปุ่มและข้อความ
    headerTitleStyle: {
      fontWeight: 'bold',
      fontSize: 18,
    },
    headerTitleAlign: 'center',  // 'left' | 'center'
    
    // ===== Header Buttons =====
    headerRight: () => (
      <TouchableOpacity onPress={() => alert('กด!')}>
        <Text style={{ color: '#FFF', marginRight: 10 }}>บันทึก</Text>
      </TouchableOpacity>
    ),
    headerLeft: () => (
      <TouchableOpacity onPress={() => navigation.goBack()}>
        <Text style={{ color: '#FFF', marginLeft: 10 }}>ยกเลิก</Text>
      </TouchableOpacity>
    ),
    
    // ===== Header Visibility =====
    headerShown: true,    // แสดง/ซ่อน header
    headerTransparent: true,  // Header โปร่งใส
    
    // ===== Header Shadow =====
    headerShadowVisible: false,  // ซ่อนเงา
    
    // ===== Back Button =====
    headerBackTitle: 'กลับ',  // ข้อความปุ่ม back (iOS)
    headerBackTitleVisible: true,
    headerBackButtonMenuEnabled: true,  // Long press back button
    
    // ===== Status Bar =====
    statusBarStyle: 'light',  // 'auto' | 'inverted' | 'light' | 'dark'
    statusBarColor: '#6200EE',  // Android
    statusBarHidden: false,
    
    // ===== Presentation =====
    presentation: 'card',  // 'card' | 'modal' | 'transparentModal' | 'containedModal'
    
    // ===== Gesture =====
    gestureEnabled: true,  // iOS swipe back
    fullScreenGestureEnabled: false,
    
    // ===== Animation =====
    animation: 'default',  // 'default' | 'fade' | 'slide_from_bottom' | 'none'
    animationDuration: 350,
    
    // ===== Custom Background =====
    contentStyle: {
      backgroundColor: '#F5F5F5',
    },
  }}
/>
```

### ตัวอย่าง Header แบบต่างๆ

```javascript
// 1. Header ธรรมดา
const BasicHeader = {
  title: 'หน้าหลัก',
  headerStyle: { backgroundColor: '#2196F3' },
  headerTintColor: '#FFF',
};

// 2. Header โปร่งใส (เหมาะกับหน้าที่มี image ใหญ่)
const TransparentHeader = {
  headerTransparent: true,
  headerBlurEffect: 'light',  // iOS
  headerStyle: { backgroundColor: 'transparent' },
  headerTintColor: '#FFF',
  headerShadowVisible: false,
};

// 3. ไม่มี Header
const NoHeader = {
  headerShown: false,
};

// 4. Header มีปุ่มขวา
const HeaderWithButton = ({ navigation }) => ({
  title: 'รายการ',
  headerRight: () => (
    <TouchableOpacity
      style={{ marginRight: 15 }}
      onPress={() => navigation.navigate('AddItem')}
    >
      <Ionicons name="add" size={24} color="#FFF" />
    </TouchableOpacity>
  ),
});
```

### Screen Options ที่ใช้บ่อย

```javascript
// screenOptions ระดับ Navigator (ใช้กับทุก screen)
<Stack.Navigator
  screenOptions={({ navigation, route }) => ({
    title: route.name,
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#FFF',
    headerTitleStyle: { fontWeight: 'bold' },
    
    // Custom back button ทุกหน้า
    headerLeft: () => (
      <TouchableOpacity onPress={() => navigation.goBack()}>
        <Ionicons name="arrow-back" size={24} color="#FFF" />
      </TouchableOpacity>
    ),
  })}
>
```

---

## 3. Navigation Params

### ส่ง Params ไปยังหน้าอื่น

```javascript
// HomeScreen.js
function HomeScreen({ navigation }) {
  const products = [
    { id: 1, name: 'iPhone 15', price: 35000, image: 'https://...' },
    { id: 2, name: 'Samsung S24', price: 30000, image: 'https://...' },
    { id: 3, name: 'Pixel 8', price: 25000, image: 'https://...' },
  ];

  return (
    <FlatList
      data={products}
      keyExtractor={item => item.id.toString()}
      renderItem={({ item }) => (
        <TouchableOpacity
          onPress={() => navigation.navigate('ProductDetail', {
            productId: item.id,
            productName: item.name,
            price: item.price,
            imageUrl: item.image,
          })}
        >
          <Text>{item.name}</Text>
        </TouchableOpacity>
      )}
    />
  );
}
```

### รับ Params ในหน้าปลายทาง

```javascript
// ProductDetailScreen.js
function ProductDetailScreen({ route, navigation }) {
  // วิธีที่ 1: รับทั้งหมด
  const params = route.params;
  
  // วิธีที่ 2: Destructuring
  const { productId, productName, price, imageUrl } = route.params;
  
  // วิธีที่ 3: พร้อม default values
  const { 
    productId, 
    productName = 'ไม่ทราบชื่อ', 
    price = 0 
  } = route.params || {};

  // ตั้งค่า header title จาก params
  useEffect(() => {
    navigation.setOptions({ title: productName });
  }, [navigation, productName]);

  return (
    <ScrollView>
      <Image source={{ uri: imageUrl }} style={{ height: 300 }} />
      <View style={{ padding: 16 }}>
        <Text style={{ fontSize: 24, fontWeight: 'bold' }}>{productName}</Text>
        <Text style={{ fontSize: 20, color: '#6200EE' }}>
          ฿{price.toLocaleString()}
        </Text>
      </View>
    </ScrollView>
  );
}
```

### ส่ง Params กลับ (Return Params)

```javascript
// สถานการณ์: หน้า EditProfile บันทึกข้อมูลและต้องการส่งกลับไปหน้า Profile

// EditProfileScreen.js
function EditProfileScreen({ navigation, route }) {
  const { userId } = route.params;
  const [newName, setNewName] = useState('');

  const handleSave = () => {
    // วิธีที่ 1: navigate กลับพร้อม params
    navigation.navigate('Profile', {
      updatedName: newName,
      updatedAt: new Date().toISOString(),
    });
    
    // วิธีที่ 2: goBack พร้อม params (ผ่าน state)
    // ใช้ไม่ได้โดยตรง ต้องใช้ navigate แทน
  };

  return (
    <View>
      <TextInput value={newName} onChangeText={setNewName} />
      <Button title="บันทึก" onPress={handleSave} />
    </View>
  );
}

// ProfileScreen.js
function ProfileScreen({ navigation, route }) {
  const [name, setName] = useState('สมชาย');
  
  // รับ params ที่ส่งกลับมา
  useEffect(() => {
    if (route.params?.updatedName) {
      setName(route.params.updatedName);
      // ล้าง params เพื่อไม่ให้ trigger ซ้ำ
      navigation.setParams({ updatedName: undefined });
    }
  }, [route.params?.updatedName]);

  return (
    <View>
      <Text>ชื่อ: {name}</Text>
      <Button 
        title="แก้ไข" 
        onPress={() => navigation.navigate('EditProfile', { userId: '123' })}
      />
    </View>
  );
}
```

---

## 4. Navigation Methods

### navigate()

```javascript
// ไปยัง screen ใหม่
navigation.navigate('Profile');

// ไปพร้อม params
navigation.navigate('Profile', { userId: '123' });

// ไปยัง nested screen
navigation.navigate('MainTabs', {
  screen: 'Profile',
  params: { userId: '123' }
});
```

### push()

```javascript
// เหมือน navigate แต่เพิ่มหน้าใน stack เสมอ
// แม้จะเป็นหน้าที่อยู่ใน stack แล้วก็ตาม
navigation.push('ProductDetail', { productId: '456' });

// ตัวอย่าง: gallery ที่คลิก related product ได้เรื่อยๆ
function ProductItem({ item }) {
  return (
    <TouchableOpacity onPress={() => navigation.push('Product', { id: item.id })}>
      <Text>{item.name}</Text>
    </TouchableOpacity>
  );
}
```

### goBack()

```javascript
// กลับหน้าก่อนหน้า
navigation.goBack();

// ตรวจสอบก่อน goBack
if (navigation.canGoBack()) {
  navigation.goBack();
} else {
  // อยู่หน้าแรกแล้ว
  BackHandler.exitApp();
}
```

### replace()

```javascript
// แทนที่หน้าปัจจุบันใน stack ด้วยหน้าใหม่
// ใช้เมื่อไม่ต้องการให้กลับมาหน้าเดิม

// ตัวอย่าง: หลัง login สำเร็จ แทนที่หน้า Login ด้วยหน้า Home
function LoginScreen({ navigation }) {
  const handleLogin = async () => {
    await loginUser(email, password);
    navigation.replace('Home');  // ไม่สามารถกดปุ่ม Back กลับมาหน้า Login ได้
  };
}
```

### reset()

```javascript
// รีเซ็ต navigation stack ทั้งหมด
navigation.reset({
  index: 0,           // index ของหน้าที่จะอยู่ปัจจุบัน
  routes: [           // array ของหน้าใน stack
    { name: 'Home' }
  ],
});

// ตัวอย่าง: หลัง logout ต้องการล้าง stack ทั้งหมด
function SettingsScreen({ navigation }) {
  const handleLogout = async () => {
    await logout();
    navigation.reset({
      index: 0,
      routes: [{ name: 'Login' }],
    });
  };
}

// สร้าง stack หลายหน้า
navigation.reset({
  index: 2,
  routes: [
    { name: 'Home' },
    { name: 'Category' },
    { name: 'Product', params: { id: '123' } },
  ],
});
```

### popToTop()

```javascript
// กลับไปหน้าแรกสุดของ stack
navigation.popToTop();
```

### dispatch()

```javascript
import { CommonActions } from '@react-navigation/native';

// navigate โดยใช้ action
navigation.dispatch(
  CommonActions.navigate({
    name: 'Profile',
    params: { userId: '123' },
  })
);

// reset โดยใช้ action
navigation.dispatch(
  CommonActions.reset({
    index: 0,
    routes: [{ name: 'Home' }],
  })
);
```

---

## 5. Nested Navigators

### ตัวอย่าง: App จริงที่ซับซ้อน

```javascript
// navigation/AppNavigator.js
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Stack = createNativeStackNavigator();
const Tab = createBottomTabNavigator();

// Tab Navigator สำหรับ authenticated users
function MainTabNavigator() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}

// Stack Navigator หลัก
function RootNavigator() {
  const { isLoggedIn } = useAuth();
  
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {isLoggedIn ? (
        <>
          <Stack.Screen name="MainTabs" component={MainTabNavigator} />
          <Stack.Screen 
            name="ProductDetail" 
            component={ProductDetailScreen}
            options={{ headerShown: true, title: 'รายละเอียด' }}
          />
          <Stack.Screen 
            name="Checkout" 
            component={CheckoutScreen}
            options={{ presentation: 'modal' }}
          />
        </>
      ) : (
        <>
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Register" component={RegisterScreen} />
          <Stack.Screen name="ForgotPassword" component={ForgotPasswordScreen} />
        </>
      )}
    </Stack.Navigator>
  );
}

export default function App() {
  return (
    <NavigationContainer>
      <RootNavigator />
    </NavigationContainer>
  );
}
```

### Navigate ไปยัง Nested Screen

```javascript
// ไปยัง screen ใน nested navigator
navigation.navigate('MainTabs', {
  screen: 'Profile',
  params: { userId: '123' }
});

// ไปยัง screen ที่ซ้อนกันหลายชั้น
navigation.navigate('MainTabs', {
  screen: 'Shop',
  params: {
    screen: 'Product',
    params: { productId: '456' }
  }
});
```

---

## 6. Modal Screen

```javascript
// เปิดหน้าแบบ modal (ขึ้นมาจากด้านล่าง)
<Stack.Screen
  name="AddProduct"
  component={AddProductScreen}
  options={{
    presentation: 'modal',
    title: 'เพิ่มสินค้า',
    headerLeft: ({ onPress }) => (
      <Button title="ยกเลิก" onPress={onPress} />
    ),
  }}
/>

// เรียกใช้
navigation.navigate('AddProduct');
```

---

## Workshop: Multi-screen E-Commerce App

### โครงสร้างแอป

```
ECommerceApp/
├── App.js
├── navigation/
│   └── AppNavigator.js
├── screens/
│   ├── HomeScreen.js
│   ├── ProductListScreen.js
│   ├── ProductDetailScreen.js
│   ├── CartScreen.js
│   └── CheckoutScreen.js
├── components/
│   ├── ProductCard.js
│   └── CartIcon.js
└── data/
    └── products.js
```

### Data

```javascript
// data/products.js
export const categories = [
  { id: 1, name: 'มือถือ', icon: '📱' },
  { id: 2, name: 'แล็ปท็อป', icon: '💻' },
  { id: 3, name: 'หูฟัง', icon: '🎧' },
];

export const products = [
  {
    id: 1,
    categoryId: 1,
    name: 'iPhone 15 Pro',
    price: 48900,
    originalPrice: 52900,
    rating: 4.8,
    reviews: 2341,
    image: 'https://picsum.photos/id/1/400/400',
    description: 'iPhone 15 Pro พร้อม chip A17 Pro รองรับ USB-C และ Action Button',
    specs: ['A17 Pro chip', '6.1 inch display', '48MP camera', 'USB-C'],
    stock: 50,
  },
  {
    id: 2,
    categoryId: 1,
    name: 'Samsung Galaxy S24',
    price: 35900,
    originalPrice: 39900,
    rating: 4.6,
    reviews: 1523,
    image: 'https://picsum.photos/id/2/400/400',
    description: 'Samsung Galaxy S24 มาพร้อมกับ AI features และกล้อง 50MP',
    specs: ['Snapdragon 8 Gen 3', '6.2 inch display', '50MP camera', '4000mAh battery'],
    stock: 35,
  },
  {
    id: 3,
    categoryId: 2,
    name: 'MacBook Air M3',
    price: 44900,
    originalPrice: 49900,
    rating: 4.9,
    reviews: 3102,
    image: 'https://picsum.photos/id/3/400/400',
    description: 'MacBook Air ที่บางที่สุดพร้อม M3 chip ประสิทธิภาพสูง',
    specs: ['M3 chip', '13.6 inch Retina', '8GB RAM', '256GB SSD'],
    stock: 20,
  },
];
```

### HomeScreen

```javascript
// screens/HomeScreen.js
import React from 'react';
import {
  View, Text, StyleSheet, ScrollView,
  TouchableOpacity, Image, FlatList
} from 'react-native';
import { categories, products } from '../data/products';

export default function HomeScreen({ navigation }) {
  const featuredProducts = products.slice(0, 4);

  return (
    <ScrollView style={styles.container}>
      {/* Banner */}
      <View style={styles.banner}>
        <Text style={styles.bannerTitle}>🛍️ ยินดีต้อนรับ!</Text>
        <Text style={styles.bannerSubtitle}>สินค้าคุณภาพราคาดีทุกวัน</Text>
      </View>

      {/* Categories */}
      <View style={styles.section}>
        <Text style={styles.sectionTitle}>หมวดหมู่</Text>
        <FlatList
          horizontal
          showsHorizontalScrollIndicator={false}
          data={categories}
          keyExtractor={item => item.id.toString()}
          renderItem={({ item }) => (
            <TouchableOpacity
              style={styles.categoryItem}
              onPress={() => navigation.navigate('ProductList', { 
                categoryId: item.id, 
                categoryName: item.name 
              })}
            >
              <Text style={styles.categoryIcon}>{item.icon}</Text>
              <Text style={styles.categoryName}>{item.name}</Text>
            </TouchableOpacity>
          )}
        />
      </View>

      {/* Featured Products */}
      <View style={styles.section}>
        <View style={styles.sectionHeader}>
          <Text style={styles.sectionTitle}>สินค้าแนะนำ</Text>
          <TouchableOpacity onPress={() => navigation.navigate('ProductList', {})}>
            <Text style={styles.seeAll}>ดูทั้งหมด →</Text>
          </TouchableOpacity>
        </View>
        {featuredProducts.map(product => (
          <TouchableOpacity
            key={product.id}
            style={styles.productCard}
            onPress={() => navigation.navigate('ProductDetail', { product })}
          >
            <Image source={{ uri: product.image }} style={styles.productImage} />
            <View style={styles.productInfo}>
              <Text style={styles.productName} numberOfLines={2}>{product.name}</Text>
              <Text style={styles.productPrice}>฿{product.price.toLocaleString()}</Text>
              <Text style={styles.originalPrice}>฿{product.originalPrice.toLocaleString()}</Text>
              <Text style={styles.rating}>⭐ {product.rating} ({product.reviews})</Text>
            </View>
          </TouchableOpacity>
        ))}
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  banner: {
    backgroundColor: '#6200EE',
    padding: 24,
    alignItems: 'center',
  },
  bannerTitle: { fontSize: 28, fontWeight: 'bold', color: '#FFF' },
  bannerSubtitle: { fontSize: 16, color: '#E0D7FF', marginTop: 4 },
  section: { padding: 16 },
  sectionHeader: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 12 },
  sectionTitle: { fontSize: 20, fontWeight: 'bold', color: '#212121', marginBottom: 12 },
  seeAll: { color: '#6200EE', fontSize: 14 },
  categoryItem: {
    alignItems: 'center',
    backgroundColor: '#FFF',
    borderRadius: 12,
    padding: 16,
    marginRight: 12,
    width: 90,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
    elevation: 2,
  },
  categoryIcon: { fontSize: 28 },
  categoryName: { fontSize: 12, marginTop: 6, color: '#555' },
  productCard: {
    flexDirection: 'row',
    backgroundColor: '#FFF',
    borderRadius: 12,
    marginBottom: 12,
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
    elevation: 2,
  },
  productImage: { width: 100, height: 100 },
  productInfo: { flex: 1, padding: 12 },
  productName: { fontSize: 15, fontWeight: '600', color: '#212121' },
  productPrice: { fontSize: 18, fontWeight: 'bold', color: '#6200EE', marginTop: 4 },
  originalPrice: { fontSize: 13, color: '#999', textDecorationLine: 'line-through' },
  rating: { fontSize: 13, color: '#FFA000', marginTop: 4 },
});
```

### ProductDetailScreen

```javascript
// screens/ProductDetailScreen.js
import React, { useState, useLayoutEffect } from 'react';
import {
  View, Text, StyleSheet, ScrollView,
  Image, TouchableOpacity, Alert
} from 'react-native';

export default function ProductDetailScreen({ route, navigation }) {
  const { product } = route.params;
  const [quantity, setQuantity] = useState(1);
  const [cartCount, setCartCount] = useState(0);

  useLayoutEffect(() => {
    navigation.setOptions({
      title: product.name,
      headerRight: () => (
        <TouchableOpacity 
          style={{ marginRight: 15 }}
          onPress={() => navigation.navigate('Cart')}
        >
          <Text style={{ fontSize: 24 }}>🛒</Text>
          {cartCount > 0 && (
            <View style={styles.badge}>
              <Text style={styles.badgeText}>{cartCount}</Text>
            </View>
          )}
        </TouchableOpacity>
      ),
    });
  }, [navigation, product.name, cartCount]);

  const addToCart = () => {
    setCartCount(prev => prev + quantity);
    Alert.alert(
      'เพิ่มในตะกร้าแล้ว',
      `${product.name} x${quantity} ถูกเพิ่มในตะกร้าแล้ว`,
      [
        { text: 'ช้อปต่อ' },
        { text: 'ดูตะกร้า', onPress: () => navigation.navigate('Cart') }
      ]
    );
  };

  const discount = Math.round((1 - product.price / product.originalPrice) * 100);

  return (
    <View style={styles.container}>
      <ScrollView>
        {/* Product Image */}
        <Image source={{ uri: product.image }} style={styles.image} />
        
        {/* Discount Badge */}
        {discount > 0 && (
          <View style={styles.discountBadge}>
            <Text style={styles.discountText}>ลด {discount}%</Text>
          </View>
        )}

        <View style={styles.content}>
          {/* Name and Rating */}
          <Text style={styles.name}>{product.name}</Text>
          <View style={styles.ratingRow}>
            <Text style={styles.rating}>⭐ {product.rating}</Text>
            <Text style={styles.reviews}>({product.reviews} รีวิว)</Text>
          </View>

          {/* Price */}
          <View style={styles.priceRow}>
            <Text style={styles.price}>฿{product.price.toLocaleString()}</Text>
            <Text style={styles.originalPrice}>฿{product.originalPrice.toLocaleString()}</Text>
          </View>

          {/* Stock */}
          <Text style={styles.stock}>
            {product.stock > 0 ? `✅ มีสินค้า (${product.stock} ชิ้น)` : '❌ สินค้าหมด'}
          </Text>

          {/* Description */}
          <Text style={styles.sectionTitle}>รายละเอียดสินค้า</Text>
          <Text style={styles.description}>{product.description}</Text>

          {/* Specs */}
          <Text style={styles.sectionTitle}>สเปคหลัก</Text>
          {product.specs.map((spec, index) => (
            <View key={index} style={styles.specItem}>
              <Text style={styles.specBullet}>•</Text>
              <Text style={styles.specText}>{spec}</Text>
            </View>
          ))}

          {/* Quantity Selector */}
          <Text style={styles.sectionTitle}>จำนวน</Text>
          <View style={styles.quantityRow}>
            <TouchableOpacity
              style={styles.qtyBtn}
              onPress={() => setQuantity(Math.max(1, quantity - 1))}
            >
              <Text style={styles.qtyBtnText}>-</Text>
            </TouchableOpacity>
            <Text style={styles.quantity}>{quantity}</Text>
            <TouchableOpacity
              style={styles.qtyBtn}
              onPress={() => setQuantity(Math.min(product.stock, quantity + 1))}
            >
              <Text style={styles.qtyBtnText}>+</Text>
            </TouchableOpacity>
          </View>
        </View>
      </ScrollView>

      {/* Bottom Action Bar */}
      <View style={styles.actionBar}>
        <View>
          <Text style={styles.totalLabel}>ยอดรวม</Text>
          <Text style={styles.totalPrice}>฿{(product.price * quantity).toLocaleString()}</Text>
        </View>
        <TouchableOpacity
          style={[styles.addButton, product.stock === 0 && styles.disabledButton]}
          onPress={addToCart}
          disabled={product.stock === 0}
        >
          <Text style={styles.addButtonText}>🛒 เพิ่มในตะกร้า</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  image: { width: '100%', height: 300 },
  discountBadge: {
    position: 'absolute',
    top: 16,
    left: 16,
    backgroundColor: '#E53935',
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 6,
  },
  discountText: { color: '#FFF', fontWeight: 'bold', fontSize: 14 },
  content: { padding: 16, backgroundColor: '#FFF' },
  name: { fontSize: 22, fontWeight: 'bold', color: '#212121' },
  ratingRow: { flexDirection: 'row', alignItems: 'center', marginTop: 4 },
  rating: { color: '#FFA000', fontSize: 16 },
  reviews: { color: '#888', fontSize: 14, marginLeft: 6 },
  priceRow: { flexDirection: 'row', alignItems: 'baseline', marginTop: 8 },
  price: { fontSize: 28, fontWeight: 'bold', color: '#6200EE' },
  originalPrice: { 
    fontSize: 16, color: '#999', 
    textDecorationLine: 'line-through', 
    marginLeft: 8 
  },
  stock: { marginTop: 8, fontSize: 14, color: '#388E3C' },
  sectionTitle: { 
    fontSize: 18, fontWeight: 'bold', 
    color: '#212121', marginTop: 16, marginBottom: 8 
  },
  description: { fontSize: 15, color: '#555', lineHeight: 22 },
  specItem: { flexDirection: 'row', marginBottom: 4 },
  specBullet: { color: '#6200EE', marginRight: 8, fontSize: 15 },
  specText: { fontSize: 15, color: '#555' },
  quantityRow: { 
    flexDirection: 'row', 
    alignItems: 'center', 
    marginTop: 8,
    marginBottom: 80,
  },
  qtyBtn: {
    width: 40, height: 40,
    backgroundColor: '#6200EE',
    borderRadius: 20,
    alignItems: 'center',
    justifyContent: 'center',
  },
  qtyBtnText: { color: '#FFF', fontSize: 20, fontWeight: 'bold' },
  quantity: { fontSize: 20, fontWeight: 'bold', marginHorizontal: 20 },
  actionBar: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: '#FFF',
    padding: 16,
    borderTopWidth: 1,
    borderTopColor: '#EEE',
  },
  totalLabel: { fontSize: 13, color: '#666' },
  totalPrice: { fontSize: 20, fontWeight: 'bold', color: '#6200EE' },
  addButton: {
    backgroundColor: '#6200EE',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 10,
  },
  disabledButton: { backgroundColor: '#CCC' },
  addButtonText: { color: '#FFF', fontSize: 16, fontWeight: 'bold' },
  badge: {
    position: 'absolute',
    top: -5,
    right: -5,
    backgroundColor: '#E53935',
    width: 18,
    height: 18,
    borderRadius: 9,
    alignItems: 'center',
    justifyContent: 'center',
  },
  badgeText: { color: '#FFF', fontSize: 10, fontWeight: 'bold' },
});
```

### AppNavigator

```javascript
// navigation/AppNavigator.js
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import HomeScreen from '../screens/HomeScreen';
import ProductListScreen from '../screens/ProductListScreen';
import ProductDetailScreen from '../screens/ProductDetailScreen';
import CartScreen from '../screens/CartScreen';
import CheckoutScreen from '../screens/CheckoutScreen';

const Stack = createNativeStackNavigator();

export default function AppNavigator() {
  return (
    <Stack.Navigator
      initialRouteName="Home"
      screenOptions={{
        headerStyle: { backgroundColor: '#6200EE' },
        headerTintColor: '#FFF',
        headerTitleStyle: { fontWeight: 'bold' },
      }}
    >
      <Stack.Screen 
        name="Home" 
        component={HomeScreen}
        options={{ title: '🛍️ ร้านค้า' }}
      />
      <Stack.Screen 
        name="ProductList" 
        component={ProductListScreen}
        options={({ route }) => ({ 
          title: route.params?.categoryName || 'สินค้าทั้งหมด' 
        })}
      />
      <Stack.Screen 
        name="ProductDetail" 
        component={ProductDetailScreen}
        options={({ route }) => ({ 
          title: route.params?.product?.name || 'รายละเอียดสินค้า' 
        })}
      />
      <Stack.Screen 
        name="Cart" 
        component={CartScreen}
        options={{ title: 'ตะกร้าสินค้า' }}
      />
      <Stack.Screen 
        name="Checkout" 
        component={CheckoutScreen}
        options={{ 
          title: 'ชำระเงิน',
          presentation: 'modal',
        }}
      />
    </Stack.Navigator>
  );
}
```

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ NativeStack แทน Stack ทุกครั้ง (performance ดีกว่า)
- กำหนด initialRouteName อย่างชัดเจน
- ใช้ TypeScript กำหนด type ของ navigation params
- ใช้ useLayoutEffect สำหรับ setOptions เพื่อหลีกเลี่ยง flickering
- จัดกลุ่ม Screen ที่เกี่ยวข้องกันใน Navigator เดียวกัน

❌ DON'T:
- ไม่ pass object ขนาดใหญ่เป็น navigation params - ใช้ ID แล้ว fetch ข้อมูล
- ไม่ใช้ navigate() ใน constructor หรือ render phase
- ไม่ลืม handle กรณีที่ goBack() ไม่ได้ (first screen)
- ไม่ซ้อน Stack ใน Stack โดยไม่จำเป็น

📝 Pattern ที่แนะนำ:
- pass minimal data เป็น params (ID เท่านั้น)
- fetch ข้อมูลจาก ID ใน destination screen
- ใช้ global state (Redux/Context) สำหรับ shared data
```

---

## สรุป

Stack Navigator คือพื้นฐานของ navigation ใน React Native โดยเราได้เรียนรู้:

1. **createNativeStackNavigator** - วิธีสร้างและใช้งาน
2. **Screen Options** - การปรับแต่ง header และ animation
3. **Navigation Params** - การส่งและรับข้อมูลระหว่างหน้า
4. **Navigation Methods** - navigate, push, goBack, replace, reset
5. **Nested Navigators** - การซ้อน navigator
6. **Workshop** - E-commerce app จริง
