# Part 018: Hooks: useCallback, useMemo, useRef

## สารบัญ
1. useCallback: Memoize Functions
2. useMemo: Memoize Values
3. useRef: DOM References และ Mutable Values
4. useReducer: Complex State
5. Workshop: Performance Optimization

---

## 1. useCallback: Memoize Functions

`useCallback` ใช้สำหรับ memoize function เพื่อป้องกันการสร้าง function ใหม่ทุก render

### ปัญหาที่ useCallback แก้

```javascript
// ❌ ปัญหา: handlePress สร้างใหม่ทุก render
// ทำให้ ButtonComponent re-render ทุกครั้ง แม้ props ไม่เปลี่ยน
function ParentComponent() {
  const [count, setCount] = useState(0);
  
  const handlePress = () => {
    console.log('กด!');
  };
  
  return (
    <View>
      <Text>{count}</Text>
      <Button title="เพิ่ม" onPress={() => setCount(c => c + 1)} />
      {/* ButtonComponent re-render ทุกครั้งที่ count เปลี่ยน */}
      <MemoizedButton onPress={handlePress} />
    </View>
  );
}
```

```javascript
// ✅ แก้ด้วย useCallback
function ParentComponent() {
  const [count, setCount] = useState(0);
  
  // handlePress จะเป็น reference เดิมตลอด (เพราะ dependencies ว่าง)
  const handlePress = useCallback(() => {
    console.log('กด!');
  }, []);  // ไม่มี dependencies = สร้างครั้งเดียว
  
  return (
    <View>
      <Text>{count}</Text>
      <Button title="เพิ่ม" onPress={() => setCount(c => c + 1)} />
      {/* MemoizedButton ไม่ re-render เพราะ handlePress ไม่เปลี่ยน */}
      <MemoizedButton onPress={handlePress} />
    </View>
  );
}

const MemoizedButton = React.memo(({ onPress }) => {
  console.log('Button rendered');
  return <Button title="กด!" onPress={onPress} />;
});
```

### useCallback กับ Dependencies

```javascript
function SearchComponent({ userId }) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  // สร้างใหม่เมื่อ userId หรือ query เปลี่ยน
  const search = useCallback(async () => {
    const response = await fetch(
      `https://api.example.com/search?userId=${userId}&q=${query}`
    );
    const data = await response.json();
    setResults(data);
  }, [userId, query]);

  useEffect(() => {
    if (query) search();
  }, [search]);  // ✅ ใช้ function เป็น dependency ได้

  return (
    <View>
      <TextInput value={query} onChangeText={setQuery} />
      <FlatList data={results} renderItem={...} />
    </View>
  );
}
```

### useCallback กับ useEffect

```javascript
function DataTable({ filters }) {
  const [data, setData] = useState([]);

  // memoize fetch function
  const fetchData = useCallback(async () => {
    const response = await fetch(`/api/data?${new URLSearchParams(filters)}`);
    const json = await response.json();
    setData(json);
  }, [filters]);  // สร้างใหม่เมื่อ filters เปลี่ยน

  useEffect(() => {
    fetchData();
  }, [fetchData]);  // fetchData เปลี่ยน = effect ทำงานใหม่

  return <FlatList data={data} renderItem={...} />;
}
```

### เมื่อไหร่ควรใช้ useCallback

```javascript
// ✅ ใช้เมื่อ:
// 1. ส่ง function ให้ memoized child component
const handleSubmit = useCallback(() => {
  processForm(formData);
}, [formData]);

// 2. ใช้ function เป็น useEffect dependency
const loadData = useCallback(() => {
  fetchPosts(page);
}, [page]);

// 3. ฟังก์ชันที่ใช้เวลาสร้างนานหรือมี side effects

// ❌ ไม่จำเป็นเมื่อ:
// 1. Component ธรรมดาที่ไม่ใช้ React.memo
// 2. ฟังก์ชันง่ายๆ ที่เรียกใช้ครั้งเดียว
```

---

## 2. useMemo: Memoize Values

`useMemo` ใช้สำหรับ memoize ผลลัพธ์ของการคำนวณ เพื่อหลีกเลี่ยงการคำนวณซ้ำโดยไม่จำเป็น

### พื้นฐาน

```javascript
import React, { useMemo, useState } from 'react';

