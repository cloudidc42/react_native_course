# Part 095: Monorepo Architecture ใน React Native

## บทนำ

Monorepo เป็นแนวทางที่เก็บ codebase ทั้งหมดไว้ในที่เดียว ช่วยให้แชร์โค้ดระหว่าง projects ได้ง่าย เราจะเรียนรู้ Turborepo, Nx, Shared packages, และ React Native + Next.js monorepo

## หัวข้อที่จะเรียน

1. Turborepo Setup
2. Nx Workspace
3. Shared Packages
4. React Native + Next.js Monorepo
5. Workshop: Monorepo Setup

---

## 1. Turborepo Setup

### สร้าง Turborepo Monorepo

```bash
npx create-turbo@latest my-monorepo
cd my-monorepo
```

### โครงสร้างโปรเจค

```
my-monorepo/
├── apps/
│   ├── mobile/          # React Native app
│   ├── web/             # Next.js app
│   └── admin/           # Admin dashboard
├── packages/
│   ├── ui/              # Shared UI components
│   ├── config/          # Shared configurations
│   ├── utils/           # Shared utilities
│   ├── api-client/      # API client
│   └── types/           # Shared TypeScript types
├── turbo.json
├── package.json
└── pnpm-workspace.yaml
```

### package.json (root)

```json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "test": "turbo test",
    "lint": "turbo lint",
    "type-check": "turbo type-check",
    "mobile": "turbo dev --filter=mobile",
    "web": "turbo dev --filter=web"
  },
  "devDependencies": {
    "turbo": "^1.10.0",
    "typescript": "^5.0.0"
  },
  "packageManager": "pnpm@8.0.0"
}
```

### pnpm-workspace.yaml

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [
        ".next/**",
        "!.next/cache/**",
        "android/app/build/**",
        "ios/build/**",
        "lib/**"
      ]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "type-check": {
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
```

---

## 2. Shared Packages

### packages/types/

```typescript
// packages/types/src/index.ts

// User types
export interface User {
  id: string;
  email: string;
  displayName: string;
  photoURL?: string;
  role: UserRole;
  createdAt: Date;
  updatedAt: Date;
}

export type UserRole = 'admin' | 'user' | 'moderator';

// Product types
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  currency: string;
  images: string[];
  category: Category;
  stock: number;
  isActive: boolean;
}

export interface Category {
  id: string;
  name: string;
  slug: string;
  parentId?: string;
}

// Order types
export interface Order {
  id: string;
  userId: string;
  items: OrderItem[];
  status: OrderStatus;
  total: number;
  shippingAddress: Address;
  paymentMethod: string;
  createdAt: Date;
}

export interface OrderItem {
  productId: string;
  name: string;
  quantity: number;
  price: number;
}

export type OrderStatus = 
  | 'pending'
  | 'confirmed'
  | 'processing'
  | 'shipped'
  | 'delivered'
  | 'cancelled'
  | 'refunded';

export interface Address {
  street: string;
  city: string;
  province: string;
  postalCode: string;
  country: string;
  phone?: string;
}

// API Response types
export interface ApiResponse<T> {
  data: T;
  message?: string;
  success: boolean;
}

export interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination: {
    total: number;
    page: number;
    perPage: number;
    totalPages: number;
  };
}

export interface ApiError {
  message: string;
  code: string;
  statusCode: number;
  details?: Record<string, string[]>;
}
```

### packages/types/package.json

```json
{
  "name": "@myapp/types",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  }
}
```

### packages/utils/

```typescript
// packages/utils/src/format.ts

export const formatCurrency = (
  amount: number,
  currency: string = 'THB',
  locale: string = 'th-TH'
): string => {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
    minimumFractionDigits: 0,
    maximumFractionDigits: 2,
  }).format(amount);
};

