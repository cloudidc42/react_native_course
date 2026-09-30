# Part 017: Hooks: useState และ useEffect

## สารบัญ
1. useState Patterns
2. useEffect: componentDidMount
3. useEffect: componentDidUpdate
4. useEffect: componentWillUnmount
5. Dependencies Array
6. Cleanup Function
7. Common Patterns
8. Workshop: Data Fetching with Hooks

---

## 1. useState Patterns

`useState` คือ Hook ที่ใช้จัดการ state ใน functional component

### พื้นฐาน

```javascript
import React, { useState } from 'react';

function Counter() {
  // ประกาศ state: [value, setter]
  const [count, setCount] = useState(0);  // 0 คือ initial value
  
  return (
    <View>
      <Text>จำนวน: {count}</Text>
      <Button title="เพิ่ม" onPress={() => setCount(count + 1)} />
      <Button title="ลด" onPress={() => setCount(count - 1)} />
      <Button title="รีเซ็ต" onPress={() => setCount(0)} />
    </View>
  );
}
```

### Functional Updates

```javascript
// ✅ ใช้ functional update เมื่อ state ใหม่ขึ้นอยู่กับ state เก่า
setCount(prevCount => prevCount + 1);

// ❌ ไม่ควรใช้ค่าปัจจุบันโดยตรงใน async context
// (อาจทำให้ค่าไม่ถูกต้องถ้า setState ถูกเรียกหลายครั้ง)
setCount(count + 1);  // ใช้ได้ใน event handlers ปกติ
```

### Object State

```javascript
function UserForm() {
  const [user, setUser] = useState({
    name: '',
    email: '',
    age: 0,
    address: {
      city: '',
      country: 'Thailand',
    },
  });

  // อัพเดท field เดียว
  const updateField = (field, value) => {
    setUser(prev => ({ ...prev, [field]: value }));
  };

  // อัพเดท nested object
  const updateAddress = (field, value) => {
    setUser(prev => ({
      ...prev,
      address: { ...prev.address, [field]: value },
    }));
  };

  return (
    <View>
      <TextInput
        value={user.name}
        onChangeText={(text) => updateField('name', text)}
        placeholder="ชื่อ"
      />
      <TextInput
        value={user.address.city}
        onChangeText={(text) => updateAddress('city', text)}
        placeholder="เมือง"
      />
    </View>
  );
}
```

### Array State

```javascript
function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'เรียน React Native', done: false },
    { id: 2, text: 'สร้าง app', done: false },
  ]);

  // เพิ่ม item
  const addTodo = (text) => {
    setTodos(prev => [
      ...prev,
      { id: Date.now(), text, done: false }
    ]);
  };

  // อัพเดท item
  const toggleTodo = (id) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      )
    );
  };

  // ลบ item
  const removeTodo = (id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  // ย้าย item (เรียงลำดับใหม่)
  const moveTodoUp = (id) => {
    setTodos(prev => {
      const index = prev.findIndex(t => t.id === id);
      if (index <= 0) return prev;
      const newTodos = [...prev];
      [newTodos[index - 1], newTodos[index]] = [newTodos[index], newTodos[index - 1]];
      return newTodos;
    });
  };

  return (
    <FlatList
      data={todos}
      keyExtractor={item => item.id.toString()}
      renderItem={({ item }) => (
        <View style={{ flexDirection: 'row', padding: 12 }}>
          <Switch value={item.done} onValueChange={() => toggleTodo(item.id)} />
          <Text style={{ 
            flex: 1,
            textDecorationLine: item.done ? 'line-through' : 'none',
            color: item.done ? '#999' : '#212121',
          }}>
            {item.text}
          </Text>
          <TouchableOpacity onPress={() => removeTodo(item.id)}>
            <Text style={{ color: '#E53935' }}>ลบ</Text>
          </TouchableOpacity>
        </View>
      )}
    />
  );
}
```

### Lazy Initial State

```javascript
// สำหรับ initial state ที่คำนวณยาก ใช้ function
const [data, setData] = useState(() => {
  // ฟังก์ชันนี้จะถูกเรียกแค่ครั้งเดียวตอน mount
  const savedData = loadFromStorage();  // expensive operation
  return savedData || defaultData;
});
```

### Multiple State vs Object State

```javascript
// ✅ Multiple states - ดีเมื่อ states ไม่เกี่ยวข้องกัน
const [name, setName] = useState('');
const [age, setAge] = useState(0);
const [isLoading, setIsLoading] = useState(false);

// ✅ Object state - ดีเมื่อ states เกี่ยวข้องกัน
const [formData, setFormData] = useState({
  name: '',
  age: 0,
  email: '',
});
```

