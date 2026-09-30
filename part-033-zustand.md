# Part 033: Zustand - Modern State Management

## Zustand คืออะไร?

Zustand เป็น state management library ขนาดเล็ก (ประมาณ 1KB) ที่เรียบง่ายแต่ทรงพลัง พัฒนาโดยทีม pmndrs (ผู้สร้าง react-spring และ zustand)

### เปรียบเทียบ Zustand vs Redux Toolkit

| Feature | Redux Toolkit | Zustand |
|---------|--------------|---------|
| Bundle size | ~12KB | ~1KB |
| Boilerplate | Medium | Minimal |
| DevTools | Excellent | Good |
| Learning curve | Medium | Low |
| TypeScript | Good | Excellent |
| Async | createAsyncThunk | ใส่ใน actions โดยตรง |
| Middleware | Complex | Simple |

### เมื่อไหรควรใช้ Zustand?

- App ขนาดกลาง ที่ไม่ต้องการ Redux
- ต้องการ state ที่ใช้ร่วมกันระหว่าง components
- ต้องการ code น้อย ๆ
- Team เล็ก หรือ Prototype

---

## ติดตั้ง Zustand

```bash
npm install zustand

# หรือ
yarn add zustand
```

---

## สร้าง Store พื้นฐาน

### Counter Store

```typescript
// src/stores/counterStore.ts
import { create } from 'zustand';

// กำหนด interface ของ store
interface CounterStore {
  // State
  count: number;
  step: number;

  // Actions
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  incrementByAmount: (amount: number) => void;
  setStep: (step: number) => void;
}

// สร้าง store
export const useCounterStore = create<CounterStore>((set) => ({
  // Initial state
  count: 0,
  step: 1,

  // Actions (functions ที่ update state)
  increment: () =>
    set((state) => ({ count: state.count + state.step })),

  decrement: () =>
    set((state) => ({ count: state.count - state.step })),

  reset: () => set({ count: 0 }),

  incrementByAmount: (amount) =>
    set((state) => ({ count: state.count + amount })),

  setStep: (step) => set({ step }),
}));
```

### ใช้งาน Store ใน Component

```typescript
// src/screens/CounterScreen.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useCounterStore } from '../stores/counterStore';

export const CounterScreen: React.FC = () => {
  // ดึง state และ actions จาก store
  const count = useCounterStore((state) => state.count);
  const step = useCounterStore((state) => state.step);
  const increment = useCounterStore((state) => state.increment);
  const decrement = useCounterStore((state) => state.decrement);
  const reset = useCounterStore((state) => state.reset);

  return (
    <View style={styles.container}>
      <Text style={styles.count}>{count}</Text>
      <Text style={styles.step}>Step: {step}</Text>
      <View style={styles.buttons}>
        <TouchableOpacity onPress={decrement} style={styles.button}>
          <Text style={styles.buttonText}>-</Text>
        </TouchableOpacity>
        <TouchableOpacity onPress={reset} style={[styles.button, styles.resetButton]}>
          <Text style={styles.buttonText}>Reset</Text>
        </TouchableOpacity>
        <TouchableOpacity onPress={increment} style={styles.button}>
          <Text style={styles.buttonText}>+</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  count: { fontSize: 72, fontWeight: 'bold', marginBottom: 24 },
  step: { fontSize: 16, color: '#666', marginBottom: 16 },
  buttons: { flexDirection: 'row', gap: 16 },
  button: {
    width: 64,
    height: 64,
    borderRadius: 32,
    backgroundColor: '#6200EE',
    alignItems: 'center',
    justifyContent: 'center',
  },
  resetButton: { backgroundColor: '#999' },
  buttonText: { color: '#fff', fontSize: 24, fontWeight: 'bold' },
});
```

---

## Actions และ State - Patterns ขั้นสูง

### Todo Store แบบสมบูรณ์