export const formatDate = (
  date: Date | string,
  format: 'short' | 'long' | 'relative' = 'short',
  locale: string = 'th-TH'
): string => {
  const d = new Date(date);
  
  if (format === 'relative') {
    const now = new Date();
    const diffMs = now.getTime() - d.getTime();
    const diffMins = Math.floor(diffMs / 60000);
    const diffHours = Math.floor(diffMins / 60);
    const diffDays = Math.floor(diffHours / 24);
    
    if (diffMins < 1) return 'เมื่อกี้';
    if (diffMins < 60) return `${diffMins} นาทีที่แล้ว`;
    if (diffHours < 24) return `${diffHours} ชั่วโมงที่แล้ว`;
    if (diffDays < 7) return `${diffDays} วันที่แล้ว`;
  }
  
  return d.toLocaleDateString(locale, 
    format === 'long' 
      ? { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' }
      : { year: 'numeric', month: 'short', day: 'numeric' }
  );
};

export const truncateText = (text: string, maxLength: number): string => {
  if (text.length <= maxLength) return text;
  return text.substring(0, maxLength - 3) + '...';
};

export const slugify = (text: string): string => {
  return text
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/[\s_-]+/g, '-')
    .replace(/^-+|-+$/g, '');
};

// packages/utils/src/validation.ts
export const validators = {
  email: (value: string): boolean => {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  },
  
  phone: (value: string): boolean => {
    // Thai phone number
    return /^(0[6-9]\d{8}|0[2-8]\d{7})$/.test(value.replace(/\s/g, ''));
  },
  
  thaiId: (value: string): boolean => {
    if (value.length !== 13) return false;
    let sum = 0;
    for (let i = 0; i < 12; i++) {
      sum += parseInt(value[i]) * (13 - i);
    }
    const lastDigit = (11 - (sum % 11)) % 10;
    return lastDigit === parseInt(value[12]);
  },
  
  password: (value: string): {
    isValid: boolean;
    errors: string[];
  } => {
    const errors: string[] = [];
    if (value.length < 8) errors.push('ต้องมีอย่างน้อย 8 ตัวอักษร');
    if (!/[A-Z]/.test(value)) errors.push('ต้องมีตัวพิมพ์ใหญ่');
    if (!/[a-z]/.test(value)) errors.push('ต้องมีตัวพิมพ์เล็ก');
    if (!/[0-9]/.test(value)) errors.push('ต้องมีตัวเลข');
    if (!/[!@#$%^&*]/.test(value)) errors.push('ต้องมีอักขระพิเศษ');
    return { isValid: errors.length === 0, errors };
  },
};

// packages/utils/src/index.ts
export * from './format';
export * from './validation';
export * from './array';
export * from './object';
```

### packages/api-client/

```typescript
// packages/api-client/src/client.ts
import type { ApiResponse, PaginatedResponse, ApiError } from '@myapp/types';

export interface RequestConfig {
  baseURL: string;
  timeout?: number;
  headers?: Record<string, string>;
  getAuthToken?: () => Promise<string | null>;
  onAuthError?: () => void;
}

export class APIClient {
  private config: Required<RequestConfig>;

  constructor(config: RequestConfig) {
    this.config = {
      timeout: 30000,
      headers: {},
      getAuthToken: async () => null,
      onAuthError: () => {},
      ...config,
    };
  }

  private async getHeaders(): Promise<Record<string, string>> {
    const token = await this.config.getAuthToken();
    return {
      'Content-Type': 'application/json',
      ...this.config.headers,
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    };
  }

  async request<T>(
    method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE',
    path: string,
    options: {
      body?: unknown;
      query?: Record<string, string | number | boolean>;
    } = {}
  ): Promise<T> {
    const headers = await this.getHeaders();
    const url = new URL(`${this.config.baseURL}${path}`);
    
    if (options.query) {
      Object.entries(options.query).forEach(([key, value]) => {
        url.searchParams.set(key, String(value));
      });
    }

    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), this.config.timeout);

    try {
      const response = await fetch(url.toString(), {
        method,
        headers,
        body: options.body ? JSON.stringify(options.body) : undefined,
        signal: controller.signal,
      });

      if (response.status === 401) {
        this.config.onAuthError();
        throw new Error('Unauthorized');
      }

      if (!response.ok) {
        const error: ApiError = await response.json();
        throw error;
      }

      return response.json();
    } finally {
      clearTimeout(timeout);
    }
  }

  get<T>(path: string, query?: Record<string, any>) {
    return this.request<T>('GET', path, { query });
  }

  post<T>(path: string, body: unknown) {
    return this.request<T>('POST', path, { body });
  }

  put<T>(path: string, body: unknown) {
    return this.request<T>('PUT', path, { body });
  }

  patch<T>(path: string, body: unknown) {
    return this.request<T>('PATCH', path, { body });
  }

  delete<T>(path: string) {
    return this.request<T>('DELETE', path);
  }
}

