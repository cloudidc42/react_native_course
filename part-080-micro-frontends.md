# Part 080: Micro Frontends

## Micro Frontends คืออะไร?

Micro Frontends คือ architectural pattern ที่แบ่ง frontend application ออกเป็น feature teams ที่อิสระต่อกัน แต่ละ team รับผิดชอบ feature ของตัวเองตั้งแต่ UI จนถึง backend service

## ประโยชน์ของ Micro Frontends

1. **Independent Development** - แต่ละ team ทำงานอิสระ
2. **Independent Deployment** - deploy ได้โดยไม่ต้องรอ team อื่น
3. **Technology Diversity** - แต่ละ feature ใช้ tech stack ที่เหมาะสม
4. **Smaller Codebases** - easier to understand and maintain
5. **Fault Isolation** - bug ใน feature หนึ่งไม่กระทบ feature อื่น

---

## 1. Micro Frontend Patterns สำหรับ React Native

### 1.1 Feature-based Module Structure

```
my-app/
├── app/                    # Host app (shell)
│   ├── App.tsx
│   ├── navigation/
│   └── shared/
├── features/
│   ├── home/               # Feature module
│   │   ├── package.json
│   │   ├── src/
│   │   └── index.ts
│   ├── profile/
│   │   ├── package.json
│   │   ├── src/
│   │   └── index.ts
│   ├── products/
│   │   ├── package.json
│   │   ├── src/
│   │   └── index.ts
│   └── checkout/
│       ├── package.json
│       ├── src/
│       └── index.ts
└── shared/
    ├── ui-components/
    ├── utils/
    └── types/
```

### 1.2 Feature Module Structure

```typescript
// features/home/index.ts
export { default as HomeScreen } from './src/screens/HomeScreen';
export { default as HomeNavigator } from './src/navigation/HomeNavigator';
export { homeReducer } from './src/store/homeSlice';
export type { HomeState } from './src/types';

// features/home/package.json
{
  "name": "@myapp/feature-home",
  "version": "1.0.0",
  "main": "index.ts",
  "peerDependencies": {
    "react": "^18.0.0",
    "react-native": "^0.71.0",
    "@myapp/shared-ui": "^1.0.0"
  }
}
```

---

## 2. Module Federation

### 2.1 Setup Module Federation

```bash
# ติดตั้ง webpack plugin
npm install --save-dev @callstack/repack

# หรือใช้ rspack
npm install --save-dev @callstack/repack
```

### 2.2 Host App Configuration

```javascript
// webpack.config.js (Host App)
const { ModuleFederationPlugin } = require('@callstack/repack/mf');
const path = require('path');

module.exports = (env) => {
  return {
    mode: env.mode || 'development',
    entry: ['./index.js'],
    resolve: {
      extensions: ['.ts', '.tsx', '.js', '.jsx']
    },
    plugins: [
      new ModuleFederationPlugin({
        name: 'HostApp',
        remotes: {
          // Remote feature apps
          HomeFeature: `HomeFeature@http://localhost:9001/[platformOS]/remoteEntry.js`,
          ProductsFeature: `ProductsFeature@http://localhost:9002/[platformOS]/remoteEntry.js`,
          CheckoutFeature: `CheckoutFeature@http://localhost:9003/[platformOS]/remoteEntry.js`
        },
        shared: {
          react: { singleton: true, eager: true },
          'react-native': { singleton: true, eager: true },
          '@react-navigation/native': { singleton: true }
        }
      })
    ]
  };
};
```

### 2.3 Remote App Configuration

```javascript
// webpack.config.js (Products Feature)
const { ModuleFederationPlugin } = require('@callstack/repack/mf');

module.exports = (env) => {
  return {
    mode: env.mode || 'development',
    entry: ['./index.js'],
    plugins: [
      new ModuleFederationPlugin({
        name: 'ProductsFeature',
        filename: 'remoteEntry.js',
        exposes: {
          // Expose screens and components
          './ProductsNavigator': './src/navigation/ProductsNavigator',
          './ProductCard': './src/components/ProductCard',
          './useProducts': './src/hooks/useProducts'
        },
        shared: {
          react: { singleton: true },
          'react-native': { singleton: true },
          '@react-navigation/native': { singleton: true }
        }
      })
    ]
  };
};
```

---

## 3. Feature Modules ตัวอย่าง

### 3.1 Products Feature Module

```typescript
// features/products/src/types.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  imageUrl: string;
  category: string;
  stock: number;
}