```typescript
// src/stores/todoStore.ts
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

export interface Todo {
  id: string;
  text: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  createdAt: number;
  tags: string[];
}

type Filter = 'all' | 'active' | 'completed';

interface TodoStore {
  // State
  todos: Todo[];
  filter: Filter;
  searchQuery: string;
  editingId: string | null;

  // Computed (derived state)
  filteredTodos: () => Todo[];
  stats: () => { total: number; active: number; completed: number };

  // Actions
  addTodo: (text: string, priority?: Todo['priority']) => void;
  toggleTodo: (id: string) => void;
  deleteTodo: (id: string) => void;
  updateTodo: (id: string, updates: Partial<Pick<Todo, 'text' | 'priority' | 'tags'>>) => void;
  setFilter: (filter: Filter) => void;
  setSearchQuery: (query: string) => void;
  clearCompleted: () => void;
  setEditingId: (id: string | null) => void;
  addTag: (todoId: string, tag: string) => void;
  removeTag: (todoId: string, tag: string) => void;
}

export const useTodoStore = create<TodoStore>()(
  immer((set, get) => ({
    // State
    todos: [],
    filter: 'all',
    searchQuery: '',
    editingId: null,

    // Computed state (functions ที่ return ค่า derived จาก state)
    filteredTodos: () => {
      const { todos, filter, searchQuery } = get();
      let result = todos;

      if (filter === 'active') {
        result = result.filter((t) => !t.completed);
      } else if (filter === 'completed') {
        result = result.filter((t) => t.completed);
      }

      if (searchQuery) {
        const q = searchQuery.toLowerCase();
        result = result.filter(
          (t) =>
            t.text.toLowerCase().includes(q) ||
            t.tags.some((tag) => tag.toLowerCase().includes(q))
        );
      }

      return result;
    },

    stats: () => {
      const todos = get().todos;
      return {
        total: todos.length,
        active: todos.filter((t) => !t.completed).length,
        completed: todos.filter((t) => t.completed).length,
      };
    },

    // Actions
    addTodo: (text, priority = 'medium') => {
      set((state) => {
        state.todos.unshift({
          id: Date.now().toString(),
          text,
          completed: false,
          priority,
          createdAt: Date.now(),
          tags: [],
        });
      });
    },

    toggleTodo: (id) => {
      set((state) => {
        const todo = state.todos.find((t) => t.id === id);
        if (todo) {
          todo.completed = !todo.completed;
        }
      });
    },

    deleteTodo: (id) => {
      set((state) => {
        state.todos = state.todos.filter((t) => t.id !== id);
      });
    },

    updateTodo: (id, updates) => {
      set((state) => {
        const todo = state.todos.find((t) => t.id === id);
        if (todo) {
          Object.assign(todo, updates);
        }
      });
    },

    setFilter: (filter) => set({ filter }),

    setSearchQuery: (searchQuery) => set({ searchQuery }),

    clearCompleted: () => {
      set((state) => {
        state.todos = state.todos.filter((t) => !t.completed);
      });
    },

    setEditingId: (editingId) => set({ editingId }),

    addTag: (todoId, tag) => {
      set((state) => {
        const todo = state.todos.find((t) => t.id === todoId);
        if (todo && !todo.tags.includes(tag)) {
          todo.tags.push(tag);
        }
      });
    },

    removeTag: (todoId, tag) => {
      set((state) => {
        const todo = state.todos.find((t) => t.id === todoId);
        if (todo) {
          todo.tags = todo.tags.filter((t) => t !== tag);
        }
      });
    },
  }))
);
```

---

## Middleware

Zustand มี middleware ที่ทรงพลัง

### Immer Middleware

ใช้ immer เพื่อ mutate state โดยตรง (เหมือน Redux Toolkit)

```bash
npm install immer
```

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

const useStore = create<State>()(
  immer((set) => ({
    todos: [],
    addTodo: (text: string) => {
      set((state) => {
        // mutate ได้โดยตรงด้วย immer
        state.todos.push({ id: Date.now().toString(), text, completed: false });
      });
    },
    toggleTodo: (id: string) => {
      set((state) => {
        const todo = state.todos.find((t) => t.id === id);
        if (todo) todo.completed = !todo.completed;
      });
    },
  }))
);
```

### Logger Middleware

```typescript
import { create, StateCreator, StoreApi } from 'zustand';