---

## 2. useEffect: componentDidMount

`useEffect` ที่มี dependency array ว่างเปล่า `[]` จะทำงานแค่ครั้งเดียวตอน mount (เหมือน componentDidMount)

```javascript
import React, { useState, useEffect } from 'react';

function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  // ทำงานครั้งเดียวตอน component mount
  useEffect(() => {
    fetchUser();
  }, []);  // [] = ไม่มี dependencies = ทำงานครั้งเดียว

  const fetchUser = async () => {
    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/users/1');
      const data = await response.json();
      setUser(data);
    } catch (error) {
      console.error(error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <Text>กำลังโหลด...</Text>;
  
  return <Text>{user?.name}</Text>;
}
```

### ตัวอย่าง componentDidMount patterns

```javascript
function AppSetup() {
  useEffect(() => {
    // ตั้งค่า Analytics
    Analytics.init();
    
    // Subscribe to notifications
    const subscription = Notifications.addListener(handleNotification);
    
    // Request permissions
    requestCameraPermission();
    
    // Load saved data
    loadUserPreferences();
    
    // Cleanup (componentWillUnmount)
    return () => {
      subscription.remove();
    };
  }, []);
}
```

---

## 3. useEffect: componentDidUpdate

```javascript
// ทำงานทุกครั้งที่ dependencies เปลี่ยน
useEffect(() => {
  console.log('count เปลี่ยนแล้ว:', count);
}, [count]);  // ทำงานเมื่อ count เปลี่ยน

// ทำงานทุกครั้งที่ render (ไม่แนะนำ)
useEffect(() => {
  console.log('render!');
});  // ไม่มี dependency array
```

### ตัวอย่าง componentDidUpdate patterns

```javascript
function SearchScreen() {
  const [searchQuery, setSearchQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  // ค้นหาเมื่อ searchQuery เปลี่ยน
  useEffect(() => {
    if (searchQuery.length < 2) {
      setResults([]);
      return;
    }

    const searchTimer = setTimeout(async () => {
      setLoading(true);
      try {
        const response = await fetch(`https://api.example.com/search?q=${searchQuery}`);
        const data = await response.json();
        setResults(data);
      } catch (error) {
        console.error(error);
      } finally {
        setLoading(false);
      }
    }, 500);  // Debounce 500ms

    return () => clearTimeout(searchTimer);  // Cleanup timer
  }, [searchQuery]);

  return (
    <View>
      <TextInput
        value={searchQuery}
        onChangeText={setSearchQuery}
        placeholder="ค้นหา..."
      />
      {loading && <ActivityIndicator />}
      <FlatList
        data={results}
        keyExtractor={(item) => item.id.toString()}
        renderItem={({ item }) => <Text>{item.name}</Text>}
      />
    </View>
  );
}
```

### Multiple useEffect

```javascript
function ComplexScreen({ userId, categoryId }) {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  // Effect สำหรับ user - ทำงานเมื่อ userId เปลี่ยน
  useEffect(() => {
    fetchUser(userId);
  }, [userId]);

  // Effect สำหรับ posts - ทำงานเมื่อ categoryId เปลี่ยน
  useEffect(() => {
    fetchPosts(categoryId);
  }, [categoryId]);

  // Effect สำหรับ analytics - ทำงานทุกครั้งที่ render
  useEffect(() => {
    logPageView(userId, categoryId);
  });  // ระวัง: ไม่มี array = ทำงานทุก render

  // ✅ วิธีที่ถูกต้องกว่า
  useEffect(() => {
    logPageView(userId, categoryId);
  }, [userId, categoryId]);
}
```

---

## 4. useEffect: componentWillUnmount

```javascript
function TimerScreen() {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    if (!isRunning) return;

    const interval = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);

    // Cleanup function = componentWillUnmount
    // ทำงานเมื่อ:
    // 1. component unmount
    // 2. ก่อน effect ถัดไปทำงาน (เมื่อ dependency เปลี่ยน)
    return () => {
      clearInterval(interval);
      console.log('Timer cleared');
    };
  }, [isRunning]);  // ทำงานเมื่อ isRunning เปลี่ยน

  return (
    <View>
      <Text style={{ fontSize: 48 }}>{seconds}s</Text>
      <Button
        title={isRunning ? 'หยุด' : 'เริ่ม'}
        onPress={() => setIsRunning(!isRunning)}
      />
      <Button title="รีเซ็ต" onPress={() => { setIsRunning(false); setSeconds(0); }} />
    </View>
  );
}
```

---

## 5. Dependencies Array

```javascript
// ❌ Missing dependencies - ESLint จะเตือน
useEffect(() => {
  fetchData(userId);  // userId ถูกใช้แต่ไม่ได้อยู่ใน deps
}, []);