function ProductList({ products, searchQuery, sortBy }) {
  // ❌ ไม่ดี: คำนวณซ้ำทุก render แม้ products ไม่เปลี่ยน
  const filteredAndSortedProducts = products
    .filter(p => p.name.toLowerCase().includes(searchQuery.toLowerCase()))
    .sort((a, b) => {
      if (sortBy === 'price') return a.price - b.price;
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return 0;
    });

  // ✅ ดี: คำนวณเฉพาะเมื่อ dependencies เปลี่ยน
  const processedProducts = useMemo(() => {
    return products
      .filter(p => p.name.toLowerCase().includes(searchQuery.toLowerCase()))
      .sort((a, b) => {
        if (sortBy === 'price') return a.price - b.price;
        if (sortBy === 'name') return a.name.localeCompare(b.name);
        return 0;
      });
  }, [products, searchQuery, sortBy]);

  return (
    <FlatList
      data={processedProducts}
      renderItem={({ item }) => <ProductCard product={item} />}
    />
  );
}
```

### useMemo กับการคำนวณ Expensive

```javascript
function Analytics({ transactions }) {
  // การคำนวณสถิติที่ใช้เวลานาน - memoize เพื่อประสิทธิภาพ
  const stats = useMemo(() => {
    if (!transactions?.length) return null;
    
    const total = transactions.reduce((sum, t) => sum + t.amount, 0);
    const average = total / transactions.length;
    const max = Math.max(...transactions.map(t => t.amount));
    const min = Math.min(...transactions.map(t => t.amount));
    
    // จัดกลุ่มตามเดือน
    const byMonth = transactions.reduce((acc, t) => {
      const month = new Date(t.date).toLocaleString('th', { month: 'long', year: 'numeric' });
      acc[month] = (acc[month] || 0) + t.amount;
      return acc;
    }, {});
    
    return { total, average, max, min, byMonth };
  }, [transactions]);

  if (!stats) return <Text>ไม่มีข้อมูล</Text>;

  return (
    <View>
      <Text>ยอดรวม: {stats.total.toLocaleString()}</Text>
      <Text>เฉลี่ย: {stats.average.toFixed(2)}</Text>
      <Text>สูงสุด: {stats.max.toLocaleString()}</Text>
    </View>
  );
}
```

### useMemo สำหรับ Object ที่ใช้เป็น Props

```javascript
function Map({ center, zoom }) {
  // ❌ ปัญหา: config object สร้างใหม่ทุก render
  // MapComponent จะ re-render ทุกครั้ง แม้ค่าไม่เปลี่ยน
  const mapConfig = {
    center: { lat: center.lat, lng: center.lng },
    zoom,
    style: 'streets',
  };

  // ✅ memoize config
  const mapConfig = useMemo(() => ({
    center: { lat: center.lat, lng: center.lng },
    zoom,
    style: 'streets',
  }), [center.lat, center.lng, zoom]);

  return <MapComponent config={mapConfig} />;
}
```

---

## 3. useRef: DOM References และ Mutable Values

`useRef` มีสองการใช้งานหลัก:
1. อ้างอิง DOM element / component
2. เก็บค่าที่ไม่ต้องการ trigger re-render

### 3.1 อ้างอิง Element

```javascript
import React, { useRef } from 'react';
import { TextInput, Button, View, Text } from 'react-native';

function LoginForm() {
  const emailRef = useRef(null);
  const passwordRef = useRef(null);

  const focusPassword = () => {
    passwordRef.current?.focus();
  };

  const clearForm = () => {
    emailRef.current?.clear();
    passwordRef.current?.clear();
    emailRef.current?.focus();
  };

  return (
    <View>
      <TextInput
        ref={emailRef}
        placeholder="อีเมล"
        returnKeyType="next"
        onSubmitEditing={focusPassword}  // กด Enter ไปช่อง password
        keyboardType="email-address"
        autoCapitalize="none"
      />
      <TextInput
        ref={passwordRef}
        placeholder="รหัสผ่าน"
        secureTextEntry
        returnKeyType="done"
      />
      <Button title="ล้างฟอร์ม" onPress={clearForm} />
    </View>
  );
}
```

### 3.2 FlatList ref

```javascript
function ChatScreen({ messages }) {
  const flatListRef = useRef(null);

  // Scroll ไปล่างสุดเมื่อมีข้อความใหม่
  useEffect(() => {
    if (messages.length > 0) {
      flatListRef.current?.scrollToEnd({ animated: true });
    }
  }, [messages]);

  return (
    <FlatList
      ref={flatListRef}
      data={messages}
      keyExtractor={item => item.id}
      renderItem={({ item }) => <MessageBubble message={item} />}
    />
  );
}
```

### 3.3 Mutable Values ที่ไม่ trigger re-render

```javascript
// useRef สำหรับ mutable value ที่เปลี่ยนได้แต่ไม่ต้องการ re-render