// Custom logger middleware
const logger =
  <T extends object>(
    config: StateCreator<T>
  ): StateCreator<T> =>
  (set, get, api) =>
    config(
      (args) => {
        console.log('Previous state:', get());
        set(args);
        console.log('Next state:', get());
      },
      get,
      api
    );

const useStore = create<State>()(
  logger((set) => ({
    count: 0,
    increment: () => set((state) => ({ count: state.count + 1 })),
  }))
);
```

### DevTools Middleware

```typescript
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

const useStore = create<State>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set((state) => ({ count: state.count + 1 }), false, 'increment'),
      //                                                                          ↑ action name
    }),
    { name: 'MyStore', enabled: __DEV__ }
  )
);
```

### Combine Middlewares

```typescript
import { create } from 'zustand';
import { devtools, persist, immer } from 'zustand/middleware';

const useStore = create<State>()(
  devtools(
    persist(
      immer((set) => ({
        // ...
      })),
      { name: 'my-storage' }
    )
  )
);
```

---

## Persist State

บันทึก state ลง AsyncStorage สำหรับ React Native

```bash
npm install @react-native-async-storage/async-storage
```

```typescript
// src/stores/settingsStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface SettingsStore {
  theme: 'light' | 'dark' | 'system';
  language: 'th' | 'en';
  notifications: boolean;
  fontSize: 'small' | 'medium' | 'large';
  setTheme: (theme: SettingsStore['theme']) => void;
  setLanguage: (language: SettingsStore['language']) => void;
  toggleNotifications: () => void;
  setFontSize: (size: SettingsStore['fontSize']) => void;
  resetSettings: () => void;
}

const defaultSettings = {
  theme: 'system' as const,
  language: 'th' as const,
  notifications: true,
  fontSize: 'medium' as const,
};

export const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      ...defaultSettings,

      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      toggleNotifications: () =>
        set((state) => ({ notifications: !state.notifications })),
      setFontSize: (fontSize) => set({ fontSize }),
      resetSettings: () => set(defaultSettings),
    }),
    {
      name: 'app-settings',
      storage: createJSONStorage(() => AsyncStorage),
      // เลือก state ที่จะ persist (ถ้าไม่ระบุจะ persist ทั้งหมด)
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
        notifications: state.notifications,
        fontSize: state.fontSize,
      }),
    }
  )
);
```

---

## Workshop: Shopping Cart ด้วย Zustand

### สร้าง Product Store

```typescript
// src/stores/productStore.ts
import { create } from 'zustand';

export interface Product {
  id: string;
  name: string;
  price: number;
  image: string;
  category: string;
  rating: number;
  stock: number;
  description: string;
}

interface ProductStore {
  products: Product[];
  isLoading: boolean;
  error: string | null;
  selectedCategory: string | null;
  fetchProducts: () => Promise<void>;
  setCategory: (category: string | null) => void;
  filteredProducts: () => Product[];
}

// Mock data
const MOCK_PRODUCTS: Product[] = [
  {
    id: '1',
    name: 'iPhone 15 Pro',
    price: 44900,
    image: 'https://picsum.photos/seed/iphone/300/300',
    category: 'phones',
    rating: 4.8,
    stock: 10,
    description: 'สมาร์ทโฟนล่าสุดจาก Apple',
  },
  {
    id: '2',
    name: 'Samsung Galaxy S24',
    price: 38900,
    image: 'https://picsum.photos/seed/samsung/300/300',
    category: 'phones',
    rating: 4.6,
    stock: 15,
    description: 'สมาร์ทโฟน Android รุ่นท็อป',
  },
  {
    id: '3',
    name: 'MacBook Pro M3',
    price: 89900,
    image: 'https://picsum.photos/seed/macbook/300/300',
    category: 'laptops',
    rating: 4.9,
    stock: 5,
    description: 'Laptop ประสิทธิภาพสูง',
  },
  {
    id: '4',
    name: 'AirPods Pro',
    price: 9900,
    image: 'https://picsum.photos/seed/airpods/300/300',
    category: 'accessories',
    rating: 4.7,
    stock: 30,
    description: 'หูฟังไร้สายพร้อม ANC',
  },
];