export interface ProductFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  search?: string;
}

export interface ProductsState {
  items: Product[];
  filters: ProductFilters;
  loading: boolean;
  error: string | null;
  selectedProduct: Product | null;
}
```

```typescript
// features/products/src/store/productsSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import type { Product, ProductFilters, ProductsState } from '../types';

export const fetchProducts = createAsyncThunk(
  'products/fetchAll',
  async (filters: ProductFilters = {}) => {
    const params = new URLSearchParams();
    if (filters.category) params.set('category', filters.category);
    if (filters.search) params.set('search', filters.search);
    if (filters.minPrice) params.set('minPrice', String(filters.minPrice));
    if (filters.maxPrice) params.set('maxPrice', String(filters.maxPrice));
    
    const response = await fetch(`/api/products?${params}`);
    return response.json() as Promise<Product[]>;
  }
);

const initialState: ProductsState = {
  items: [],
  filters: {},
  loading: false,
  error: null,
  selectedProduct: null
};

const productsSlice = createSlice({
  name: 'products',
  initialState,
  reducers: {
    setFilters(state, action: PayloadAction<ProductFilters>) {
      state.filters = { ...state.filters, ...action.payload };
    },
    selectProduct(state, action: PayloadAction<Product | null>) {
      state.selectedProduct = action.payload;
    },
    clearFilters(state) {
      state.filters = {};
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message ?? 'Failed to fetch products';
      });
  }
});

export const { setFilters, selectProduct, clearFilters } = productsSlice.actions;
export default productsSlice.reducer;
```

```tsx
// features/products/src/screens/ProductListScreen.tsx
import React, { useEffect, useCallback } from 'react';
import {
  View,
  FlatList,
  TextInput,
  StyleSheet,
  TouchableOpacity,
  Text,
  ActivityIndicator,
  RefreshControl
} from 'react-native';
import { useDispatch, useSelector } from 'react-redux';
import { fetchProducts, setFilters } from '../store/productsSlice';
import ProductCard from '../components/ProductCard';
import type { Product } from '../types';

interface Props {
  onProductPress: (product: Product) => void;
}

const ProductListScreen: React.FC<Props> = ({ onProductPress }) => {
  const dispatch = useDispatch<any>();
  const { items, loading, filters } = useSelector((state: any) => state.products);
  
  useEffect(() => {
    dispatch(fetchProducts(filters));
  }, [dispatch, filters]);
  
  const handleSearch = useCallback((text: string) => {
    dispatch(setFilters({ search: text }));
  }, [dispatch]);
  
  const handleRefresh = useCallback(() => {
    dispatch(fetchProducts(filters));
  }, [dispatch, filters]);
  
  return (
    <View style={styles.container}>
      <TextInput
        style={styles.searchInput}
        placeholder="ค้นหาสินค้า..."
        onChangeText={handleSearch}
        value={filters.search || ''}
      />
      
      <FlatList
        data={items}
        keyExtractor={item => item.id}
        numColumns={2}
        renderItem={({ item }) => (
          <ProductCard
            product={item}
            onPress={() => onProductPress(item)}
          />
        )}
        refreshControl={
          <RefreshControl refreshing={loading} onRefresh={handleRefresh} />
        }
        ListEmptyComponent={
          loading ? (
            <ActivityIndicator style={styles.loader} />
          ) : (
            <Text style={styles.emptyText}>ไม่พบสินค้า</Text>
          )
        }
        contentContainerStyle={styles.list}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  searchInput: {
    margin: 12,
    padding: 10,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    fontSize: 16
  },
  list: { paddingHorizontal: 8 },
  loader: { marginTop: 50 },
  emptyText: {
    textAlign: 'center',
    marginTop: 50,
    fontSize: 16,
    color: '#666'
  }
});