// ✅ ถูกต้อง
useEffect(() => {
  fetchData(userId);
}, [userId]);

// ✅ ถ้าต้องการ ignore - ใช้ comment
useEffect(() => {
  initApp();  // ต้องการเรียกครั้งเดียว แม้ deps เปลี่ยน
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []);
```

### ระวัง Object/Array ใน Dependencies

```javascript
// ❌ Object ใหม่ทุก render = infinite loop!
function BadExample() {
  const config = { theme: 'dark' };  // สร้างใหม่ทุก render
  
  useEffect(() => {
    applyConfig(config);
  }, [config]);  // เปลี่ยนทุก render!
}

// ✅ ใช้ useMemo หรือ extract ออกมานอก component
const config = { theme: 'dark' };  // สร้างครั้งเดียว

function GoodExample() {
  useEffect(() => {
    applyConfig(config);
  }, []);  // config ไม่เปลี่ยน
}

// หรือใช้ useMemo
function AnotherGoodExample({ theme }) {
  const config = useMemo(() => ({ theme }), [theme]);
  
  useEffect(() => {
    applyConfig(config);
  }, [config]);
}
```

---

## 6. Cleanup Function

```javascript
// 1. Cleanup subscription
useEffect(() => {
  const subscription = EventEmitter.subscribe('event', handler);
  return () => subscription.unsubscribe();
}, []);

// 2. Cleanup timer
useEffect(() => {
  const timer = setTimeout(callback, delay);
  return () => clearTimeout(timer);
}, [delay]);

// 3. Cancel fetch (AbortController)
useEffect(() => {
  const controller = new AbortController();
  
  fetch(url, { signal: controller.signal })
    .then(res => res.json())
    .then(setData)
    .catch(err => {
      if (err.name !== 'AbortError') setError(err);
    });
  
  return () => controller.abort();
}, [url]);

// 4. Animate cleanup
useEffect(() => {
  const animation = Animated.loop(
    Animated.timing(value, { toValue: 1, duration: 1000, useNativeDriver: true })
  );
  animation.start();
  return () => animation.stop();
}, []);
```

---

## 7. Common Patterns

### Pattern 1: Data Fetching

```javascript
function useData(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let isMounted = true;
    
    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        const json = await response.json();
        if (isMounted) setData(json);
      } catch (err) {
        if (isMounted) setError(err.message);
      } finally {
        if (isMounted) setLoading(false);
      }
    };

    fetchData();

    return () => { isMounted = false; };
  }, [url]);

  return { data, loading, error };
}

// ใช้งาน
function UserList() {
  const { data: users, loading, error } = useData('https://jsonplaceholder.typicode.com/users');
  
  if (loading) return <ActivityIndicator />;
  if (error) return <Text>Error: {error}</Text>;
  
  return (
    <FlatList
      data={users}
      keyExtractor={item => item.id.toString()}
      renderItem={({ item }) => <Text>{item.name}</Text>}
    />
  );
}
```

### Pattern 2: Previous Value

```javascript
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref.current;
}

// ใช้งาน
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  
  return (
    <View>
      <Text>ก่อนหน้า: {prevCount}</Text>
      <Text>ปัจจุบัน: {count}</Text>
      <Button title="เพิ่ม" onPress={() => setCount(c => c + 1)} />
    </View>
  );
}
```

### Pattern 3: Window Dimensions

```javascript
function useWindowSize() {
  const [size, setSize] = useState({
    width: Dimensions.get('window').width,
    height: Dimensions.get('window').height,
  });

  useEffect(() => {
    const subscription = Dimensions.addEventListener('change', ({ window }) => {
      setSize({ width: window.width, height: window.height });
    });
    
    return () => subscription?.remove();
  }, []);

  return size;
}
```

---

## Workshop: Data Fetching with Hooks

### โครงสร้าง

```
HooksApp/
├── App.js
├── hooks/
│   ├── useFetch.js
│   └── useDebounce.js
└── screens/
    ├── ProductsScreen.js
    └── ProductDetailScreen.js