export const useProductStore = create<ProductStore>((set, get) => ({
  products: [],
  isLoading: false,
  error: null,
  selectedCategory: null,

  fetchProducts: async () => {
    set({ isLoading: true, error: null });
    try {
      await new Promise((r) => setTimeout(r, 800));
      set({ products: MOCK_PRODUCTS, isLoading: false });
    } catch {
      set({ error: 'ไม่สามารถโหลดสินค้าได้', isLoading: false });
    }
  },

  setCategory: (category) => set({ selectedCategory: category }),

  filteredProducts: () => {
    const { products, selectedCategory } = get();
    if (!selectedCategory) return products;
    return products.filter((p) => p.category === selectedCategory);
  },
}));
```

### สร้าง Cart Store

```typescript
// src/stores/cartStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Product } from './productStore';

export interface CartItem {
  product: Product;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  couponCode: string | null;
  discount: number;

  // Computed
  totalItems: () => number;
  subtotal: () => number;
  total: () => number;
  isInCart: (productId: string) => boolean;
  getQuantity: (productId: string) => number;

  // Actions
  addToCart: (product: Product, quantity?: number) => void;
  removeFromCart: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  clearCart: () => void;
  applyCoupon: (code: string) => boolean;
  removeCoupon: () => void;
}

const COUPONS: { [code: string]: number } = {
  SAVE10: 10,
  SAVE20: 20,
  SAVE50: 50,
};

