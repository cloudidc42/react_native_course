# Part 008: Props และ State

## สารบัญ
1. [Props คืออะไร](#props)
2. [การส่ง Props](#การส่ง-props)
3. [PropTypes Validation](#proptypes)
4. [TypeScript Props Types](#typescript-props)
5. [State ด้วย useState](#usestate)
6. [State ที่ซับซ้อน](#complex-state)
7. [Lifting State Up](#lifting-state-up)
8. [Controlled vs Uncontrolled](#controlled-uncontrolled)
9. [useReducer](#usereducer)
10. [Workshop: Counter App และ Form](#workshop)

---

## Props

Props (Properties) คือข้อมูลที่ส่งจาก **Parent** ไปยัง **Child** component

### หลักการ

```
┌─────────────────────────────────┐
│        Parent Component         │
│                                 │
│  const name = "สมชาย"          │
│  const age = 25                 │
│                                 │
│  <Child name={name} age={age} />│
│     ↓                           │
└─────────────────────────────────┘
             ↓ Props
┌─────────────────────────────────┐
│         Child Component         │
│                                 │
│  ({ name, age }) => {           │
│    return <Text>{name}</Text>   │
│  }                              │
└─────────────────────────────────┘
```

### Props เป็น Read-only (Immutable)

```tsx
// ❌ ไม่ได้ - ห้ามแก้ props
const BadChild = ({ count }) => {
  count = count + 1;  // Error! Props are read-only
  return <Text>{count}</Text>;
};

// ✅ ถูกต้อง - ใช้ local state ถ้าต้องการแก้ค่า
const GoodChild = ({ initialCount }) => {
  const [count, setCount] = useState(initialCount);
  return (
    <TouchableOpacity onPress={() => setCount(count + 1)}>
      <Text>{count}</Text>
    </TouchableOpacity>
  );
};
```

---

## การส่ง Props

### Props ชนิดต่างๆ

```tsx
// Parent ส่ง props หลายชนิด
<UserCard
  // String
  name="สมชาย ใจดี"
  title="React Native Developer"
  
  // Number
  age={25}
  rating={4.5}
  
  // Boolean
  isActive={true}
  isAdmin     // เหมือนกัน isAdmin={true}
  
  // Array
  skills={['JavaScript', 'TypeScript', 'React']}
  
  // Object
  address={{
    street: '123 ถนนสุขุมวิท',
    city: 'กรุงเทพฯ',
  }}
  
  // Function
  onPress={() => console.log('กด!')}
  onDelete={handleDelete}
  
  // JSX/Component
  icon={<StarIcon size={20} color="#FFD700" />}
  footer={
    <View>
      <Button label="Follow" />
    </View>
  }
  
  // Spread ทั้ง object
  {...userProfile}
/>
```

### Children Prop

```tsx
// children คือ special prop สำหรับ nested elements
const Card = ({ children, title, style }) => (
  <View style={[styles.card, style]}>
    {title && <Text style={styles.title}>{title}</Text>}
    {children}
  </View>
);

// ใช้งาน
<Card title="ชื่อการ์ด">
  <Text>เนื้อหา 1</Text>
  <Text>เนื้อหา 2</Text>
  <Button label="ปุ่ม" />
</Card>

// ตรวจสอบ children
const Layout = ({ children }) => {
  const childCount = React.Children.count(children);
  console.log('มี children:', childCount);
  
  return (
    <View>
      {React.Children.map(children, (child, index) => (
        <View key={index} style={{ marginBottom: 8 }}>
          {child}
        </View>
      ))}
    </View>
  );
};
```

### Default Props

```tsx
// วิธีที่ 1: Default value ใน destructuring (แนะนำ)
const Button = ({ label, variant = 'primary', size = 'md', disabled = false }) => {
  // ...
};

// วิธีที่ 2: defaultProps (legacy)
Button.defaultProps = {
  variant: 'primary',
  size: 'md',
  disabled: false,
};

// วิธีที่ 3: Nullish coalescing
const Button = ({ label, variant, size }) => {
  const actualVariant = variant ?? 'primary';
  const actualSize = size ?? 'md';
  // ...
};
```

---

## PropTypes Validation

PropTypes ช่วย validate props ใน JavaScript (ไม่ต้องใช้ถ้าใช้ TypeScript)

```bash
npm install prop-types
```

```jsx
import PropTypes from 'prop-types';

const UserCard = ({ name, age, role, onPress, skills, address }) => {
  // ...
};

UserCard.propTypes = {
  // Required
  name: PropTypes.string.isRequired,
  onPress: PropTypes.func.isRequired,
  
  // Optional
  age: PropTypes.number,
  role: PropTypes.string,
  
  // One of specific values
  variant: PropTypes.oneOf(['primary', 'secondary', 'outline']),
  
  // Array of specific type
  skills: PropTypes.arrayOf(PropTypes.string),
  
  // Object shape
  address: PropTypes.shape({
    street: PropTypes.string,
    city: PropTypes.string.isRequired,
    zip: PropTypes.string,
  }),
  
  // One of multiple types
  id: PropTypes.oneOfType([PropTypes.string, PropTypes.number]),
  
  // Array of objects
  items: PropTypes.arrayOf(
    PropTypes.shape({
      id: PropTypes.number.isRequired,
      name: PropTypes.string.isRequired,
    })
  ),
  
  // Children
  children: PropTypes.node,
  children: PropTypes.element,
  children: PropTypes.elementType,
  
  // Any
  data: PropTypes.any,
  
  // Custom validator
  evenNumber: (props, propName, componentName) => {
    if (props[propName] % 2 !== 0) {
      return new Error(`${propName} ใน ${componentName} ต้องเป็นเลขคู่`);
    }
  },
};
```

---

## TypeScript Props Types

TypeScript เป็น solution ที่แนะนำ (แทน PropTypes)

```tsx
// Pattern 1: type
type ButtonProps = {
  label: string;
  onPress: () => void;
  variant?: 'primary' | 'secondary' | 'outline' | 'destructive';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  loading?: boolean;
  icon?: React.ReactNode;
};

// Pattern 2: interface (เหมาะสำหรับ OOP/extending)
interface CardProps {
  title: string;
  description?: string;
  imageUrl?: string;
  onPress?: () => void;
  children?: React.ReactNode;
}

// Extending interface
interface PressableCardProps extends CardProps {
  onLongPress?: () => void;
  hapticFeedback?: boolean;
}

// Generic Props
interface ListProps<T> {
  data: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string;
  emptyComponent?: React.ReactNode;
}

// ใช้งาน Generic
const List = <T,>({ data, renderItem, keyExtractor, emptyComponent }: ListProps<T>) => {
  if (data.length === 0) {
    return <>{emptyComponent ?? <Text>ไม่มีข้อมูล</Text>}</>;
  }
  
  return (
    <View>
      {data.map((item, index) => (
        <View key={keyExtractor(item)}>
          {renderItem(item, index)}
        </View>
      ))}
    </View>
  );
};
```

### Utility Types ที่ใช้บ่อย

```tsx
// Partial - ทุก property เป็น optional
type PartialUser = Partial<User>;

// Required - ทุก property เป็น required
type RequiredUser = Required<User>;

// Pick - เลือก properties
type UserNameAndAge = Pick<User, 'name' | 'age'>;

// Omit - ลบ properties
type UserWithoutPassword = Omit<User, 'password'>;

// ใช้ใน Props
type EditUserProps = Partial<Omit<User, 'id' | 'createdAt'>> & {
  userId: string;
  onSave: (user: User) => void;
};
```

---

## useState

useState คือ Hook ที่ใช้จัดการ state ใน Functional Component

### การใช้งานพื้นฐาน

```tsx
import React, { useState } from 'react';

const Counter = () => {
  // [currentValue, setterFunction] = useState(initialValue)
  const [count, setCount] = useState<number>(0);

  // Direct update
  const increment = () => setCount(count + 1);
  
  // Functional update (แนะนำเมื่อ new state ขึ้นกับ old state)
  const decrement = () => setCount(prev => prev - 1);
  
  // Reset
  const reset = () => setCount(0);

  return (
    <View>
      <Text style={styles.count}>{count}</Text>
      <View style={styles.buttons}>
        <Button label="−" onPress={decrement} />
        <Button label="Reset" onPress={reset} variant="outline" />
        <Button label="+" onPress={increment} />
      </View>
    </View>
  );
};
```

### State ชนิดต่างๆ

```tsx
// Boolean State
const [isVisible, setIsVisible] = useState(false);
const toggle = () => setIsVisible(prev => !prev);

// String State
const [searchText, setSearchText] = useState('');

// Array State
const [items, setItems] = useState<string[]>([]);

// Object State
const [user, setUser] = useState<User | null>(null);

// Multiple States
const [loading, setLoading] = useState(false);
const [error, setError] = useState<string | null>(null);
const [data, setData] = useState<Product[]>([]);
```

---

## Complex State

### State ที่เป็น Object

```tsx
type FormData = {
  name: string;
  email: string;
  phone: string;
  isTermsAccepted: boolean;
};

const Form = () => {
  const [formData, setFormData] = useState<FormData>({
    name: '',
    email: '',
    phone: '',
    isTermsAccepted: false,
  });

  // ✅ วิธีอัปเดต Object State - spread แล้ว override
  const updateField = (field: keyof FormData) => (value: any) => {
    setFormData(prev => ({
      ...prev,        // copy ทั้งหมดก่อน
      [field]: value, // แล้ว override field ที่ต้องการ
    }));
  };

  return (
    <View>
      <TextInput
        value={formData.name}
        onChangeText={updateField('name')}
        placeholder="ชื่อ"
      />
      <TextInput
        value={formData.email}
        onChangeText={updateField('email')}
        placeholder="Email"
      />
      <Switch
        value={formData.isTermsAccepted}
        onValueChange={updateField('isTermsAccepted')}
      />
    </View>
  );
};
```

### State ที่เป็น Array

```tsx
type TodoItem = {
  id: string;
  text: string;
  completed: boolean;
};

const TodoApp = () => {
  const [todos, setTodos] = useState<TodoItem[]>([]);
  const [inputText, setInputText] = useState('');

  // Add item
  const addTodo = () => {
    if (!inputText.trim()) return;
    
    const newTodo: TodoItem = {
      id: Date.now().toString(),
      text: inputText.trim(),
      completed: false,
    };
    
    setTodos(prev => [...prev, newTodo]);  // spread + new item
    setInputText('');
  };

  // Toggle completed
  const toggleTodo = (id: string) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      )
    );
  };

  // Remove item
  const removeTodo = (id: string) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  // Update item
  const updateTodo = (id: string, text: string) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, text } : todo
      )
    );
  };

  return (
    <View>
      <View style={{ flexDirection: 'row', gap: 8 }}>
        <TextInput
          value={inputText}
          onChangeText={setInputText}
          placeholder="เพิ่ม Todo..."
          style={{ flex: 1 }}
          onSubmitEditing={addTodo}
          returnKeyType="done"
        />
        <Button label="เพิ่ม" onPress={addTodo} />
      </View>

      {todos.map(todo => (
        <View key={todo.id} style={{ flexDirection: 'row', alignItems: 'center', gap: 8 }}>
          <TouchableOpacity onPress={() => toggleTodo(todo.id)}>
            <Text>{todo.completed ? '✅' : '⬜'}</Text>
          </TouchableOpacity>
          <Text
            style={[
              { flex: 1 },
              todo.completed && { textDecorationLine: 'line-through', color: '#999' },
            ]}
          >
            {todo.text}
          </Text>
          <TouchableOpacity onPress={() => removeTodo(todo.id)}>
            <Text style={{ color: '#FF3B30' }}>ลบ</Text>
          </TouchableOpacity>
        </View>
      ))}

      <Text style={{ color: '#999', marginTop: 8 }}>
        ทำแล้ว: {todos.filter(t => t.completed).length}/{todos.length}
      </Text>
    </View>
  );
};
```

---

## Lifting State Up

เมื่อ siblings components ต้องการ share state → ยก state ขึ้นไปยัง parent

### ปัญหา: State กระจาย

```tsx
// ❌ ปัญหา - TabA และ TabB ไม่รู้ state ของกัน
const Parent = () => (
  <View>
    <TabA />  {/* มี selectedItem state ของตัวเอง */}
    <TabB />  {/* ต้องการรู้ selectedItem ของ TabA */}
  </View>
);
```

### แก้ด้วย Lifting State Up

```tsx
// ✅ แก้ - ยก state ขึ้นไป Parent
const Parent = () => {
  // State อยู่ที่ Parent
  const [selectedItem, setSelectedItem] = useState<Item | null>(null);

  return (
    <View>
      <TabA
        onSelectItem={setSelectedItem}  // ส่ง setter ลงไป
      />
      <TabB
        selectedItem={selectedItem}     // ส่ง state ลงไป
      />
    </View>
  );
};

const TabA = ({ onSelectItem }) => {
  const items = ['สินค้า 1', 'สินค้า 2', 'สินค้า 3'];
  
  return (
    <View>
      {items.map(item => (
        <TouchableOpacity
          key={item}
          onPress={() => onSelectItem(item)}
        >
          <Text>{item}</Text>
        </TouchableOpacity>
      ))}
    </View>
  );
};

const TabB = ({ selectedItem }) => (
  <View>
    {selectedItem ? (
      <Text>คุณเลือก: {selectedItem}</Text>
    ) : (
      <Text>ยังไม่ได้เลือก</Text>
    )}
  </View>
);
```

### ตัวอย่างจริง: Shopping Cart

```tsx
type Product = { id: string; name: string; price: number };
type CartItem = Product & { quantity: number };

// Parent Component (lifting state)
const ShopScreen = () => {
  const [cartItems, setCartItems] = useState<CartItem[]>([]);

  const addToCart = (product: Product) => {
    setCartItems(prev => {
      const existing = prev.find(item => item.id === product.id);
      if (existing) {
        return prev.map(item =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }
      return [...prev, { ...product, quantity: 1 }];
    });
  };

  const removeFromCart = (productId: string) => {
    setCartItems(prev => prev.filter(item => item.id !== productId));
  };

  const totalPrice = cartItems.reduce(
    (sum, item) => sum + item.price * item.quantity, 0
  );

  return (
    <View style={{ flex: 1 }}>
      {/* Product List */}
      <ProductList onAddToCart={addToCart} />
      
      {/* Cart Summary */}
      <CartSummary
        items={cartItems}
        totalPrice={totalPrice}
        onRemove={removeFromCart}
      />
    </View>
  );
};

// Child 1: Products
const ProductList = ({ onAddToCart }) => {
  const products: Product[] = [
    { id: '1', name: 'เสื้อ', price: 299 },
    { id: '2', name: 'กางเกง', price: 499 },
    { id: '3', name: 'รองเท้า', price: 1299 },
  ];

  return (
    <ScrollView>
      {products.map(product => (
        <View key={product.id} style={{ flexDirection: 'row', alignItems: 'center', padding: 12 }}>
          <View style={{ flex: 1 }}>
            <Text style={{ fontWeight: '600' }}>{product.name}</Text>
            <Text style={{ color: '#007AFF' }}>฿{product.price}</Text>
          </View>
          <TouchableOpacity
            style={{ backgroundColor: '#007AFF', padding: 8, borderRadius: 6 }}
            onPress={() => onAddToCart(product)}
          >
            <Text style={{ color: '#fff' }}>+ เพิ่ม</Text>
          </TouchableOpacity>
        </View>
      ))}
    </ScrollView>
  );
};

// Child 2: Cart
const CartSummary = ({ items, totalPrice, onRemove }) => (
  <View style={{ padding: 16, backgroundColor: '#f9f9f9', borderTopWidth: 1, borderTopColor: '#e0e0e0' }}>
    <Text style={{ fontSize: 16, fontWeight: 'bold', marginBottom: 8 }}>
      ตะกร้า ({items.length} ชนิด)
    </Text>
    {items.map(item => (
      <View key={item.id} style={{ flexDirection: 'row', alignItems: 'center', marginBottom: 4 }}>
        <Text style={{ flex: 1 }}>{item.name} x{item.quantity}</Text>
        <Text style={{ color: '#007AFF', marginRight: 8 }}>฿{item.price * item.quantity}</Text>
        <TouchableOpacity onPress={() => onRemove(item.id)}>
          <Text style={{ color: '#FF3B30' }}>ลบ</Text>
        </TouchableOpacity>
      </View>
    ))}
    <Text style={{ fontSize: 16, fontWeight: 'bold', marginTop: 8, color: '#007AFF' }}>
      รวม: ฿{totalPrice}
    </Text>
  </View>
);
```

---

## Controlled vs Uncontrolled Components

### Controlled Component

React ควบคุม value ของ input ทั้งหมด

```tsx
// Controlled: React เป็น "source of truth"
const ControlledInput = () => {
  const [value, setValue] = useState('');

  return (
    <TextInput
      value={value}           // ← controlled: React กำหนด value
      onChangeText={setValue} // ← React update state เมื่อ user พิมพ์
      placeholder="Controlled input"
    />
  );
};

// ✅ ข้อดี:
// - อ่านค่าได้ตลอดเวลา
// - Validate ได้ทันที
// - Format input ได้ (เช่น uppercase เสมอ)
// - Disable submit ได้ตาม conditions

const ControlledForm = () => {
  const [email, setEmail] = useState('');
  const [isValid, setIsValid] = useState(false);

  const handleEmailChange = (text: string) => {
    const lowerText = text.toLowerCase().trim();  // Format: lowercase
    setEmail(lowerText);
    setIsValid(/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(lowerText));
  };

  return (
    <View>
      <TextInput
        value={email}
        onChangeText={handleEmailChange}
        placeholder="Email"
        keyboardType="email-address"
        style={{ borderColor: isValid ? 'green' : 'red' }}
      />
      {!isValid && email.length > 0 && (
        <Text style={{ color: 'red' }}>Email ไม่ถูกต้อง</Text>
      )}
      <Button
        label="ส่ง"
        onPress={handleSubmit}
        disabled={!isValid}
      />
    </View>
  );
};
```

### Uncontrolled Component

ใช้ ref เข้าถึง value โดยตรง

```tsx
import React, { useRef } from 'react';
import { TextInput } from 'react-native';

const UncontrolledInput = () => {
  const inputRef = useRef<TextInput>(null);

  const handleSubmit = () => {
    // อ่านค่า (ต้องใช้ library เพิ่มหรือ track state เอง)
    // React Native TextInput ไม่รองรับ .value โดยตรง
    // ต้องใช้ callback ref แทน
  };

  return (
    <View>
      {/* defaultValue แทน value = uncontrolled */}
      <TextInput
        defaultValue="ค่าเริ่มต้น"
        ref={inputRef}
        placeholder="Uncontrolled"
      />
      <Button label="ส่ง" onPress={handleSubmit} />
    </View>
  );
};

// ⚠️ ใน React Native uncontrolled ไม่ค่อยนิยม
// แนะนำใช้ controlled เสมอ
```

---

## useReducer

useReducer เหมาะสำหรับ state ที่ซับซ้อน มีหลาย actions

```tsx
import React, { useReducer } from 'react';

// 1. กำหนด State Type
type CartState = {
  items: CartItem[];
  loading: boolean;
  error: string | null;
  discount: number;
};

// 2. กำหนด Action Types
type CartAction =
  | { type: 'ADD_ITEM'; payload: Product }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'UPDATE_QUANTITY'; payload: { id: string; quantity: number } }
  | { type: 'CLEAR_CART' }
  | { type: 'APPLY_DISCOUNT'; payload: number }
  | { type: 'SET_LOADING'; payload: boolean }
  | { type: 'SET_ERROR'; payload: string | null };

// 3. สร้าง Initial State
const initialState: CartState = {
  items: [],
  loading: false,
  error: null,
  discount: 0,
};

// 4. สร้าง Reducer
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existing = state.items.find(item => item.id === action.payload.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map(item =>
            item.id === action.payload.id
              ? { ...item, quantity: item.quantity + 1 }
              : item
          ),
        };
      }
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
      };
    }

    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.payload),
      };

    case 'UPDATE_QUANTITY':
      return {
        ...state,
        items: state.items.map(item =>
          item.id === action.payload.id
            ? { ...item, quantity: Math.max(0, action.payload.quantity) }
            : item
        ).filter(item => item.quantity > 0),
      };

    case 'CLEAR_CART':
      return { ...state, items: [], discount: 0 };

    case 'APPLY_DISCOUNT':
      return { ...state, discount: action.payload };

    case 'SET_LOADING':
      return { ...state, loading: action.payload };

    case 'SET_ERROR':
      return { ...state, error: action.payload };

    default:
      return state;
  }
}

// 5. ใช้งาน
const CartScreen = () => {
  const [state, dispatch] = useReducer(cartReducer, initialState);

  // Derived state
  const subtotal = state.items.reduce(
    (sum, item) => sum + item.price * item.quantity, 0
  );
  const total = subtotal * (1 - state.discount / 100);

  return (
    <View>
      {state.items.map(item => (
        <View key={item.id}>
          <Text>{item.name}</Text>
          <View style={{ flexDirection: 'row', gap: 8 }}>
            <Button
              label="−"
              onPress={() => dispatch({
                type: 'UPDATE_QUANTITY',
                payload: { id: item.id, quantity: item.quantity - 1 }
              })}
            />
            <Text>{item.quantity}</Text>
            <Button
              label="+"
              onPress={() => dispatch({
                type: 'UPDATE_QUANTITY',
                payload: { id: item.id, quantity: item.quantity + 1 }
              })}
            />
          </View>
        </View>
      ))}
      
      <Text>รวม: ฿{total.toFixed(0)}</Text>
      
      <Button
        label="ล้างตะกร้า"
        onPress={() => dispatch({ type: 'CLEAR_CART' })}
        variant="outline"
      />
    </View>
  );
};
```

---

## Workshop: Counter App และ Form

### Workshop 8.1: Counter App สมบูรณ์

```tsx
import React, { useState, useCallback } from 'react';
import {
  View, Text, TouchableOpacity, StyleSheet, SafeAreaView, StatusBar,
} from 'react-native';

type CounterStep = 1 | 5 | 10;

const CounterApp = () => {
  const [count, setCount] = useState(0);
  const [step, setStep] = useState<CounterStep>(1);
  const [history, setHistory] = useState<number[]>([0]);

  const MAX = 100;
  const MIN = -100;

  const increment = useCallback(() => {
    setCount(prev => {
      const next = Math.min(prev + step, MAX);
      setHistory(h => [...h.slice(-9), next]);
      return next;
    });
  }, [step]);

  const decrement = useCallback(() => {
    setCount(prev => {
      const next = Math.max(prev - step, MIN);
      setHistory(h => [...h.slice(-9), next]);
      return next;
    });
  }, [step]);

  const reset = () => {
    setCount(0);
    setHistory([0]);
  };

  const getCountColor = () => {
    if (count > 0) return '#34C759';
    if (count < 0) return '#FF3B30';
    return '#333';
  };

  const steps: CounterStep[] = [1, 5, 10];

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="dark-content" />
      
      {/* Title */}
      <Text style={styles.title}>Counter App</Text>

      {/* Count Display */}
      <View style={styles.countContainer}>
        <Text style={[styles.count, { color: getCountColor() }]}>
          {count > 0 ? '+' : ''}{count}
        </Text>
        <View style={[
          styles.progressBar,
          { backgroundColor: '#f0f0f0' }
        ]}>
          <View
            style={[
              styles.progressFill,
              {
                width: `${Math.abs(count)}%`,
                backgroundColor: getCountColor(),
                alignSelf: count >= 0 ? 'flex-start' : 'flex-end',
              }
            ]}
          />
        </View>
        <Text style={styles.rangeText}>-100 ─── 0 ─── +100</Text>
      </View>

      {/* Step Selector */}
      <View style={styles.stepContainer}>
        <Text style={styles.stepLabel}>Step:</Text>
        <View style={styles.stepOptions}>
          {steps.map(s => (
            <TouchableOpacity
              key={s}
              style={[styles.stepButton, step === s && styles.stepButtonActive]}
              onPress={() => setStep(s)}
            >
              <Text style={[styles.stepText, step === s && styles.stepTextActive]}>
                {s}
              </Text>
            </TouchableOpacity>
          ))}
        </View>
      </View>

      {/* Buttons */}
      <View style={styles.buttonContainer}>
        <TouchableOpacity
          style={[styles.button, styles.decrementButton, count <= MIN && styles.buttonDisabled]}
          onPress={decrement}
          disabled={count <= MIN}
        >
          <Text style={styles.buttonText}>−{step}</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={[styles.button, styles.resetButton]}
          onPress={reset}
        >
          <Text style={styles.resetButtonText}>Reset</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={[styles.button, styles.incrementButton, count >= MAX && styles.buttonDisabled]}
          onPress={increment}
          disabled={count >= MAX}
        >
          <Text style={styles.buttonText}>+{step}</Text>
        </TouchableOpacity>
      </View>

      {/* History */}
      <View style={styles.historyContainer}>
        <Text style={styles.historyTitle}>ประวัติ (10 ล่าสุด)</Text>
        <View style={styles.historyItems}>
          {history.slice().reverse().map((value, index) => (
            <View
              key={index}
              style={[
                styles.historyItem,
                index === 0 && styles.historyItemCurrent,
              ]}
            >
              <Text style={[
                styles.historyValue,
                index === 0 && styles.historyValueCurrent,
              ]}>
                {value > 0 ? '+' : ''}{value}
              </Text>
            </View>
          ))}
        </View>
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    padding: 20,
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#111',
    textAlign: 'center',
    marginBottom: 32,
  },
  countContainer: {
    alignItems: 'center',
    marginBottom: 32,
  },
  count: {
    fontSize: 72,
    fontWeight: '700',
    lineHeight: 84,
    marginBottom: 12,
  },
  progressBar: {
    width: '100%',
    height: 8,
    borderRadius: 4,
    overflow: 'hidden',
  },
  progressFill: {
    height: '100%',
    borderRadius: 4,
  },
  rangeText: {
    fontSize: 12,
    color: '#999',
    marginTop: 6,
  },
  stepContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    gap: 12,
    marginBottom: 28,
  },
  stepLabel: {
    fontSize: 16,
    color: '#666',
    fontWeight: '500',
  },
  stepOptions: {
    flexDirection: 'row',
    gap: 8,
  },
  stepButton: {
    paddingHorizontal: 16,
    paddingVertical: 6,
    borderRadius: 20,
    backgroundColor: '#f0f0f0',
  },
  stepButtonActive: {
    backgroundColor: '#007AFF',
  },
  stepText: {
    fontSize: 14,
    color: '#666',
    fontWeight: '600',
  },
  stepTextActive: {
    color: '#fff',
  },
  buttonContainer: {
    flexDirection: 'row',
    gap: 12,
    marginBottom: 32,
  },
  button: {
    flex: 1,
    height: 56,
    borderRadius: 14,
    justifyContent: 'center',
    alignItems: 'center',
  },
  decrementButton: {
    backgroundColor: '#FF3B30',
  },
  incrementButton: {
    backgroundColor: '#34C759',
  },
  resetButton: {
    flex: 0.8,
    backgroundColor: '#fff',
    borderWidth: 1.5,
    borderColor: '#D1D5DB',
  },
  buttonDisabled: {
    opacity: 0.3,
  },
  buttonText: {
    color: '#fff',
    fontSize: 20,
    fontWeight: '700',
  },
  resetButtonText: {
    color: '#666',
    fontSize: 15,
    fontWeight: '600',
  },
  historyContainer: {
    backgroundColor: '#f9f9f9',
    borderRadius: 12,
    padding: 16,
  },
  historyTitle: {
    fontSize: 14,
    fontWeight: '600',
    color: '#666',
    marginBottom: 10,
  },
  historyItems: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 8,
  },
  historyItem: {
    backgroundColor: '#fff',
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 8,
    borderWidth: 1,
    borderColor: '#e0e0e0',
  },
  historyItemCurrent: {
    borderColor: '#007AFF',
    backgroundColor: '#EBF5FB',
  },
  historyValue: {
    fontSize: 13,
    color: '#666',
  },
  historyValueCurrent: {
    color: '#007AFF',
    fontWeight: '700',
  },
});

export default CounterApp;
```

### Workshop 8.2: Registration Form

```tsx
import React, { useState, useCallback } from 'react';
import {
  View, Text, TextInput, TouchableOpacity, ScrollView,
  StyleSheet, SafeAreaView, KeyboardAvoidingView, Platform, Switch,
} from 'react-native';

type FormField = {
  value: string;
  error: string | null;
  touched: boolean;
};

type RegisterForm = {
  firstName: FormField;
  lastName: FormField;
  email: FormField;
  phone: FormField;
  password: FormField;
  confirmPassword: FormField;
};

const validateField = (name: string, value: string, formData?: RegisterForm): string | null => {
  switch (name) {
    case 'firstName':
    case 'lastName':
      if (!value.trim()) return 'ต้องกรอก';
      if (value.trim().length < 2) return 'ต้องมีอย่างน้อย 2 ตัวอักษร';
      return null;
    
    case 'email':
      if (!value.trim()) return 'ต้องกรอก Email';
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) return 'รูปแบบ Email ไม่ถูกต้อง';
      return null;
    
    case 'phone':
      if (!value.trim()) return 'ต้องกรอกเบอร์โทร';
      if (!/^0[0-9]{8,9}$/.test(value.replace(/[-\s]/g, '')))
        return 'รูปแบบเบอร์โทรไม่ถูกต้อง (0xxxxxxxxx)';
      return null;
    
    case 'password':
      if (!value) return 'ต้องกรอกรหัสผ่าน';
      if (value.length < 8) return 'ต้องมีอย่างน้อย 8 ตัวอักษร';
      if (!/[A-Z]/.test(value)) return 'ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว';
      if (!/[0-9]/.test(value)) return 'ต้องมีตัวเลขอย่างน้อย 1 ตัว';
      return null;
    
    case 'confirmPassword':
      if (!value) return 'ต้องยืนยันรหัสผ่าน';
      if (formData && value !== formData.password.value) return 'รหัสผ่านไม่ตรงกัน';
      return null;
    
    default:
      return null;
  }
};

const createField = (value = ''): FormField => ({ value, error: null, touched: false });

const RegisterScreen = () => {
  const [form, setForm] = useState<RegisterForm>({
    firstName: createField(),
    lastName: createField(),
    email: createField(),
    phone: createField(),
    password: createField(),
    confirmPassword: createField(),
  });
  const [isTermsAccepted, setIsTermsAccepted] = useState(false);
  const [showPassword, setShowPassword] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submitSuccess, setSubmitSuccess] = useState(false);

  const updateField = useCallback((fieldName: keyof RegisterForm) => (value: string) => {
    setForm(prev => {
      const error = validateField(fieldName, value, prev);
      return {
        ...prev,
        [fieldName]: {
          value,
          error: prev[fieldName].touched ? error : null,
          touched: prev[fieldName].touched,
        },
      };
    });
  }, []);

  const blurField = useCallback((fieldName: keyof RegisterForm) => () => {
    setForm(prev => {
      const error = validateField(fieldName, prev[fieldName].value, prev);
      return {
        ...prev,
        [fieldName]: { ...prev[fieldName], error, touched: true },
      };
    });
  }, []);

  const isFormValid = () => {
    const fields = Object.keys(form) as (keyof RegisterForm)[];
    return fields.every(field => {
      const error = validateField(field, form[field].value, form);
      return error === null;
    }) && isTermsAccepted;
  };

  const handleSubmit = async () => {
    // Mark all fields as touched and validate
    setForm(prev => {
      const newForm = { ...prev };
      (Object.keys(newForm) as (keyof RegisterForm)[]).forEach(field => {
        const error = validateField(field, newForm[field].value, prev);
        newForm[field] = { ...newForm[field], error, touched: true };
      });
      return newForm;
    });

    if (!isFormValid()) return;

    setIsSubmitting(true);
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 2000));
    setIsSubmitting(false);
    setSubmitSuccess(true);
  };

  if (submitSuccess) {
    return (
      <SafeAreaView style={[styles.container, { justifyContent: 'center', alignItems: 'center' }]}>
        <Text style={{ fontSize: 64, marginBottom: 16 }}>🎉</Text>
        <Text style={{ fontSize: 24, fontWeight: 'bold', color: '#34C759' }}>
          สมัครสมาชิกสำเร็จ!
        </Text>
        <Text style={{ color: '#666', marginTop: 8 }}>
          ยินดีต้อนรับ {form.firstName.value} {form.lastName.value}
        </Text>
      </SafeAreaView>
    );
  }

  const renderInput = (
    fieldName: keyof RegisterForm,
    label: string,
    props: any = {}
  ) => {
    const field = form[fieldName];
    const isPasswordField = fieldName === 'password' || fieldName === 'confirmPassword';

    return (
      <View style={styles.fieldContainer}>
        <Text style={styles.fieldLabel}>{label}</Text>
        <View style={[
          styles.inputWrapper,
          field.touched && field.error && styles.inputError,
          field.touched && !field.error && styles.inputValid,
        ]}>
          <TextInput
            value={field.value}
            onChangeText={updateField(fieldName)}
            onBlur={blurField(fieldName)}
            secureTextEntry={isPasswordField && !showPassword}
            style={styles.input}
            placeholderTextColor="#9CA3AF"
            {...props}
          />
          {isPasswordField && (
            <TouchableOpacity
              onPress={() => setShowPassword(!showPassword)}
              style={styles.eyeButton}
            >
              <Text style={styles.eyeButtonText}>
                {showPassword ? '🙈' : '👁'}
              </Text>
            </TouchableOpacity>
          )}
        </View>
        {field.touched && field.error && (
          <Text style={styles.errorText}>⚠️ {field.error}</Text>
        )}
      </View>
    );
  };

  return (
    <SafeAreaView style={styles.container}>
      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      >
        <ScrollView
          contentContainerStyle={styles.scrollContent}
          keyboardShouldPersistTaps="handled"
        >
          <Text style={styles.title}>สมัครสมาชิก</Text>
          <Text style={styles.subtitle}>สร้างบัญชีใหม่ของคุณ</Text>

          {/* Name Row */}
          <View style={styles.row}>
            <View style={{ flex: 1 }}>
              {renderInput('firstName', 'ชื่อ', {
                placeholder: 'สมชาย',
                autoCapitalize: 'words',
              })}
            </View>
            <View style={{ flex: 1 }}>
              {renderInput('lastName', 'นามสกุล', {
                placeholder: 'ใจดี',
                autoCapitalize: 'words',
              })}
            </View>
          </View>

          {renderInput('email', 'อีเมล', {
            placeholder: 'somchai@example.com',
            keyboardType: 'email-address',
            autoCapitalize: 'none',
            autoComplete: 'email',
          })}

          {renderInput('phone', 'เบอร์โทรศัพท์', {
            placeholder: '0812345678',
            keyboardType: 'phone-pad',
          })}

          {renderInput('password', 'รหัสผ่าน', {
            placeholder: 'อย่างน้อย 8 ตัว',
          })}

          {renderInput('confirmPassword', 'ยืนยันรหัสผ่าน', {
            placeholder: 'พิมพ์รหัสผ่านอีกครั้ง',
          })}

          {/* Terms */}
          <View style={styles.termsContainer}>
            <Switch
              value={isTermsAccepted}
              onValueChange={setIsTermsAccepted}
              trackColor={{ true: '#007AFF' }}
            />
            <View style={{ flex: 1, marginLeft: 12 }}>
              <Text style={styles.termsText}>
                ฉันยอมรับ{' '}
                <Text style={styles.termsLink}>ข้อกำหนดการใช้บริการ</Text>
                {' '}และ{' '}
                <Text style={styles.termsLink}>นโยบายความเป็นส่วนตัว</Text>
              </Text>
            </View>
          </View>

          {/* Submit Button */}
          <TouchableOpacity
            style={[
              styles.submitButton,
              (!isFormValid() || isSubmitting) && styles.submitDisabled,
            ]}
            onPress={handleSubmit}
            disabled={!isFormValid() || isSubmitting}
          >
            <Text style={styles.submitText}>
              {isSubmitting ? 'กำลังสมัคร...' : 'สมัครสมาชิก'}
            </Text>
          </TouchableOpacity>

          <View style={styles.loginLink}>
            <Text style={styles.loginText}>มีบัญชีแล้ว? </Text>
            <TouchableOpacity>
              <Text style={styles.loginLinkText}>เข้าสู่ระบบ</Text>
            </TouchableOpacity>
          </View>
        </ScrollView>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  scrollContent: { padding: 24, paddingBottom: 40 },
  title: { fontSize: 28, fontWeight: 'bold', color: '#111', marginBottom: 4 },
  subtitle: { fontSize: 16, color: '#666', marginBottom: 28 },
  row: { flexDirection: 'row', gap: 12 },
  fieldContainer: { marginBottom: 16 },
  fieldLabel: {
    fontSize: 14, fontWeight: '500', color: '#374151', marginBottom: 6,
  },
  inputWrapper: {
    flexDirection: 'row',
    alignItems: 'center',
    borderWidth: 1.5,
    borderColor: '#D1D5DB',
    borderRadius: 10,
    backgroundColor: '#FAFAFA',
  },
  inputError: { borderColor: '#FF3B30', backgroundColor: '#FFF5F5' },
  inputValid: { borderColor: '#34C759', backgroundColor: '#F0FFF4' },
  input: {
    flex: 1,
    paddingHorizontal: 14,
    paddingVertical: 12,
    fontSize: 15,
    color: '#111',
  },
  eyeButton: { padding: 12 },
  eyeButtonText: { fontSize: 16 },
  errorText: { marginTop: 4, fontSize: 12, color: '#FF3B30' },
  termsContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 24,
    padding: 12,
    backgroundColor: '#F9FAFB',
    borderRadius: 10,
  },
  termsText: { fontSize: 13, color: '#374151', lineHeight: 18 },
  termsLink: { color: '#007AFF', fontWeight: '500' },
  submitButton: {
    backgroundColor: '#007AFF',
    borderRadius: 12,
    paddingVertical: 15,
    alignItems: 'center',
    marginBottom: 20,
  },
  submitDisabled: { backgroundColor: '#BFDBFE' },
  submitText: { color: '#fff', fontSize: 16, fontWeight: '700' },
  loginLink: { flexDirection: 'row', justifyContent: 'center' },
  loginText: { color: '#6B7280' },
  loginLinkText: { color: '#007AFF', fontWeight: '600' },
});

export default RegisterScreen;
```

---

## Tips และ Best Practices

### 1. Functional Updates

```tsx
// ✅ ใช้ functional update เมื่อ new state ขึ้นกับ old state
setCount(prev => prev + 1);
setItems(prev => [...prev, newItem]);

// ❌ อย่าใช้ค่าโดยตรงเมื่อ multiple updates
setCount(count + 1);  // อาจไม่ได้ค่าล่าสุดใน batched updates
```

### 2. ไม่ Mutate State โดยตรง

```tsx
// ❌ ไม่ได้ - mutate directly
const addUser = () => {
  users.push(newUser);  // Mutate!
  setUsers(users);      // React ไม่ detect การเปลี่ยนแปลง
};

// ✅ ถูกต้อง - สร้าง array ใหม่
const addUser = () => {
  setUsers(prev => [...prev, newUser]);
};
```

### 3. Batch State Updates

```tsx
// React 18: batches automatically
const handleAction = () => {
  setLoading(true);    // ┐
  setError(null);      // ├── batched เป็น 1 render
  setData(newData);    // ┘
};
```

---

## สรุป Part 008

### ได้เรียนรู้

1. **Props** - ข้อมูลจาก parent ถึง child, read-only
2. **PropTypes** - validate props ใน JavaScript
3. **TypeScript Props** - type-safe props
4. **useState** - จัดการ local state
5. **Complex State** - object, array state updates
6. **Lifting State Up** - share state ผ่าน parent
7. **Controlled Components** - React เป็น source of truth
8. **useReducer** - state machine สำหรับ logic ซับซ้อน

---

**ต่อไป → Part 009: Event Handling และ User Interaction**