// Specific API services
// packages/api-client/src/services/products.ts
import type { Product, PaginatedResponse } from '@myapp/types';
import { APIClient } from '../client';

export const createProductsService = (client: APIClient) => ({
  list: (params?: {
    page?: number;
    limit?: number;
    category?: string;
    search?: string;
  }) => client.get<PaginatedResponse<Product>>('/products', params),
  
  get: (id: string) =>
    client.get<{ data: Product }>(`/products/${id}`),
  
  create: (data: Partial<Product>) =>
    client.post<{ data: Product }>('/products', data),
  
  update: (id: string, data: Partial<Product>) =>
    client.patch<{ data: Product }>(`/products/${id}`, data),
  
  delete: (id: string) =>
    client.delete(`/products/${id}`),
});
```

---

## 3. React Native App Setup

### apps/mobile/package.json

```json
{
  "name": "mobile",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "start": "react-native start",
    "dev": "react-native start",
    "build": "echo 'build handled by EAS'"
  },
  "dependencies": {
    "@myapp/types": "*",
    "@myapp/utils": "*",
    "@myapp/api-client": "*",
    "@myapp/ui": "*",
    "react": "18.2.0",
    "react-native": "0.72.0"
  }
}
```

### apps/mobile/babel.config.js

```javascript
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: [
    [
      'module-resolver',
      {
        root: ['./src'],
        extensions: ['.ios.js', '.android.js', '.js', '.ts', '.tsx', '.json'],
        alias: {
          '@': './src',
          '@myapp/ui': '../../packages/ui/src',
          '@myapp/utils': '../../packages/utils/src',
        },
      },
    ],
  ],
};
```

---

## 4. Next.js App Setup

### apps/web/package.json

```json
{
  "name": "web",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "@myapp/types": "*",
    "@myapp/utils": "*",
    "@myapp/api-client": "*",
    "next": "14.0.0",
    "react": "18.2.0",
    "react-dom": "18.2.0"
  }
}
```

---

## 5. Shared UI Package

### packages/ui/

```typescript
// packages/ui/src/index.ts
// Re-export ทุก components
export { Button } from './components/Button';
export { Input } from './components/Input';
export { Card } from './components/Card';
export { Badge } from './components/Badge';

// packages/ui/package.json
{
  "name": "@myapp/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./native": "./src/native/index.ts",
    "./web": "./src/web/index.ts"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-native": ">=0.71.0"
  }
}
```

---

## Workshop: Complete Monorepo Setup

### scripts/setup.sh

```bash
#!/bin/bash
echo "🚀 Setting up Monorepo..."

# Install dependencies
pnpm install

# Build all packages
pnpm turbo build --filter=./packages/*

# Setup mobile
cd apps/mobile
npx react-native setup-ios-permissions

echo "✅ Monorepo setup complete!"
```

### CI/CD with Turborepo

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Type check
        run: pnpm turbo type-check
      
      - name: Lint
        run: pnpm turbo lint
      
      - name: Test
        run: pnpm turbo test
      
      - name: Build
        run: pnpm turbo build
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ secrets.TURBO_TEAM }}
```

---

## Tips

### 1. Remote Caching

```bash
# ใช้ Vercel Remote Cache เพื่อแชร์ cache ระหว่าง CI runs
npx turbo link
```

### 2. Affected Changes Only

```bash
# Build เฉพาะ packages ที่เปลี่ยนแปลง
pnpm turbo build --filter='...[HEAD^1]'
```

### 3. Dependency Graph

```bash
# ดู dependency graph
pnpm turbo graph
```

---

## Workshop Exercises

1. **Add new package** - สร้าง `packages/hooks` สำหรับ shared hooks
2. **Version bumping** - setup Changesets สำหรับ version management
3. **Docker** - สร้าง Docker setup สำหรับทุก apps
4. **Deploy** - setup deployment pipeline สำหรับทุก apps

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Turborepo** - task runner สำหรับ monorepo
2. **Shared Packages** - types, utils, api-client, ui
3. **React Native** - setup ใน monorepo
4. **Next.js** - web app ใน monorepo
5. **CI/CD** - pipeline สำหรับ monorepo