function VideoPlayer({ onProgress }) {
  const intervalRef = useRef(null);
  const currentTimeRef = useRef(0);
  const [isPlaying, setIsPlaying] = useState(false);

  const startTimer = () => {
    intervalRef.current = setInterval(() => {
      currentTimeRef.current += 1;
      onProgress(currentTimeRef.current);
    }, 1000);
  };

  const stopTimer = () => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };

  const togglePlay = () => {
    if (isPlaying) {
      stopTimer();
    } else {
      startTimer();
    }
    setIsPlaying(!isPlaying);
  };

  // ล้าง interval เมื่อ unmount
  useEffect(() => {
    return () => stopTimer();
  }, []);

  return (
    <Button title={isPlaying ? 'หยุด' : 'เล่น'} onPress={togglePlay} />
  );
}
```

### 3.4 isMounted Pattern

```javascript
function DataLoader() {
  const isMountedRef = useRef(true);
  const [data, setData] = useState(null);

  useEffect(() => {
    isMountedRef.current = true;
    
    fetchData().then(result => {
      // ตรวจสอบก่อน setState
      if (isMountedRef.current) {
        setData(result);
      }
    });

    return () => {
      isMountedRef.current = false;
    };
  }, []);

  return <View>{/* ... */}</View>;
}
```

### 3.5 Previous Value Pattern

```javascript
function useCompare(value) {
  const prevValueRef = useRef(value);
  const prevValue = prevValueRef.current;
  prevValueRef.current = value;
  return prevValue !== value;
}

function PriceDisplay({ price }) {
  const isPriceChanged = useCompare(price);
  const prevPrice = useRef(price);
  
  useEffect(() => {
    if (isPriceChanged) {
      console.log(`ราคาเปลี่ยน: ${prevPrice.current} → ${price}`);
      prevPrice.current = price;
    }
  });

  return (
    <Text style={{ color: isPriceChanged ? '#E53935' : '#212121' }}>
      ฿{price.toLocaleString()}
    </Text>
  );
}
```

---

## 4. useReducer: Complex State

`useReducer` เหมาะสำหรับ state ที่มีความซับซ้อน มีหลาย state ที่เกี่ยวข้องกัน หรือมี state transitions ที่ซับซ้อน

### พื้นฐาน

```javascript
import React, { useReducer } from 'react';

// State shape
const initialState = {
  count: 0,
  step: 1,
};

// Reducer function: รับ state + action → คืน state ใหม่
function counterReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + state.step };
    case 'DECREMENT':
      return { ...state, count: state.count - state.step };
    case 'RESET':
      return initialState;
    case 'SET_STEP':
      return { ...state, step: action.payload };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, initialState);

  return (
    <View>
      <Text>จำนวน: {state.count}</Text>
      <Text>Step: {state.step}</Text>
      
      <Button title="เพิ่ม" onPress={() => dispatch({ type: 'INCREMENT' })} />
      <Button title="ลด" onPress={() => dispatch({ type: 'DECREMENT' })} />
      <Button title="รีเซ็ต" onPress={() => dispatch({ type: 'RESET' })} />
      
      <Button 
        title="Step = 5" 
        onPress={() => dispatch({ type: 'SET_STEP', payload: 5 })} 
      />
    </View>
  );
}
```

### Shopping Cart กับ useReducer

```javascript
const cartInitialState = {
  items: [],
  total: 0,
  itemCount: 0,
};

function cartReducer(state, action) {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existingItem = state.items.find(i => i.id === action.payload.id);
      
      if (existingItem) {
        const updatedItems = state.items.map(i =>
          i.id === action.payload.id
            ? { ...i, quantity: i.quantity + 1 }
            : i
        );
        return {
          ...state,
          items: updatedItems,
          total: state.total + action.payload.price,
          itemCount: state.itemCount + 1,
        };
      }
      
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
        total: state.total + action.payload.price,
        itemCount: state.itemCount + 1,
      };
    }
    
    case 'REMOVE_ITEM': {
      const item = state.items.find(i => i.id === action.payload);
      const updatedItems = state.items.filter(i => i.id !== action.payload);
      
      return {
        ...state,
        items: updatedItems,
        total: state.total - (item ? item.price * item.quantity : 0),
        itemCount: state.itemCount - (item ? item.quantity : 0),
      };
    }
    
    case 'UPDATE_QUANTITY': {
      const { id, quantity } = action.payload;
      const item = state.items.find(i => i.id === id);
      
      if (!item) return state;
      
      const diff = quantity - item.quantity;
      const updatedItems = quantity === 0
        ? state.items.filter(i => i.id !== id)
        : state.items.map(i => i.id === id ? { ...i, quantity } : i);
      
      return {
        ...state,
        items: updatedItems,
        total: state.total + (diff * item.price),
        itemCount: state.itemCount + diff,
      };
    }
    
    case 'CLEAR_CART':
      return cartInitialState;
      
    default:
      return state;
  }
}

