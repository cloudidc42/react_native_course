# Part 005: JSX และ Components เบื้องต้น

## สารบัญ
1. [JSX คืออะไร](#jsx-คืออะไร)
2. [JSX Rules และ Syntax](#jsx-rules)
3. [Functional Components](#functional-components)
4. [Class Components (Legacy)](#class-components)
5. [Component Composition](#component-composition)
6. [Fragment](#fragment)
7. [Conditional Rendering](#conditional-rendering)
8. [List Rendering](#list-rendering)
9. [Workshop: สร้าง Reusable Components](#workshop)

---

## JSX คืออะไร

JSX (JavaScript XML) คือ syntax extension ของ JavaScript ที่ช่วยให้เขียน UI markup ร่วมกับ JavaScript ได้

### JSX ไม่ใช่ HTML

```jsx
// JSX - ใช้ className ไม่ได้! ใช้ style แทน
// และไม่มี <div>, <span> ใน React Native

// ❌ HTML - ใช้ใน Web เท่านั้น
<div class="container">
  <span>Hello</span>
</div>

// ✅ JSX ใน React Native
<View style={styles.container}>
  <Text>Hello</Text>
</View>
```

### Babel แปลง JSX

Babel แปลง JSX เป็น JavaScript ธรรมดา:

```jsx
// ก่อน Babel (JSX)
const element = <Text style={styles.text}>Hello World</Text>;

// หลัง Babel (Pure JavaScript)
const element = React.createElement(
  Text,
  { style: styles.text },
  'Hello World'
);
```

ดังนั้น React ต้องอยู่ใน scope เสมอ (ก่อน React 17)

### JSX เป็น Expression

JSX เป็น JavaScript expression ที่ return ค่าได้:

```tsx
const App = () => {
  // JSX สามารถใช้ใน variable
  const greeting = <Text>สวัสดี!</Text>;

  // JSX สามารถ return จาก function
  const renderButton = () => (
    <TouchableOpacity>
      <Text>กดปุ่ม</Text>
    </TouchableOpacity>
  );

  // JSX สามารถใช้ใน ternary
  const isLoggedIn = true;
  const content = isLoggedIn ? <Text>ยินดีต้อนรับ!</Text> : <Text>กรุณาเข้าสู่ระบบ</Text>;

  return (
    <View>
      {greeting}
      {renderButton()}
      {content}
    </View>
  );
};
```

---

## JSX Rules

### Rule 1: ต้องมี Root Element เดียว

```tsx
// ❌ ไม่ได้ - หลาย root elements
const BadComponent = () => {
  return (
    <Text>First</Text>
    <Text>Second</Text>
  );
};

// ✅ ดี - ห่อด้วย View
const GoodComponent1 = () => {
  return (
    <View>
      <Text>First</Text>
      <Text>Second</Text>
    </View>
  );
};

// ✅ ดี - ใช้ Fragment
const GoodComponent2 = () => {
  return (
    <>
      <Text>First</Text>
      <Text>Second</Text>
    </>
  );
};
```

### Rule 2: Tags ต้อง Close เสมอ

```tsx
// ❌ ไม่ได้ - ไม่ปิด tag
<Image source={require('./img.png')}>

// ✅ ดี - Self-closing
<Image source={require('./img.png')} />

// ✅ ดี - Open + Close
<View></View>
```

### Rule 3: JSX ใส่ใน Parentheses สำหรับ multiline

```tsx
// ✅ ดี - parentheses สำหรับ multiline
const App = () => (
  <View>
    <Text>Hello</Text>
  </View>
);

// ✅ ก็ได้ - return statement
const App = () => {
  return (
    <View>
      <Text>Hello</Text>
    </View>
  );
};
```

### Rule 4: Expression ต้องใช้ {} Curly Braces

```tsx
const App = () => {
  const name = 'สมชาย';
  const age = 25;
  const isActive = true;

  return (
    <View>
      {/* ✅ Variable */}
      <Text>{name}</Text>

      {/* ✅ Expression */}
      <Text>{age + 1}</Text>

      {/* ✅ Function call */}
      <Text>{name.toUpperCase()}</Text>

      {/* ✅ Template literal */}
      <Text>{`ชื่อ: ${name}, อายุ: ${age}`}</Text>

      {/* ✅ Ternary */}
      <Text>{isActive ? 'ใช้งานอยู่' : 'ไม่ได้ใช้งาน'}</Text>

      {/* ✅ Short circuit */}
      {isActive && <Text>Active User</Text>}

      {/* ❌ Statement ใช้ใน {} ไม่ได้ */}
      {/* {if (isActive) { return <Text>Active</Text> }} */}
    </View>
  );
};
```

### Rule 5: Attributes ใช้ camelCase

```tsx
// HTML attributes → JSX attributes
// class → style (React Native ไม่มี className)
// onclick → onPress (React Native)
// tabindex → tabIndex

const App = () => (
  <View
    style={styles.container}        // camelCase
    testID="container"              // camelCase
    accessibilityLabel="เนื้อหา"    // camelCase
  >
    <TextInput
      autoCorrect={false}           // camelCase
      autoCapitalize="none"
      keyboardType="email-address"
      onChangeText={handleChange}
    />
  </View>
);
```

### Rule 6: Comments ใน JSX

```tsx
const App = () => (
  <View>
    {/* Comment ใน JSX ต้องใช้แบบนี้ */}
    <Text>Hello</Text>

    {/*
      หลายบรรทัด
      ก็ได้
    */}
    <Text>World</Text>
  </View>
);
```

---

## Functional Components

Functional Components คือ JavaScript functions ที่ return JSX

### รูปแบบพื้นฐาน

```tsx
import React from 'react';
import { View, Text } from 'react-native';

// Pattern 1: Function declaration
function MyComponent() {
  return (
    <View>
      <Text>Hello from Function Declaration</Text>
    </View>
  );
}

// Pattern 2: Arrow function (แนะนำ)
const MyComponent = () => {
  return (
    <View>
      <Text>Hello from Arrow Function</Text>
    </View>
  );
};

// Pattern 3: Arrow function ย่อ (ไม่มี logic)
const MyComponent = () => (
  <View>
    <Text>Hello from Concise Arrow</Text>
  </View>
);

// Pattern 4: TypeScript + Type annotation
const MyComponent: React.FC = () => {
  return (
    <View>
      <Text>Hello with TypeScript</Text>
    </View>
  );
};
```

### Props ใน Functional Components

```tsx
// TypeScript: กำหนด type ของ props
type GreetingProps = {
  name: string;
  age?: number;           // Optional prop
  isAdmin?: boolean;
  onPress: () => void;
  children?: React.ReactNode;
};

const Greeting: React.FC<GreetingProps> = ({
  name,
  age = 0,              // Default value
  isAdmin = false,
  onPress,
  children,
}) => {
  return (
    <TouchableOpacity onPress={onPress}>
      <View>
        <Text>สวัสดี, {name}!</Text>
        {age > 0 && <Text>อายุ: {age} ปี</Text>}
        {isAdmin && <Text style={{ color: 'red' }}>Admin</Text>}
        {children}
      </View>
    </TouchableOpacity>
  );
};

// ใช้งาน
const App = () => (
  <View>
    <Greeting
      name="สมชาย"
      age={25}
      isAdmin={true}
      onPress={() => console.log('กด!')}
    >
      <Text>This is a child</Text>
    </Greeting>
  </View>
);
```

### useState Hook

```tsx
import React, { useState } from 'react';
import { View, Text, TouchableOpacity } from 'react-native';

const Counter = () => {
  // [ค่าปัจจุบัน, function ที่ใช้เปลี่ยนค่า] = useState(ค่าเริ่มต้น)
  const [count, setCount] = useState(0);
  const [name, setName] = useState('สมชาย');
  const [isVisible, setIsVisible] = useState(true);

  return (
    <View>
      <Text>Count: {count}</Text>
      
      <TouchableOpacity onPress={() => setCount(count + 1)}>
        <Text>+ เพิ่ม</Text>
      </TouchableOpacity>
      
      <TouchableOpacity onPress={() => setCount(prev => prev - 1)}>
        {/* ใช้ functional update เมื่อ state ใหม่ขึ้นกับ state เก่า */}
        <Text>- ลด</Text>
      </TouchableOpacity>

      <TouchableOpacity onPress={() => setCount(0)}>
        <Text>Reset</Text>
      </TouchableOpacity>
    </View>
  );
};
```

### useEffect Hook

```tsx
import React, { useState, useEffect } from 'react';
import { View, Text } from 'react-native';

const DataFetcher = () => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [userId, setUserId] = useState(1);

  // ทำงานเมื่อ component mount ([] = run once)
  useEffect(() => {
    console.log('Component mounted');
    
    return () => {
      console.log('Component unmounted (cleanup)');
    };
  }, []);

  // ทำงานเมื่อ userId เปลี่ยน
  useEffect(() => {
    const fetchUser = async () => {
      setLoading(true);
      try {
        const response = await fetch(
          `https://jsonplaceholder.typicode.com/users/${userId}`
        );
        const json = await response.json();
        setData(json);
      } catch (error) {
        console.error('Error:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
  }, [userId]); // dependency array

  if (loading) {
    return <Text>กำลังโหลด...</Text>;
  }

  return (
    <View>
      <Text>{data?.name}</Text>
    </View>
  );
};
```

### React.memo สำหรับ Performance

```tsx
import React, { memo } from 'react';

// Component จะ re-render เฉพาะเมื่อ props เปลี่ยน
const ExpensiveComponent = memo(({ data, onPress }) => {
  console.log('Rendering ExpensiveComponent');
  
  return (
    <View>
      <Text>{data.title}</Text>
      <TouchableOpacity onPress={onPress}>
        <Text>กด</Text>
      </TouchableOpacity>
    </View>
  );
});

// Custom comparison function
const AreEqual = (prevProps, nextProps) => {
  return prevProps.data.id === nextProps.data.id;
};

const ExpensiveComponentWithCustomComparison = memo(
  MyComponent,
  AreEqual
);
```

---

## Class Components (Legacy)

Class Components เป็นรูปแบบเก่าที่ใช้ก่อน Hooks จะมา

> ⚠️ **หมายเหตุ**: ปัจจุบันนิยมใช้ Functional Components + Hooks แทน แต่ควรรู้ไว้เพราะอาจเจอใน codebase เก่า

### รูปแบบ Class Component

```tsx
import React, { Component } from 'react';
import { View, Text, TouchableOpacity } from 'react-native';

// กำหนด types สำหรับ TypeScript
type Props = {
  initialCount?: number;
};

type State = {
  count: number;
  isVisible: boolean;
};

class CounterClass extends Component<Props, State> {
  // Constructor (ถ้าต้องการ)
  constructor(props: Props) {
    super(props);
    // กำหนด state เริ่มต้น
    this.state = {
      count: props.initialCount ?? 0,
      isVisible: true,
    };
    
    // Bind methods (สำหรับ non-arrow function)
    this.handleIncrement = this.handleIncrement.bind(this);
  }

  // Lifecycle Methods
  componentDidMount() {
    // ทำงานเมื่อ component mount (เหมือน useEffect(fn, []))
    console.log('Mounted');
  }

  componentDidUpdate(prevProps: Props, prevState: State) {
    // ทำงานเมื่อ props หรือ state เปลี่ยน
    if (prevState.count !== this.state.count) {
      console.log('Count changed:', this.state.count);
    }
  }

  componentWillUnmount() {
    // Cleanup เมื่อ component จะ unmount
    console.log('Will unmount');
  }

  // Methods
  handleIncrement() {
    // setState ไม่ควร mutate state โดยตรง
    this.setState(prevState => ({
      count: prevState.count + 1,
    }));
  }

  // Arrow function method (ไม่ต้อง bind)
  handleDecrement = () => {
    this.setState(prevState => ({
      count: prevState.count - 1,
    }));
  };

  handleReset = () => {
    this.setState({ count: 0 });
  };

  // Render
  render() {
    const { count, isVisible } = this.state;

    return (
      <View>
        {isVisible && <Text>Count: {count}</Text>}
        
        <TouchableOpacity onPress={this.handleIncrement}>
          <Text>+ เพิ่ม</Text>
        </TouchableOpacity>
        
        <TouchableOpacity onPress={this.handleDecrement}>
          <Text>- ลด</Text>
        </TouchableOpacity>
        
        <TouchableOpacity onPress={this.handleReset}>
          <Text>Reset</Text>
        </TouchableOpacity>
      </View>
    );
  }
}

export default CounterClass;
```

### เปรียบเทียบ Class vs Functional

```tsx
// Class Component - เก่า
class UserProfile extends Component {
  state = {
    user: null,
    loading: true,
  };

  async componentDidMount() {
    const user = await fetchUser();
    this.setState({ user, loading: false });
  }

  render() {
    const { user, loading } = this.state;
    if (loading) return <Text>Loading...</Text>;
    return <Text>{user.name}</Text>;
  }
}

// Functional Component - ใหม่ (แนะนำ)
const UserProfile = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser().then(user => {
      setUser(user);
      setLoading(false);
    });
  }, []);

  if (loading) return <Text>Loading...</Text>;
  return <Text>{user?.name}</Text>;
};
```

---

## Component Composition

### Props Drilling

```tsx
// ปัญหา: Props Drilling - ส่ง props ผ่านหลายชั้น
const App = () => {
  const user = { name: 'สมชาย', role: 'admin' };
  
  return <Layout user={user} />;
};

const Layout = ({ user }) => (
  <View>
    <Header user={user} />
    <Content user={user} />
  </View>
);

const Header = ({ user }) => (
  <View>
    <UserAvatar user={user} />  {/* user ส่งต่อไปอีก */}
  </View>
);

const UserAvatar = ({ user }) => (
  <View>
    <Text>{user.name}</Text>  {/* ใช้จริงที่นี่ */}
  </View>
);
```

### Composition Pattern

```tsx
// Solution: Composition - ส่ง component เป็น children หรือ props
const App = () => {
  const user = { name: 'สมชาย', role: 'admin' };
  
  return (
    <Layout
      header={<Header user={user} />}  // ส่ง component เป็น prop
      content={<Content user={user} />}
    />
  );
};

const Layout = ({ header, content }) => (
  <View>
    {header}   {/* Render header component */}
    {content}
  </View>
);

// หรือใช้ children
const Card = ({ title, children }) => (
  <View style={styles.card}>
    <Text style={styles.title}>{title}</Text>
    {children}
  </View>
);

// ใช้งาน
const App = () => (
  <Card title="ชื่อการ์ด">
    <Text>เนื้อหาการ์ด</Text>
    <Button label="ปุ่ม" />
  </Card>
);
```

### Higher-Order Components (HOC)

```tsx
// HOC: function ที่รับ component แล้ว return component ใหม่
type WithLoadingProps = {
  isLoading: boolean;
};

function withLoading<P extends object>(
  WrappedComponent: React.ComponentType<P>
) {
  return function WithLoadingComponent(props: P & WithLoadingProps) {
    const { isLoading, ...restProps } = props;

    if (isLoading) {
      return (
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <ActivityIndicator size="large" />
        </View>
      );
    }

    return <WrappedComponent {...(restProps as P)} />;
  };
}

// ใช้งาน
const UserList = ({ users }) => (
  <FlatList data={users} renderItem={...} />
);

const UserListWithLoading = withLoading(UserList);

// ใน App
<UserListWithLoading isLoading={loading} users={users} />
```

### Render Props Pattern

```tsx
type MouseTrackerProps = {
  render: (position: { x: number; y: number }) => React.ReactNode;
};

// Component ที่ใช้ render prop
const MouseTracker: React.FC<MouseTrackerProps> = ({ render }) => {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  return (
    <View
      onTouchMove={(e) => {
        setPosition({
          x: e.nativeEvent.locationX,
          y: e.nativeEvent.locationY,
        });
      }}
      style={{ flex: 1 }}
    >
      {render(position)}
    </View>
  );
};

// ใช้งาน
const App = () => (
  <MouseTracker
    render={({ x, y }) => (
      <Text>Position: {x}, {y}</Text>
    )}
  />
);
```

### Custom Hooks Pattern

```tsx
// Custom Hook - logic ที่ใช้ซ้ำได้
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

// ใช้งาน
const MyComponent = () => {
  const { width, height } = useWindowSize();
  
  return (
    <Text>Screen: {width} x {height}</Text>
  );
};
```

---

## Fragment

Fragment ช่วยให้ group elements โดยไม่ต้องเพิ่ม element พิเศษ

### ทำไมต้องใช้ Fragment

```tsx
// ปัญหา: View พิเศษที่ไม่จำเป็น
const UserInfo = () => (
  <View>  {/* View นี้ไม่มีประโยชน์ */}
    <Text>ชื่อ: สมชาย</Text>
    <Text>อายุ: 25</Text>
  </View>
);

// Solution: ใช้ Fragment
const UserInfo = () => (
  <>
    <Text>ชื่อ: สมชาย</Text>
    <Text>อายุ: 25</Text>
  </>
);

// หรือ explicit Fragment
const UserInfo = () => (
  <React.Fragment>
    <Text>ชื่อ: สมชาย</Text>
    <Text>อายุ: 25</Text>
  </React.Fragment>
);
```

### Fragment ใน List (ต้องใช้ key)

```tsx
const itemList = [
  { id: 1, title: 'Item 1', description: 'Desc 1' },
  { id: 2, title: 'Item 2', description: 'Desc 2' },
];

const ItemList = () => (
  <View>
    {itemList.map(item => (
      // ต้องใช้ React.Fragment เมื่อต้องการ key
      <React.Fragment key={item.id}>
        <Text style={styles.title}>{item.title}</Text>
        <Text style={styles.desc}>{item.description}</Text>
        <View style={styles.divider} />
      </React.Fragment>
    ))}
  </View>
);
```

---

## Conditional Rendering

### 1. If/Else ใน render

```tsx
const UserGreeting = ({ user, isLoggedIn }) => {
  // Pattern 1: if/else ก่อน return
  if (!isLoggedIn) {
    return <Text>กรุณาเข้าสู่ระบบ</Text>;
  }

  if (!user) {
    return <ActivityIndicator />;
  }

  return <Text>สวัสดี {user.name}!</Text>;
};
```

### 2. Ternary Operator

```tsx
const StatusBadge = ({ isOnline }) => (
  <View style={[
    styles.badge,
    isOnline ? styles.online : styles.offline,
  ]}>
    <Text>{isOnline ? 'ออนไลน์' : 'ออฟไลน์'}</Text>
  </View>
);
```

### 3. Short Circuit (&&)

```tsx
const NotificationBadge = ({ count }) => (
  <View>
    <BellIcon />
    {/* แสดงเฉพาะเมื่อ count > 0 */}
    {count > 0 && (
      <View style={styles.badge}>
        <Text style={styles.badgeText}>{count}</Text>
      </View>
    )}
  </View>
);
```

### 4. Switch Statement

```tsx
type UserRole = 'admin' | 'editor' | 'viewer';

const RoleBadge = ({ role }: { role: UserRole }) => {
  const renderBadge = () => {
    switch (role) {
      case 'admin':
        return <View style={styles.adminBadge}><Text>Admin</Text></View>;
      case 'editor':
        return <View style={styles.editorBadge}><Text>Editor</Text></View>;
      case 'viewer':
        return <View style={styles.viewerBadge}><Text>Viewer</Text></View>;
      default:
        return null;
    }
  };

  return renderBadge();
};
```

### 5. Lookup Object Pattern

```tsx
type Status = 'loading' | 'success' | 'error' | 'empty';

const StatusComponent = ({ status }: { status: Status }) => {
  const statusComponents = {
    loading: <ActivityIndicator size="large" />,
    success: <Text>สำเร็จ!</Text>,
    error: <Text style={{ color: 'red' }}>เกิดข้อผิดพลาด</Text>,
    empty: <Text>ไม่มีข้อมูล</Text>,
  };

  return statusComponents[status] ?? null;
};
```

---

## List Rendering

### map() พื้นฐาน

```tsx
const fruits = ['แอปเปิ้ล', 'กล้วย', 'ส้ม', 'มะม่วง'];

const FruitList = () => (
  <View>
    {fruits.map((fruit, index) => (
      // key ต้อง unique และ stable
      <Text key={index}>{fruit}</Text>
    ))}
  </View>
);
```

### key prop ที่ดี

```tsx
const users = [
  { id: 'u1', name: 'สมชาย' },
  { id: 'u2', name: 'สมหญิง' },
  { id: 'u3', name: 'สมใจ' },
];

const UserList = () => (
  <View>
    {users.map(user => (
      // ✅ ดี - ใช้ unique ID เป็น key
      <Text key={user.id}>{user.name}</Text>
    ))}
  </View>
);

// ❌ ไม่ดี - ใช้ index เมื่อ list อาจเปลี่ยนลำดับ
const BadList = () => (
  <View>
    {users.map((user, index) => (
      <Text key={index}>{user.name}</Text>  // อาจทำให้ bug
    ))}
  </View>
);
```

### filter() + map()

```tsx
const products = [
  { id: 1, name: 'สินค้า 1', price: 100, inStock: true },
  { id: 2, name: 'สินค้า 2', price: 200, inStock: false },
  { id: 3, name: 'สินค้า 3', price: 300, inStock: true },
];

const InStockProducts = () => (
  <View>
    {products
      .filter(product => product.inStock)
      .map(product => (
        <View key={product.id}>
          <Text>{product.name}</Text>
          <Text>฿{product.price}</Text>
        </View>
      ))}
  </View>
);
```

---

## Workshop: สร้าง Reusable Components

### Workshop 5.1: Button Component Library

```tsx
// src/components/common/Button/Button.tsx
import React from 'react';
import {
  TouchableOpacity,
  Text,
  ActivityIndicator,
  View,
  StyleSheet,
  ViewStyle,
  TextStyle,
} from 'react-native';

// Types
export type ButtonVariant = 'solid' | 'outline' | 'ghost' | 'destructive';
export type ButtonSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

export interface ButtonProps {
  label: string;
  onPress?: () => void;
  variant?: ButtonVariant;
  size?: ButtonSize;
  disabled?: boolean;
  loading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  fullWidth?: boolean;
  style?: ViewStyle;
  labelStyle?: TextStyle;
}

const Button: React.FC<ButtonProps> = ({
  label,
  onPress,
  variant = 'solid',
  size = 'md',
  disabled = false,
  loading = false,
  leftIcon,
  rightIcon,
  fullWidth = false,
  style,
  labelStyle,
}) => {
  const isDisabled = disabled || loading;

  return (
    <TouchableOpacity
      style={[
        styles.base,
        styles[`variant_${variant}`],
        styles[`size_${size}`],
        fullWidth && styles.fullWidth,
        isDisabled && styles.disabled,
        style,
      ]}
      onPress={onPress}
      disabled={isDisabled}
      activeOpacity={0.75}
    >
      {loading ? (
        <ActivityIndicator
          color={variant === 'solid' || variant === 'destructive' ? '#fff' : '#007AFF'}
          size="small"
        />
      ) : (
        <View style={styles.content}>
          {leftIcon && <View style={styles.iconLeft}>{leftIcon}</View>}
          <Text
            style={[
              styles.label,
              styles[`label_${variant}`],
              styles[`labelSize_${size}`],
              labelStyle,
            ]}
          >
            {label}
          </Text>
          {rightIcon && <View style={styles.iconRight}>{rightIcon}</View>}
        </View>
      )}
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  base: {
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
    overflow: 'hidden',
  },
  fullWidth: {
    alignSelf: 'stretch',
  },
  disabled: {
    opacity: 0.5,
  },
  content: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  iconLeft: {
    marginRight: 8,
  },
  iconRight: {
    marginLeft: 8,
  },

  // Variants
  variant_solid: {
    backgroundColor: '#007AFF',
  },
  variant_outline: {
    backgroundColor: 'transparent',
    borderWidth: 1.5,
    borderColor: '#007AFF',
  },
  variant_ghost: {
    backgroundColor: 'transparent',
  },
  variant_destructive: {
    backgroundColor: '#FF3B30',
  },

  // Label Variants
  label: {
    fontWeight: '600',
    textAlign: 'center',
  },
  label_solid: { color: '#fff' },
  label_outline: { color: '#007AFF' },
  label_ghost: { color: '#007AFF' },
  label_destructive: { color: '#fff' },

  // Sizes
  size_xs: { paddingHorizontal: 10, paddingVertical: 4 },
  size_sm: { paddingHorizontal: 14, paddingVertical: 7 },
  size_md: { paddingHorizontal: 18, paddingVertical: 11 },
  size_lg: { paddingHorizontal: 22, paddingVertical: 14 },
  size_xl: { paddingHorizontal: 28, paddingVertical: 17 },

  // Label Sizes
  labelSize_xs: { fontSize: 11 },
  labelSize_sm: { fontSize: 13 },
  labelSize_md: { fontSize: 15 },
  labelSize_lg: { fontSize: 17 },
  labelSize_xl: { fontSize: 19 },
});

export default Button;
```

### Workshop 5.2: Card Component

```tsx
// src/components/common/Card/Card.tsx
import React from 'react';
import { View, Text, Image, TouchableOpacity, StyleSheet, ViewStyle } from 'react-native';

interface CardProps {
  title: string;
  description?: string;
  imageUrl?: string;
  badge?: string;
  onPress?: () => void;
  footer?: React.ReactNode;
  style?: ViewStyle;
}

const Card: React.FC<CardProps> = ({
  title,
  description,
  imageUrl,
  badge,
  onPress,
  footer,
  style,
}) => {
  const Container = onPress ? TouchableOpacity : View;

  return (
    <Container
      style={[styles.card, style]}
      onPress={onPress}
      activeOpacity={0.9}
    >
      {/* Image */}
      {imageUrl && (
        <View>
          <Image source={{ uri: imageUrl }} style={styles.image} />
          {badge && (
            <View style={styles.badge}>
              <Text style={styles.badgeText}>{badge}</Text>
            </View>
          )}
        </View>
      )}

      {/* Content */}
      <View style={styles.content}>
        <Text style={styles.title} numberOfLines={2}>
          {title}
        </Text>
        {description && (
          <Text style={styles.description} numberOfLines={3}>
            {description}
          </Text>
        )}
      </View>

      {/* Footer */}
      {footer && (
        <View style={styles.footer}>
          {footer}
        </View>
      )}
    </Container>
  );
};

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.08,
    shadowRadius: 8,
    elevation: 3,
  },
  image: {
    width: '100%',
    height: 180,
    resizeMode: 'cover',
  },
  badge: {
    position: 'absolute',
    top: 12,
    right: 12,
    backgroundColor: '#FF3B30',
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 12,
  },
  badgeText: {
    color: '#fff',
    fontSize: 11,
    fontWeight: '700',
  },
  content: {
    padding: 16,
  },
  title: {
    fontSize: 17,
    fontWeight: '700',
    color: '#1a1a1a',
    lineHeight: 22,
  },
  description: {
    marginTop: 6,
    fontSize: 14,
    color: '#666',
    lineHeight: 20,
  },
  footer: {
    paddingHorizontal: 16,
    paddingBottom: 16,
    paddingTop: 0,
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    marginTop: 4,
    paddingTop: 12,
  },
});