export default ProductListScreen;
```

---

## 4. Shared State Management

### 4.1 Root Store Configuration

```typescript
// app/store/rootStore.ts
import { configureStore, combineReducers } from '@reduxjs/toolkit';

// Feature reducers
import { homeReducer } from '@myapp/feature-home';
import { productsReducer } from '@myapp/feature-products';
import { cartReducer } from '@myapp/feature-cart';
import { userReducer } from '@myapp/feature-user';

// Shared reducers
import { authReducer } from '../auth/authSlice';
import { notificationsReducer } from '../notifications/notificationsSlice';

const rootReducer = combineReducers({
  auth: authReducer,
  notifications: notificationsReducer,
  home: homeReducer,
  products: productsReducer,
  cart: cartReducer,
  user: userReducer
});

export const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        // Ignore these action types
        ignoredActions: ['auth/loginSuccess']
      }
    })
});

export type RootState = ReturnType<typeof rootReducer>;
export type AppDispatch = typeof store.dispatch;
```

### 4.2 Cross-Feature Communication

```typescript
// shared/events/EventBus.ts
type EventHandler<T = any> = (data: T) => void;

class EventBus {
  private handlers: Map<string, EventHandler[]> = new Map();
  
  on<T>(event: string, handler: EventHandler<T>): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    
    this.handlers.get(event)!.push(handler as EventHandler);
    
    // Return unsubscribe function
    return () => this.off(event, handler);
  }
  
  off<T>(event: string, handler: EventHandler<T>): void {
    const handlers = this.handlers.get(event);
    if (handlers) {
      const index = handlers.indexOf(handler as EventHandler);
      if (index > -1) {
        handlers.splice(index, 1);
      }
    }
  }
  
  emit<T>(event: string, data: T): void {
    const handlers = this.handlers.get(event);
    if (handlers) {
      handlers.forEach(handler => {
        try {
          handler(data);
        } catch (error) {
          console.error(`Error in event handler for "${event}":`, error);
        }
      });
    }
  }
  
  once<T>(event: string, handler: EventHandler<T>): void {
    const unsubscribe = this.on<T>(event, (data) => {
      handler(data);
      unsubscribe();
    });
  }
}

export const eventBus = new EventBus();

// Event types
export const EVENTS = {
  USER_LOGGED_IN: 'user:logged_in',
  USER_LOGGED_OUT: 'user:logged_out',
  CART_UPDATED: 'cart:updated',
  ORDER_PLACED: 'order:placed',
  PRODUCT_VIEWED: 'product:viewed',
  NOTIFICATION_RECEIVED: 'notification:received'
} as const;

export type EventType = typeof EVENTS[keyof typeof EVENTS];
```

---

## 5. Workshop: Micro Frontend Setup

### Step 1: Setup Monorepo

```bash
# สร้าง monorepo ด้วย Yarn Workspaces
mkdir my-micro-frontend-app
cd my-micro-frontend-app

# สร้าง root package.json
cat > package.json << EOF
{
  "name": "my-micro-frontend-app",
  "private": true,
  "workspaces": [
    "app",
    "features/*",
    "shared/*"
  ]
}
EOF

# สร้าง structure
mkdir -p app features/home features/products features/cart shared/ui shared/utils
```

### Step 2: Shared UI Components

```tsx
// shared/ui/src/components/Button.tsx
import React from 'react';
import {
  TouchableOpacity,
  Text,
  StyleSheet,
  ActivityIndicator,
  TouchableOpacityProps
} from 'react-native';

interface ButtonProps extends TouchableOpacityProps {
  title: string;
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'small' | 'medium' | 'large';
  loading?: boolean;
  fullWidth?: boolean;
}