function useCart() {
  const [cart, dispatch] = useReducer(cartReducer, cartInitialState);

  const addItem = (item) => dispatch({ type: 'ADD_ITEM', payload: item });
  const removeItem = (id) => dispatch({ type: 'REMOVE_ITEM', payload: id });
  const updateQuantity = (id, quantity) => dispatch({ 
    type: 'UPDATE_QUANTITY', 
    payload: { id, quantity } 
  });
  const clearCart = () => dispatch({ type: 'CLEAR_CART' });

  return { cart, addItem, removeItem, updateQuantity, clearCart };
}
```

---

## Workshop: Performance Optimization

### โครงสร้างแอป

```
PerfApp/
├── App.js
├── components/
│   ├── ExpensiveList.js
│   ├── FilterBar.js
│   └── ProductCard.js
├── hooks/
│   └── useCart.js
└── screens/
    ├── ShopScreen.js
    └── CartScreen.js
```

### ShopScreen - ปรับปรุง Performance

```javascript
// screens/ShopScreen.js
import React, { useState, useMemo, useCallback, useRef } from 'react';
import {
  View, Text, FlatList, StyleSheet,
  TextInput, TouchableOpacity
} from 'react-native';

// Mock data - สินค้า 100 ชิ้น
const generateProducts = () => Array.from({ length: 100 }, (_, i) => ({
  id: i + 1,
  name: `สินค้า ${i + 1}`,
  price: Math.floor(Math.random() * 10000) + 100,
  category: ['อาหาร', 'เครื่องดื่ม', 'เสื้อผ้า', 'อิเล็กทรอนิกส์'][i % 4],
  rating: (Math.random() * 2 + 3).toFixed(1),
  inStock: Math.random() > 0.2,
}));

const ALL_PRODUCTS = generateProducts();