export default Card;
```

### Workshop 5.3: Avatar Component

```tsx
// src/components/common/Avatar/Avatar.tsx
import React from 'react';
import { View, Text, Image, StyleSheet, ViewStyle } from 'react-native';

type AvatarSize = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

interface AvatarProps {
  size?: AvatarSize;
  imageUrl?: string;
  name?: string;
  style?: ViewStyle;
}

const sizeMap = {
  xs: 24,
  sm: 32,
  md: 40,
  lg: 56,
  xl: 72,
};

const fontSizeMap = {
  xs: 10,
  sm: 13,
  md: 16,
  lg: 22,
  xl: 28,
};

// สร้าง initials จากชื่อ
const getInitials = (name: string): string => {
  return name
    .split(' ')
    .map(word => word.charAt(0))
    .slice(0, 2)
    .join('')
    .toUpperCase();
};

// สร้างสีจากชื่อ
const getColorFromName = (name: string): string => {
  const colors = [
    '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4',
    '#FFEAA7', '#DDA0DD', '#98D8C8', '#F7DC6F',
  ];
  let hash = 0;
  for (let i = 0; i < name.length; i++) {
    hash = name.charCodeAt(i) + ((hash << 5) - hash);
  }
  return colors[Math.abs(hash) % colors.length];
};