export const Button: React.FC<ButtonProps> = ({
  title,
  variant = 'primary',
  size = 'medium',
  loading = false,
  fullWidth = false,
  disabled,
  style,
  ...props
}) => {
  const isDisabled = disabled || loading;
  
  return (
    <TouchableOpacity
      style={[
        styles.base,
        styles[variant],
        styles[size],
        fullWidth && styles.fullWidth,
        isDisabled && styles.disabled,
        style
      ]}
      disabled={isDisabled}
      {...props}
    >
      {loading ? (
        <ActivityIndicator color={variant === 'primary' ? '#fff' : '#007AFF'} />
      ) : (
        <Text style={[styles.text, styles[`${variant}Text`], styles[`${size}Text`]]}>
          {title}
        </Text>
      )}
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  base: {
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
    flexDirection: 'row'
  },
  primary: { backgroundColor: '#007AFF' },
  secondary: {
    backgroundColor: 'transparent',
    borderWidth: 1,
    borderColor: '#007AFF'
  },
  danger: { backgroundColor: '#FF3B30' },
  ghost: { backgroundColor: 'transparent' },
  
  small: { paddingHorizontal: 12, paddingVertical: 6 },
  medium: { paddingHorizontal: 20, paddingVertical: 12 },
  large: { paddingHorizontal: 32, paddingVertical: 16 },
  
  fullWidth: { width: '100%' },
  disabled: { opacity: 0.5 },
  
  text: { fontWeight: '600' },
  primaryText: { color: '#fff' },
  secondaryText: { color: '#007AFF' },
  dangerText: { color: '#fff' },
  ghostText: { color: '#007AFF' },
  
  smallText: { fontSize: 14 },
  mediumText: { fontSize: 16 },
  largeText: { fontSize: 18 }
});
```

### Step 3: Feature Registration

```typescript
// app/features/FeatureRegistry.ts
interface FeatureConfig {
  name: string;
  navigator: React.ComponentType<any>;
  reducer?: any;
  initialRoute?: string;
}

class FeatureRegistry {
  private features: Map<string, FeatureConfig> = new Map();
  
  register(config: FeatureConfig): void {
    if (this.features.has(config.name)) {
      console.warn(`Feature "${config.name}" is already registered`);
      return;
    }
    
    this.features.set(config.name, config);
    console.log(`Feature "${config.name}" registered`);
  }
  
  getAll(): FeatureConfig[] {
    return Array.from(this.features.values());
  }
  
  get(name: string): FeatureConfig | undefined {
    return this.features.get(name);
  }
  
  getReducers(): Record<string, any> {
    const reducers: Record<string, any> = {};
    this.features.forEach((config, name) => {
      if (config.reducer) {
        reducers[name] = config.reducer;
      }
    });
    return reducers;
  }
}

export const featureRegistry = new FeatureRegistry();
```

```typescript
// app/App.tsx
import React from 'react';
import { Provider } from 'react-redux';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';
import { featureRegistry } from './features/FeatureRegistry';

// Register features
import { HomeFeature } from '@myapp/feature-home';
import { ProductsFeature } from '@myapp/feature-products';
import { CartFeature } from '@myapp/feature-cart';

featureRegistry.register(HomeFeature);
featureRegistry.register(ProductsFeature);
featureRegistry.register(CartFeature);

// Create store with all feature reducers
import { configureStore } from '@reduxjs/toolkit';
const store = configureStore({
  reducer: featureRegistry.getReducers()
});

const Stack = createStackNavigator();

const App: React.FC = () => {
  const features = featureRegistry.getAll();
  
  return (
    <Provider store={store}>
      <NavigationContainer>
        <Stack.Navigator>
          {features.map(feature => (
            <Stack.Screen
              key={feature.name}
              name={feature.name}
              component={feature.navigator}
            />
          ))}
        </Stack.Navigator>
      </NavigationContainer>
    </Provider>
  );
};

export default App;
```

---

## Tips สำหรับ Micro Frontends

1. **Define clear boundaries** - แต่ละ feature ควรมี boundary ที่ชัดเจน
2. **Share carefully** - แชร์เฉพาะสิ่งที่จำเป็น avoid tight coupling
3. **Versioning** - version แต่ละ feature module
4. **Testing** - test แต่ละ feature module independently
5. **Communication through events** - ใช้ event bus แทน direct imports

## สรุป

Micro Frontends ใน React Native ช่วย:
1. Scale development team
2. Independent deployment
3. Technology flexibility
4. Fault isolation
5. Better code organization

เหมาะสำหรับ large applications ที่มีหลาย team ทำงานร่วมกัน