// ProductCard memoized
const ProductCard = React.memo(({ product, onAddToCart }) => {
  console.log(`Rendering: ${product.name}`);  // ดู re-render
  
  return (
    <View style={styles.card}>
      <Text style={styles.productName}>{product.name}</Text>
      <Text style={styles.productPrice}>฿{product.price.toLocaleString()}</Text>
      <Text style={styles.category}>{product.category}</Text>
      <Text style={styles.rating}>⭐ {product.rating}</Text>
      <TouchableOpacity
        style={[styles.addBtn, !product.inStock && styles.disabledBtn]}
        onPress={() => onAddToCart(product)}
        disabled={!product.inStock}
      >
        <Text style={styles.addBtnText}>
          {product.inStock ? 'เพิ่มในตะกร้า' : 'สินค้าหมด'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}, (prevProps, nextProps) => {
  // Custom comparison - re-render เฉพาะเมื่อ props เปลี่ยนจริงๆ
  return (
    prevProps.product.id === nextProps.product.id &&
    prevProps.product.price === nextProps.product.price &&
    prevProps.product.inStock === nextProps.product.inStock &&
    prevProps.onAddToCart === nextProps.onAddToCart  // ต้องใช้ useCallback
  );
});

function ShopScreen({ navigation }) {
  const [searchText, setSearchText] = useState('');
  const [selectedCategory, setSelectedCategory] = useState('ทั้งหมด');
  const [sortBy, setSortBy] = useState('default');
  const [cartItems, setCartItems] = useState([]);
  const flatListRef = useRef(null);

  // useMemo: คำนวณสินค้าที่กรองแล้ว
  const filteredProducts = useMemo(() => {
    let result = ALL_PRODUCTS;
    
    // Filter by search
    if (searchText) {
      result = result.filter(p =>
        p.name.toLowerCase().includes(searchText.toLowerCase())
      );
    }
    
    // Filter by category
    if (selectedCategory !== 'ทั้งหมด') {
      result = result.filter(p => p.category === selectedCategory);
    }
    
    // Sort
    switch (sortBy) {
      case 'price_asc':
        result = [...result].sort((a, b) => a.price - b.price);
        break;
      case 'price_desc':
        result = [...result].sort((a, b) => b.price - a.price);
        break;
      case 'rating':
        result = [...result].sort((a, b) => b.rating - a.rating);
        break;
    }
    
    return result;
  }, [searchText, selectedCategory, sortBy]);

  // useMemo: คำนวณ categories
  const categories = useMemo(() => {
    return ['ทั้งหมด', ...new Set(ALL_PRODUCTS.map(p => p.category))];
  }, []);  // ไม่มี deps = คำนวณครั้งเดียว

  // useCallback: memoize handler เพื่อส่งให้ ProductCard
  const handleAddToCart = useCallback((product) => {
    setCartItems(prev => {
      const existing = prev.find(i => i.id === product.id);
      if (existing) {
        return prev.map(i =>
          i.id === product.id ? { ...i, qty: i.qty + 1 } : i
        );
      }
      return [...prev, { ...product, qty: 1 }];
    });
  }, []);  // ไม่มี deps = สร้างครั้งเดียว

  // useCallback: scroll to top
  const scrollToTop = useCallback(() => {
    flatListRef.current?.scrollToOffset({ offset: 0, animated: true });
  }, []);

  // useMemo: cart count (ไม่ต้อง recalculate ทุก render)
  const cartCount = useMemo(() => {
    return cartItems.reduce((sum, item) => sum + item.qty, 0);
  }, [cartItems]);

  return (
    <View style={styles.container}>
      {/* Header */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>ร้านค้า</Text>
        <TouchableOpacity
          style={styles.cartBtn}
          onPress={() => navigation.navigate('Cart', { cartItems })}
        >
          <Text style={styles.cartIcon}>🛒</Text>
          {cartCount > 0 && (
            <View style={styles.cartBadge}>
              <Text style={styles.cartBadgeText}>{cartCount}</Text>
            </View>
          )}
        </TouchableOpacity>
      </View>

      {/* Search */}
      <TextInput
        style={styles.search}
        placeholder="🔍 ค้นหาสินค้า..."
        value={searchText}
        onChangeText={setSearchText}
      />

      {/* Categories */}
      <FlatList
        horizontal
        data={categories}
        keyExtractor={cat => cat}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={[styles.catChip, selectedCategory === item && styles.activeCatChip]}
            onPress={() => setSelectedCategory(item)}
          >
            <Text style={[styles.catText, selectedCategory === item && styles.activeCatText]}>
              {item}
            </Text>
          </TouchableOpacity>
        )}
        contentContainerStyle={{ padding: 8 }}
        showsHorizontalScrollIndicator={false}
      />

      {/* Sort Options */}
      <View style={styles.sortRow}>
        <Text style={styles.sortLabel}>เรียงตาม:</Text>
        {[
          { value: 'default', label: 'ค่าเริ่มต้น' },
          { value: 'price_asc', label: 'ราคา ↑' },
          { value: 'price_desc', label: 'ราคา ↓' },
          { value: 'rating', label: 'คะแนน' },
        ].map(sort => (
          <TouchableOpacity
            key={sort.value}
            style={[styles.sortBtn, sortBy === sort.value && styles.activeSortBtn]}
            onPress={() => setSortBy(sort.value)}
          >
            <Text style={[styles.sortBtnText, sortBy === sort.value && styles.activeSortBtnText]}>
              {sort.label}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      {/* Results Count */}
      <Text style={styles.resultCount}>
        {filteredProducts.length} สินค้า
      </Text>

      {/* Product List */}
      <FlatList
        ref={flatListRef}
        data={filteredProducts}
        keyExtractor={item => item.id.toString()}
        renderItem={({ item }) => (
          <ProductCard
            product={item}
            onAddToCart={handleAddToCart}
          />
        )}
        numColumns={2}
        columnWrapperStyle={styles.row}
        contentContainerStyle={styles.productList}
        showsVerticalScrollIndicator={false}
        getItemLayout={(data, index) => ({
          length: 180,
          offset: 180 * Math.floor(index / 2),
          index,
        })}
        maxToRenderPerBatch={10}
        windowSize={10}
        removeClippedSubviews={true}
      />

      {/* Scroll to top */}
      <TouchableOpacity style={styles.scrollTopBtn} onPress={scrollToTop}>
        <Text style={styles.scrollTopText}>↑</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#6200EE',
  },
  headerTitle: { fontSize: 20, fontWeight: 'bold', color: '#FFF' },
  cartBtn: { position: 'relative' },
  cartIcon: { fontSize: 26 },
  cartBadge: {
    position: 'absolute',
    top: -5,
    right: -8,
    backgroundColor: '#E53935',
    borderRadius: 10,
    width: 20,
    height: 20,
    alignItems: 'center',
    justifyContent: 'center',
  },
  cartBadgeText: { color: '#FFF', fontSize: 11, fontWeight: 'bold' },
  search: {
    margin: 12,
    backgroundColor: '#FFF',
    borderRadius: 10,
    paddingHorizontal: 14,
    paddingVertical: 10,
    fontSize: 15,
  },
  catChip: {
    paddingHorizontal: 14,
    paddingVertical: 6,
    borderRadius: 20,
    backgroundColor: '#EEE',
    marginRight: 8,
  },
  activeCatChip: { backgroundColor: '#6200EE' },
  catText: { color: '#666', fontSize: 13 },
  activeCatText: { color: '#FFF' },
  sortRow: { flexDirection: 'row', alignItems: 'center', paddingHorizontal: 12, paddingBottom: 8 },
  sortLabel: { fontSize: 13, color: '#666', marginRight: 8 },
  sortBtn: {
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 12,
    backgroundColor: '#EEE',
    marginRight: 6,
  },
  activeSortBtn: { backgroundColor: '#6200EE' },
  sortBtnText: { fontSize: 12, color: '#666' },
  activeSortBtnText: { color: '#FFF' },
  resultCount: { paddingHorizontal: 12, paddingBottom: 4, fontSize: 13, color: '#999' },
  productList: { paddingHorizontal: 8, paddingBottom: 100 },
  row: { justifyContent: 'space-between', paddingHorizontal: 4 },
  card: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    padding: 12,
    marginBottom: 12,
    width: '48%',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.08,
    shadowRadius: 3,
    elevation: 2,
  },
  productName: { fontSize: 14, fontWeight: '600', color: '#212121', marginBottom: 4 },
  productPrice: { fontSize: 16, fontWeight: 'bold', color: '#6200EE' },
  category: { fontSize: 11, color: '#03DAC6', marginTop: 2 },
  rating: { fontSize: 12, color: '#FFA000', marginTop: 2 },
  addBtn: {
    backgroundColor: '#6200EE',
    padding: 8,
    borderRadius: 8,
    alignItems: 'center',
    marginTop: 8,
  },
  disabledBtn: { backgroundColor: '#CCC' },
  addBtnText: { color: '#FFF', fontSize: 12, fontWeight: '600' },
  scrollTopBtn: {
    position: 'absolute',
    bottom: 20,
    right: 20,
    backgroundColor: '#6200EE',
    width: 44,
    height: 44,
    borderRadius: 22,
    alignItems: 'center',
    justifyContent: 'center',
    shadowColor: '#6200EE',
    shadowOffset: { width: 0, height: 3 },
    shadowOpacity: 0.4,
    shadowRadius: 5,
    elevation: 5,
  },
  scrollTopText: { color: '#FFF', fontSize: 20, fontWeight: 'bold' },
});

export default ShopScreen;
```

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ React.memo + useCallback ด้วยกันเสมอ
- ใช้ useMemo สำหรับ expensive computations
- ใช้ useRef สำหรับ values ที่ไม่ต้องการ re-render
- Profile ก่อน optimize (ใช้ React DevTools)

❌ DON'T:
- ไม่ใช้ useMemo/useCallback กับทุกอย่าง
  (มี overhead ของ memoization เอง)
- ไม่ใช้ useRef แทน useState สำหรับ UI state
- ไม่ ignore exhaustive-deps ESLint warning

📊 เมื่อไหร่ควร Optimize:
1. Component re-renders บ่อย (ใช้ React DevTools profiler)
2. Expensive calculations (ใช้ Performance API วัดเวลา)
3. Large lists (ใช้ getItemLayout, removeClippedSubviews)
```

---

## สรุป

Advanced Hooks ช่วยเพิ่มประสิทธิภาพแอป:

1. **useCallback** - memoize functions ป้องกัน child re-render
2. **useMemo** - memoize computed values
3. **useRef** - reference elements และ mutable values
4. **useReducer** - จัดการ complex state
5. **Workshop** - Shop app ที่ optimize แล้ว