const Avatar: React.FC<AvatarProps> = ({
  size = 'md',
  imageUrl,
  name = '',
  style,
}) => {
  const dimension = sizeMap[size];
  const fontSize = fontSizeMap[size];
  const initials = getInitials(name);
  const bgColor = getColorFromName(name);

  return (
    <View
      style={[
        styles.container,
        {
          width: dimension,
          height: dimension,
          borderRadius: dimension / 2,
          backgroundColor: imageUrl ? 'transparent' : bgColor,
        },
        style,
      ]}
    >
      {imageUrl ? (
        <Image
          source={{ uri: imageUrl }}
          style={[
            styles.image,
            { borderRadius: dimension / 2 },
          ]}
        />
      ) : (
        <Text style={[styles.initials, { fontSize }]}>
          {initials}
        </Text>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    justifyContent: 'center',
    alignItems: 'center',
    overflow: 'hidden',
  },
  image: {
    width: '100%',
    height: '100%',
  },
  initials: {
    color: '#fff',
    fontWeight: '600',
  },
});

export default Avatar;
```

### Workshop 5.4: แบบฝึกหัด

**แบบฝึกหัด 1:** สร้าง `Badge` component ที่:
- รองรับหลาย variants: `success`, `warning`, `error`, `info`
- มี prop `count` ที่แสดงตัวเลข
- มี prop `label` สำหรับ text

**แบบฝึกหัด 2:** สร้าง `Divider` component ที่:
- รองรับ horizontal และ vertical
- มี prop สำหรับสี
- มี prop สำหรับ spacing

**แบบฝึกหัด 3:** นำ components ทั้งหมดมาสร้าง "Product Listing Screen":
- Product Card ที่มีรูป, ชื่อ, ราคา
- Badge สำหรับแสดง "ลด 20%"
- Button สำหรับ "เพิ่มลงตะกร้า"
- Avatar ของ seller

---

## Tips และ Best Practices

### 1. Single Responsibility Principle

```tsx
// ❌ Component ทำหลายอย่าง
const UserDashboard = () => {
  // fetch users
  // handle auth
  // manage cart
  // show notifications
  // render everything
};

// ✅ แยก concerns
const UserDashboard = () => (
  <View>
    <UserProfile />     {/* รับผิดชอบ profile */}
    <CartSummary />     {/* รับผิดชอบ cart */}
    <Notifications />   {/* รับผิดชอบ notifications */}
  </View>
);
```

### 2. Composition Over Props

```tsx
// ❌ Props มากเกินไป
<Card
  title="..."
  subtitle="..."
  imageUrl="..."
  actions={[{label: 'Share'}, {label: 'Delete'}]}
  footer="..."
  badge="..."
  isHighlighted={true}
  onPress={...}
  onLongPress={...}
/>

// ✅ Composition
<Card>
  <Card.Image source="..." />
  <Card.Content>
    <Card.Title>...</Card.Title>
    <Card.Subtitle>...</Card.Subtitle>
  </Card.Content>
  <Card.Actions>
    <Button label="Share" />
    <Button label="Delete" variant="destructive" />
  </Card.Actions>
</Card>
```

### 3. Default Props

```tsx
// ✅ Default values ผ่าน destructuring
const Button = ({
  variant = 'solid',
  size = 'md',
  disabled = false,
  ...props
}) => {};

// หรือ defaultProps (legacy)
Button.defaultProps = {
  variant: 'solid',
  size: 'md',
  disabled: false,
};
```

---

## สรุป Part 005

### ได้เรียนรู้

1. **JSX** - syntax extension ที่แปลงเป็น React.createElement()
2. **JSX Rules** - root element เดียว, tags ปิด, camelCase attributes
3. **Functional Components** - function ที่ return JSX (แนะนำ)
4. **Class Components** - รูปแบบเก่า ยังต้องรู้ไว้
5. **Component Composition** - HOC, Render Props, Custom Hooks
6. **Fragment** - group elements โดยไม่เพิ่ม DOM
7. **Conditional Rendering** - if/else, ternary, &&
8. **List Rendering** - map() พร้อม key prop

---

**ต่อไป → Part 006: Core Components: View, Text, Image, TextInput, Button**