export const useCartStore = create<CartStore>()(
  persist(
    immer((set, get) => ({
      items: [],
      couponCode: null,
      discount: 0,

      totalItems: () => {
        return get().items.reduce((sum, item) => sum + item.quantity, 0);
      },

      subtotal: () => {
        return get().items.reduce(
          (sum, item) => sum + item.product.price * item.quantity,
          0
        );
      },

      total: () => {
        const subtotal = get().subtotal();
        const discount = get().discount;
        return subtotal * (1 - discount / 100);
      },

      isInCart: (productId) => {
        return get().items.some((item) => item.product.id === productId);
      },

      getQuantity: (productId) => {
        const item = get().items.find((i) => i.product.id === productId);
        return item?.quantity || 0;
      },

      addToCart: (product, quantity = 1) => {
        set((state) => {
          const existingItem = state.items.find(
            (item) => item.product.id === product.id
          );
          if (existingItem) {
            existingItem.quantity += quantity;
          } else {
            state.items.push({ product, quantity });
          }
        });
      },

      removeFromCart: (productId) => {
        set((state) => {
          state.items = state.items.filter(
            (item) => item.product.id !== productId
          );
        });
      },

      updateQuantity: (productId, quantity) => {
        set((state) => {
          if (quantity <= 0) {
            state.items = state.items.filter(
              (item) => item.product.id !== productId
            );
          } else {
            const item = state.items.find((i) => i.product.id === productId);
            if (item) {
              item.quantity = quantity;
            }
          }
        });
      },

      clearCart: () => set({ items: [], couponCode: null, discount: 0 }),

      applyCoupon: (code) => {
        const discount = COUPONS[code.toUpperCase()];
        if (discount !== undefined) {
          set({ couponCode: code.toUpperCase(), discount });
          return true;
        }
        return false;
      },

      removeCoupon: () => set({ couponCode: null, discount: 0 }),
    })),
    {
      name: 'cart-storage',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

### Product List Screen

```typescript
// src/screens/ProductsScreen.tsx
import React, { useEffect } from 'react';
import {
  View,
  FlatList,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  ScrollView,
} from 'react-native';
import { useProductStore } from '../stores/productStore';
import { useCartStore } from '../stores/cartStore';

const CATEGORIES = [
  { id: null, label: 'ทั้งหมด' },
  { id: 'phones', label: 'โทรศัพท์' },
  { id: 'laptops', label: 'คอมพิวเตอร์' },
  { id: 'accessories', label: 'อุปกรณ์เสริม' },
];

export const ProductsScreen: React.FC = () => {
  const { fetchProducts, isLoading, selectedCategory, setCategory, filteredProducts } =
    useProductStore();

  const { addToCart, isInCart, getQuantity, totalItems } = useCartStore();

  useEffect(() => {
    fetchProducts();
  }, []);

  const products = filteredProducts();

  if (isLoading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    );
  }

  return (
    <View style={styles.container}>
      {/* Cart badge */}
      <View style={styles.cartBadge}>
        <Text style={styles.cartBadgeText}>🛒 {totalItems()} รายการ</Text>
      </View>

      {/* Category Filter */}
      <ScrollView
        horizontal
        showsHorizontalScrollIndicator={false}
        style={styles.categories}
        contentContainerStyle={{ paddingHorizontal: 16 }}
      >
        {CATEGORIES.map((cat) => (
          <TouchableOpacity
            key={String(cat.id)}
            onPress={() => setCategory(cat.id)}
            style={[
              styles.categoryChip,
              selectedCategory === cat.id && styles.activeCategoryChip,
            ]}
          >
            <Text
              style={[
                styles.categoryChipText,
                selectedCategory === cat.id && styles.activeCategoryChipText,
              ]}
            >
              {cat.label}
            </Text>
          </TouchableOpacity>
        ))}
      </ScrollView>

      {/* Products Grid */}
      <FlatList
        data={products}
        keyExtractor={(item) => item.id}
        numColumns={2}
        renderItem={({ item }) => (
          <View style={styles.productCard}>
            <Image
              source={{ uri: item.image }}
              style={styles.productImage}
              resizeMode="cover"
            />
            <View style={styles.productInfo}>
              <Text style={styles.productName} numberOfLines={1}>
                {item.name}
              </Text>
              <Text style={styles.productPrice}>
                ฿{item.price.toLocaleString()}
              </Text>
              <View style={styles.ratingRow}>
                <Text style={styles.rating}>⭐ {item.rating}</Text>
                <Text style={styles.stock}>เหลือ {item.stock}</Text>
              </View>
            </View>

            {isInCart(item.id) ? (
              <View style={styles.quantityControl}>
                <Text style={styles.inCartText}>
                  ในตะกร้า: {getQuantity(item.id)}
                </Text>
                <TouchableOpacity
                  onPress={() => addToCart(item)}
                  style={styles.addMoreButton}
                >
                  <Text style={styles.addMoreText}>+</Text>
                </TouchableOpacity>
              </View>
            ) : (
              <TouchableOpacity
                onPress={() => addToCart(item)}
                style={styles.addButton}
              >
                <Text style={styles.addButtonText}>เพิ่มลงตะกร้า</Text>
              </TouchableOpacity>
            )}
          </View>
        )}
        contentContainerStyle={styles.listContent}
        columnWrapperStyle={styles.row}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  cartBadge: {
    backgroundColor: '#6200EE',
    padding: 12,
    alignItems: 'flex-end',
    paddingRight: 16,
  },
  cartBadgeText: { color: '#fff', fontWeight: 'bold' },
  categories: { maxHeight: 50, backgroundColor: '#fff', marginBottom: 8 },
  categoryChip: {
    paddingHorizontal: 16,
    paddingVertical: 10,
    marginRight: 8,
    borderRadius: 20,
    backgroundColor: '#F5F5F5',
  },
  activeCategoryChip: { backgroundColor: '#6200EE' },
  categoryChipText: { fontSize: 12, color: '#666' },
  activeCategoryChipText: { color: '#fff', fontWeight: 'bold' },
  listContent: { padding: 8 },
  row: { justifyContent: 'space-between' },
  productCard: {
    backgroundColor: '#fff',
    borderRadius: 12,
    margin: 4,
    flex: 0.48,
    overflow: 'hidden',
    elevation: 2,
  },
  productImage: { width: '100%', height: 140 },
  productInfo: { padding: 8 },
  productName: { fontSize: 13, fontWeight: 'bold', marginBottom: 4 },
  productPrice: { fontSize: 15, color: '#6200EE', fontWeight: 'bold', marginBottom: 4 },
  ratingRow: { flexDirection: 'row', justifyContent: 'space-between' },
  rating: { fontSize: 11, color: '#FF9800' },
  stock: { fontSize: 11, color: '#999' },
  addButton: {
    backgroundColor: '#6200EE',
    margin: 8,
    padding: 8,
    borderRadius: 6,
    alignItems: 'center',
  },
  addButtonText: { color: '#fff', fontSize: 12, fontWeight: 'bold' },
  quantityControl: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    margin: 8,
    padding: 4,
    backgroundColor: '#EDE7F6',
    borderRadius: 6,
  },
  inCartText: { fontSize: 11, color: '#6200EE', marginLeft: 4 },
  addMoreButton: {
    backgroundColor: '#6200EE',
    width: 24,
    height: 24,
    borderRadius: 12,
    alignItems: 'center',
    justifyContent: 'center',
  },
  addMoreText: { color: '#fff', fontWeight: 'bold' },
});
```

### Cart Screen

```typescript
// src/screens/CartScreen.tsx
import React, { useState } from 'react';
import {
  View,
  FlatList,
  Text,
  TouchableOpacity,
  Image,
  TextInput,
  StyleSheet,
  Alert,
} from 'react-native';
import { useCartStore, CartItem } from '../stores/cartStore';

export const CartScreen: React.FC = () => {
  const {
    items,
    totalItems,
    subtotal,
    total,
    couponCode,
    discount,
    removeFromCart,
    updateQuantity,
    clearCart,
    applyCoupon,
    removeCoupon,
  } = useCartStore();

  const [couponInput, setCouponInput] = useState('');

  const handleApplyCoupon = () => {
    const success = applyCoupon(couponInput);
    if (success) {
      Alert.alert('สำเร็จ', `ใช้คูปอง ${couponInput.toUpperCase()} ส่วนลด ${discount}%`);
      setCouponInput('');
    } else {
      Alert.alert('ผิดพลาด', 'คูปองไม่ถูกต้อง');
    }
  };

  const handleCheckout = () => {
    Alert.alert(
      'ยืนยันการสั่งซื้อ',
      `ยอดรวม: ฿${total().toLocaleString()}`,
      [
        { text: 'ยกเลิก', style: 'cancel' },
        {
          text: 'ยืนยัน',
          onPress: () => {
            clearCart();
            Alert.alert('สำเร็จ', 'สั่งซื้อเรียบร้อย!');
          },
        },
      ]
    );
  };

  const renderCartItem = ({ item }: { item: CartItem }) => (
    <View style={styles.cartItem}>
      <Image
        source={{ uri: item.product.image }}
        style={styles.itemImage}
        resizeMode="cover"
      />
      <View style={styles.itemInfo}>
        <Text style={styles.itemName}>{item.product.name}</Text>
        <Text style={styles.itemPrice}>
          ฿{item.product.price.toLocaleString()}
        </Text>
        <View style={styles.quantityRow}>
          <TouchableOpacity
            onPress={() => updateQuantity(item.product.id, item.quantity - 1)}
            style={styles.qtyButton}
          >
            <Text style={styles.qtyButtonText}>-</Text>
          </TouchableOpacity>
          <Text style={styles.quantity}>{item.quantity}</Text>
          <TouchableOpacity
            onPress={() => updateQuantity(item.product.id, item.quantity + 1)}
            style={styles.qtyButton}
          >
            <Text style={styles.qtyButtonText}>+</Text>
          </TouchableOpacity>
          <Text style={styles.itemTotal}>
            = ฿{(item.product.price * item.quantity).toLocaleString()}
          </Text>
        </View>
      </View>
      <TouchableOpacity
        onPress={() => removeFromCart(item.product.id)}
        style={styles.removeButton}
      >
        <Text style={styles.removeText}>✕</Text>
      </TouchableOpacity>
    </View>
  );

  if (items.length === 0) {
    return (
      <View style={styles.emptyContainer}>
        <Text style={styles.emptyIcon}>🛒</Text>
        <Text style={styles.emptyText}>ตะกร้าว่างเปล่า</Text>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <FlatList
        data={items}
        keyExtractor={(item) => item.product.id}
        renderItem={renderCartItem}
        ListFooterComponent={() => (
          <View style={styles.footer}>
            {/* Coupon */}
            <View style={styles.couponSection}>
              {couponCode ? (
                <View style={styles.appliedCoupon}>
                  <Text style={styles.couponText}>
                    คูปอง: {couponCode} (-{discount}%)
                  </Text>
                  <TouchableOpacity onPress={removeCoupon}>
                    <Text style={styles.removeCoupon}>ลบ</Text>
                  </TouchableOpacity>
                </View>
              ) : (
                <View style={styles.couponInput}>
                  <TextInput
                    value={couponInput}
                    onChangeText={setCouponInput}
                    placeholder="ใส่รหัสคูปอง"
                    style={styles.couponTextInput}
                    autoCapitalize="characters"
                  />
                  <TouchableOpacity
                    onPress={handleApplyCoupon}
                    style={styles.applyCouponButton}
                  >
                    <Text style={styles.applyCouponText}>ใช้</Text>
                  </TouchableOpacity>
                </View>
              )}
            </View>

            {/* Summary */}
            <View style={styles.summary}>
              <View style={styles.summaryRow}>
                <Text style={styles.summaryLabel}>รายการทั้งหมด</Text>
                <Text style={styles.summaryValue}>{totalItems()} ชิ้น</Text>
              </View>
              <View style={styles.summaryRow}>
                <Text style={styles.summaryLabel}>ราคารวม</Text>
                <Text style={styles.summaryValue}>
                  ฿{subtotal().toLocaleString()}
                </Text>
              </View>
              {discount > 0 && (
                <View style={styles.summaryRow}>
                  <Text style={styles.discountLabel}>ส่วนลด {discount}%</Text>
                  <Text style={styles.discountValue}>
                    -฿{(subtotal() * discount / 100).toLocaleString()}
                  </Text>
                </View>
              )}
              <View style={[styles.summaryRow, styles.totalRow]}>
                <Text style={styles.totalLabel}>ยอดรวมสุทธิ</Text>
                <Text style={styles.totalValue}>
                  ฿{total().toLocaleString()}
                </Text>
              </View>
            </View>

            {/* Checkout */}
            <TouchableOpacity
              onPress={handleCheckout}
              style={styles.checkoutButton}
            >
              <Text style={styles.checkoutText}>
                สั่งซื้อ ฿{total().toLocaleString()}
              </Text>
            </TouchableOpacity>
          </View>
        )}
        contentContainerStyle={styles.listContent}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  emptyContainer: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  emptyIcon: { fontSize: 72, marginBottom: 16 },
  emptyText: { fontSize: 20, color: '#999' },
  listContent: { padding: 16 },
  cartItem: {
    flexDirection: 'row',
    backgroundColor: '#fff',
    borderRadius: 12,
    marginBottom: 12,
    padding: 12,
    elevation: 1,
  },
  itemImage: { width: 70, height: 70, borderRadius: 8 },
  itemInfo: { flex: 1, marginLeft: 12 },
  itemName: { fontSize: 14, fontWeight: 'bold', marginBottom: 4 },
  itemPrice: { fontSize: 13, color: '#6200EE', marginBottom: 8 },
  quantityRow: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  qtyButton: {
    width: 28,
    height: 28,
    borderRadius: 14,
    backgroundColor: '#E0E0E0',
    alignItems: 'center',
    justifyContent: 'center',
  },
  qtyButtonText: { fontWeight: 'bold' },
  quantity: { fontSize: 16, fontWeight: 'bold', minWidth: 20, textAlign: 'center' },
  itemTotal: { fontSize: 13, color: '#333', marginLeft: 8 },
  removeButton: { padding: 4 },
  removeText: { color: '#FF5252', fontSize: 16 },
  footer: { marginTop: 8 },
  couponSection: { backgroundColor: '#fff', borderRadius: 12, padding: 16, marginBottom: 12 },
  couponInput: { flexDirection: 'row', gap: 8 },
  couponTextInput: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#E0E0E0',
    borderRadius: 8,
    padding: 10,
    fontSize: 14,
  },
  applyCouponButton: {
    backgroundColor: '#6200EE',
    paddingHorizontal: 16,
    borderRadius: 8,
    justifyContent: 'center',
  },
  applyCouponText: { color: '#fff', fontWeight: 'bold' },
  appliedCoupon: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: '#E8F5E9',
    padding: 12,
    borderRadius: 8,
  },
  couponText: { color: '#4CAF50', fontWeight: 'bold' },
  removeCoupon: { color: '#F44336' },
  summary: { backgroundColor: '#fff', borderRadius: 12, padding: 16, marginBottom: 12 },
  summaryRow: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 8 },
  summaryLabel: { fontSize: 14, color: '#666' },
  summaryValue: { fontSize: 14 },
  discountLabel: { fontSize: 14, color: '#4CAF50' },
  discountValue: { fontSize: 14, color: '#4CAF50' },
  totalRow: { borderTopWidth: 1, borderTopColor: '#E0E0E0', paddingTop: 8, marginTop: 4 },
  totalLabel: { fontSize: 16, fontWeight: 'bold' },
  totalValue: { fontSize: 18, fontWeight: 'bold', color: '#6200EE' },
  checkoutButton: {
    backgroundColor: '#6200EE',
    padding: 16,
    borderRadius: 12,
    alignItems: 'center',
  },
  checkoutText: { color: '#fff', fontSize: 18, fontWeight: 'bold' },
});
```

---

## Tips and Best Practices

### 1. แยก Selectors ออกมา

```typescript
// ❌ ไม่ดี - select ทั้ง store
const { items, totalItems, subtotal } = useCartStore();

// ✅ ดีกว่า - select เฉพาะที่ต้องการ (ลด re-renders)
const items = useCartStore((state) => state.items);
const totalItems = useCartStore((state) => state.totalItems());
```

### 2. ใช้ shallow comparison

```typescript
import { shallow } from 'zustand/shallow';

// Select หลาย values แบบ efficient
const { items, discount } = useCartStore(
  (state) => ({ items: state.items, discount: state.discount }),
  shallow
);
```

### 3. Subscribe outside React

```typescript
// สมัคร subscription นอก component
const unsubscribe = useCartStore.subscribe(
  (state) => state.items,
  (items) => {
    console.log('Cart items changed:', items.length);
  }
);

// ยกเลิก subscription เมื่อต้องการ
unsubscribe();
```

### 4. ใช้ getState() และ setState() โดยตรง

```typescript
// อ่าน state นอก React
const items = useCartStore.getState().items;

// อัปเดต state นอก React
useCartStore.setState({ items: [] });
```

---

## สรุป

Zustand เป็น state management ที่:
1. **เรียบง่าย** - โค้ดน้อย setup ง่าย
2. **เบา** - ขนาดเล็กมาก
3. **Immer built-in** - mutate state ได้โดยตรง
4. **Persist** - บันทึก state ลง AsyncStorage ง่ายมาก
5. **TypeScript** - รองรับดีมาก

ใน Part 034 จะเรียนเรื่อง **React Query** สำหรับ server state management