```

### useFetch Hook

```javascript
// hooks/useFetch.js
import { useState, useEffect, useCallback } from 'react';

function useFetch(url, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [refetchIndex, setRefetchIndex] = useState(0);

  const refetch = useCallback(() => {
    setRefetchIndex(i => i + 1);
  }, []);

  useEffect(() => {
    if (!url) return;
    
    let isMounted = true;
    const controller = new AbortController();

    const fetchData = async () => {
      setLoading(true);
      setError(null);
      
      try {
        const response = await fetch(url, {
          ...options,
          signal: controller.signal,
        });
        
        if (!response.ok) {
          throw new Error(`HTTP Error: ${response.status}`);
        }
        
        const json = await response.json();
        
        if (isMounted) {
          setData(json);
        }
      } catch (err) {
        if (isMounted && err.name !== 'AbortError') {
          setError(err.message || 'เกิดข้อผิดพลาด');
        }
      } finally {
        if (isMounted) {
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      isMounted = false;
      controller.abort();
    };
  }, [url, refetchIndex]);

  return { data, loading, error, refetch };
}

export default useFetch;
```

### useDebounce Hook

```javascript
// hooks/useDebounce.js
import { useState, useEffect } from 'react';

function useDebounce(value, delay = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

export default useDebounce;
```

### ProductsScreen

```javascript
// screens/ProductsScreen.js
import React, { useState, useEffect } from 'react';
import {
  View, Text, FlatList, StyleSheet,
  TextInput, TouchableOpacity, ActivityIndicator
} from 'react-native';
import useFetch from '../hooks/useFetch';
import useDebounce from '../hooks/useDebounce';

function ProductCard({ product, onPress }) {
  return (
    <TouchableOpacity style={styles.card} onPress={() => onPress(product)}>
      <Text style={styles.cardTitle}>{product.title}</Text>
      <View style={styles.cardMeta}>
        <Text style={styles.cardPrice}>
          ฿{(product.price * 35).toLocaleString()}
        </Text>
        <Text style={styles.cardCategory}>{product.category}</Text>
      </View>
      <View style={styles.ratingRow}>
        <Text style={styles.rating}>⭐ {product.rating?.rate}</Text>
        <Text style={styles.ratingCount}>({product.rating?.count})</Text>
      </View>
    </TouchableOpacity>
  );
}

export default function ProductsScreen({ navigation }) {
  const [searchText, setSearchText] = useState('');
  const [selectedCategory, setSelectedCategory] = useState('all');
  
  const debouncedSearch = useDebounce(searchText, 300);
  
  const { data: products, loading, error, refetch } = useFetch(
    'https://fakestoreapi.com/products'
  );

  // กรองสินค้า
  const filteredProducts = products?.filter(product => {
    const matchSearch = product.title.toLowerCase()
      .includes(debouncedSearch.toLowerCase());
    const matchCategory = selectedCategory === 'all' || 
      product.category === selectedCategory;
    return matchSearch && matchCategory;
  }) || [];

  // หา categories ที่ไม่ซ้ำ
  const categories = products 
    ? ['all', ...new Set(products.map(p => p.category))]
    : ['all'];

  if (loading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
        <Text style={styles.loadingText}>กำลังโหลดสินค้า...</Text>
      </View>
    );
  }

  if (error) {
    return (
      <View style={styles.center}>
        <Text style={styles.errorText}>⚠️ {error}</Text>
        <TouchableOpacity style={styles.retryBtn} onPress={refetch}>
          <Text style={styles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      {/* Search */}
      <View style={styles.searchContainer}>
        <TextInput
          style={styles.searchInput}
          placeholder="🔍 ค้นหาสินค้า..."
          value={searchText}
          onChangeText={setSearchText}
          clearButtonMode="while-editing"
        />
      </View>

      {/* Categories */}
      <FlatList
        horizontal
        showsHorizontalScrollIndicator={false}
        data={categories}
        keyExtractor={item => item}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={[
              styles.categoryChip,
              selectedCategory === item && styles.activeCategoryChip
            ]}
            onPress={() => setSelectedCategory(item)}
          >
            <Text style={[
              styles.categoryChipText,
              selectedCategory === item && styles.activeCategoryChipText
            ]}>
              {item === 'all' ? 'ทั้งหมด' : item}
            </Text>
          </TouchableOpacity>
        )}
        contentContainerStyle={styles.categoryList}
        style={styles.categoryScroll}
      />

      {/* Results count */}
      <Text style={styles.resultCount}>
        พบ {filteredProducts.length} สินค้า
      </Text>

      {/* Products */}
      <FlatList
        data={filteredProducts}
        keyExtractor={item => item.id.toString()}
        renderItem={({ item }) => (
          <ProductCard
            product={item}
            onPress={(p) => navigation.navigate('ProductDetail', { product: p })}
          />
        )}
        numColumns={1}
        contentContainerStyle={styles.productList}
        ListEmptyComponent={() => (
          <View style={styles.empty}>
            <Text style={styles.emptyText}>🔍 ไม่พบสินค้าที่ค้นหา</Text>
          </View>
        )}
        showsVerticalScrollIndicator={false}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 20 },
  loadingText: { marginTop: 12, color: '#666', fontSize: 15 },
  errorText: { color: '#E53935', fontSize: 15, textAlign: 'center', marginBottom: 12 },
  retryBtn: {
    backgroundColor: '#6200EE',
    paddingHorizontal: 20,
    paddingVertical: 10,
    borderRadius: 8,
  },
  retryText: { color: '#FFF', fontWeight: '600' },
  searchContainer: { padding: 12, backgroundColor: '#FFF' },
  searchInput: {
    backgroundColor: '#F5F5F5',
    borderRadius: 10,
    paddingHorizontal: 14,
    paddingVertical: 10,
    fontSize: 15,
  },
  categoryScroll: { maxHeight: 50, backgroundColor: '#FFF' },
  categoryList: { paddingHorizontal: 12, paddingBottom: 8 },
  categoryChip: {
    paddingHorizontal: 14,
    paddingVertical: 6,
    borderRadius: 20,
    backgroundColor: '#F0F0F0',
    marginRight: 8,
  },
  activeCategoryChip: { backgroundColor: '#6200EE' },
  categoryChipText: { color: '#666', fontSize: 13 },
  activeCategoryChipText: { color: '#FFF' },
  resultCount: { padding: 12, fontSize: 13, color: '#999' },
  productList: { paddingHorizontal: 12, paddingBottom: 20 },
  card: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    padding: 14,
    marginBottom: 10,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.08,
    shadowRadius: 3,
    elevation: 2,
  },
  cardTitle: {
    fontSize: 15,
    fontWeight: '600',
    color: '#212121',
    marginBottom: 8,
  },
  cardMeta: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  cardPrice: { fontSize: 18, fontWeight: 'bold', color: '#6200EE' },
  cardCategory: {
    fontSize: 12,
    color: '#FFF',
    backgroundColor: '#03DAC6',
    paddingHorizontal: 8,
    paddingVertical: 3,
    borderRadius: 10,
  },
  ratingRow: { flexDirection: 'row', alignItems: 'center', marginTop: 6 },
  rating: { color: '#FFA000', fontSize: 13 },
  ratingCount: { color: '#999', fontSize: 12, marginLeft: 4 },
  empty: { alignItems: 'center', paddingTop: 60 },
  emptyText: { fontSize: 16, color: '#999' },
});
```

---

## Tips และ Best Practices

```
✅ DO:
- ใส่ dependencies ครบใน array
- ทำ cleanup เสมอสำหรับ subscriptions และ timers
- ใช้ functional update สำหรับ state ที่ขึ้นกับค่าเดิม
- ตรวจสอบ isMounted ก่อน setState ใน async ops

❌ DON'T:
- ไม่ใส่ object ที่สร้างใหม่ทุก render ใน dependencies
- ไม่ทำ setState ใน render (จะเป็น infinite loop)
- ไม่ลืม cleanup สำหรับ async operations

📝 Rules of Hooks:
1. เรียกใช้ Hooks ที่ top level เท่านั้น (ไม่ใน if, loop, nested function)
2. เรียกใช้ Hooks เฉพาะใน React Function Components หรือ Custom Hooks
```

---

## สรุป

useState และ useEffect เป็น Hooks พื้นฐานที่สำคัญที่สุด:

1. **useState** - จัดการ state: string, number, object, array
2. **useEffect ComponentDidMount** - ทำงานครั้งเดียวตอน mount
3. **useEffect ComponentDidUpdate** - ทำงานเมื่อ dependencies เปลี่ยน
4. **useEffect ComponentWillUnmount** - cleanup เมื่อ unmount
5. **Dependencies Array** - ควบคุมเวลาที่ effect ทำงาน
6. **Workshop** - Data fetching app ด้วย custom hooks
