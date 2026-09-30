# Part 035: Axios และ HTTP Client

## Axios vs Fetch

### ทำไมถึงใช้ Axios?

| Feature | Fetch | Axios |
|---------|-------|-------|
| Browser support | Modern | All |
| Timeout | Manual | Built-in |
| Interceptors | ❌ | ✅ |
| Request cancellation | Manual | Built-in |
| Request/Response transform | ❌ | ✅ |
| Error handling | Manual (ต้อง check res.ok) | Automatic (throw on 4xx/5xx) |
| Upload progress | Manual | Built-in |
| CSRF protection | ❌ | ✅ |
| Base URL | ❌ | ✅ |

### ติดตั้ง Axios

```bash
npm install axios

# TypeScript types (รวมมาอยู่แล้วใน axios ≥ 1.0)
```

---

## พื้นฐาน Axios

### HTTP Methods

```typescript
import axios from 'axios';

// GET
const response = await axios.get('/api/posts');

// POST
const newPost = await axios.post('/api/posts', {
  title: 'Hello',
  body: 'World',
});

// PUT
const updated = await axios.put('/api/posts/1', {
  title: 'Updated',
});

// PATCH
const patched = await axios.patch('/api/posts/1', {
  title: 'Patched',
});

// DELETE
await axios.delete('/api/posts/1');
```

### Config Object

```typescript
const response = await axios({
  method: 'post',
  url: '/api/posts',
  data: { title: 'Hello' },
  headers: { 'Content-Type': 'application/json' },
  timeout: 5000,
  params: { userId: 1 },
});
```

---

## สร้าง Axios Instance

```typescript
// src/services/axiosInstance.ts
import axios, { AxiosInstance, InternalAxiosRequestConfig, AxiosResponse } from 'axios';

const BASE_URL = 'https://jsonplaceholder.typicode.com';

export const apiClient: AxiosInstance = axios.create({
  baseURL: BASE_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
});
```

---

## Interceptors

Interceptors ช่วยให้เราสกัดกั้น requests/responses ก่อนที่จะถึง handler

### Request Interceptor - เพิ่ม Auth Token

```typescript
// src/services/axiosInstance.ts
import axios from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';

export const apiClient = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 10000,
});

// Request Interceptor
apiClient.interceptors.request.use(
  async (config) => {
    // เพิ่ม token ทุก request
    const token = await AsyncStorage.getItem('access_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    // Log requests ใน dev mode
    if (__DEV__) {
      console.log(`[API Request] ${config.method?.toUpperCase()} ${config.url}`, {
        params: config.params,
        data: config.data,
      });
    }

    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Response Interceptor
apiClient.interceptors.response.use(
  (response) => {
    // Log responses ใน dev mode
    if (__DEV__) {
      console.log(`[API Response] ${response.status}`, response.data);
    }
    return response;
  },
  async (error) => {
    const originalRequest = error.config;

    // Token refresh logic
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = await AsyncStorage.getItem('refresh_token');
        const response = await axios.post('/auth/refresh', { refreshToken });
        const { accessToken } = response.data;

        await AsyncStorage.setItem('access_token', accessToken);
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;

        return apiClient(originalRequest);
      } catch (refreshError) {
        // Logout user
        await AsyncStorage.multiRemove(['access_token', 'refresh_token']);
        // Navigate to login
        // navigationRef.navigate('Login');
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

---

## Request/Response Transformation

```typescript
// src/services/axiosInstance.ts
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: 'https://api.example.com',

  // Transform request data ก่อนส่ง
  transformRequest: [
    (data) => {
      if (data instanceof FormData) return data;
      // Convert camelCase to snake_case
      return JSON.stringify(camelToSnake(data));
    },
  ],

  // Transform response data หลังรับ
  transformResponse: [
    (data) => {
      try {
        const parsed = JSON.parse(data);
        // Convert snake_case to camelCase
        return snakeToCamel(parsed);
      } catch {
        return data;
      }
    },
  ],
});

// Helper: camelCase → snake_case
function camelToSnake(obj: any): any {
  if (Array.isArray(obj)) {
    return obj.map(camelToSnake);
  }
  if (obj !== null && typeof obj === 'object') {
    return Object.entries(obj).reduce((acc, [key, val]) => {
      const snakeKey = key.replace(
        /[A-Z]/g,
        (letter) => `_${letter.toLowerCase()}`
      );
      acc[snakeKey] = camelToSnake(val);
      return acc;
    }, {} as any);
  }
  return obj;
}

// Helper: snake_case → camelCase
function snakeToCamel(obj: any): any {
  if (Array.isArray(obj)) {
    return obj.map(snakeToCamel);
  }
  if (obj !== null && typeof obj === 'object') {
    return Object.entries(obj).reduce((acc, [key, val]) => {
      const camelKey = key.replace(/_([a-z])/g, (_, letter) =>
        letter.toUpperCase()
      );
      acc[camelKey] = snakeToCamel(val);
      return acc;
    }, {} as any);
  }
  return obj;
}
```

---

## Error Handling

### Axios Error Types

```typescript
import axios, { AxiosError } from 'axios';

interface ApiError {
  message: string;
  code: string;
  status: number;
}

export class ApiException extends Error {
  status: number;
  code: string;
  data: any;

  constructor(message: string, status: number, code: string, data?: any) {
    super(message);
    this.name = 'ApiException';
    this.status = status;
    this.code = code;
    this.data = data;
  }
}

export function handleAxiosError(error: unknown): never {
  if (axios.isAxiosError(error)) {
    const axiosError = error as AxiosError<ApiError>;

    if (axiosError.response) {
      // Server responded with error
      const { status, data } = axiosError.response;
      const message = data?.message || getDefaultMessage(status);
      const code = data?.code || String(status);

      throw new ApiException(message, status, code, data);
    } else if (axiosError.request) {
      // Request made but no response (network error)
      throw new ApiException(
        'ไม่สามารถเชื่อมต่อ server ได้ กรุณาตรวจสอบ internet',
        0,
        'NETWORK_ERROR'
      );
    } else if (axiosError.code === 'ECONNABORTED') {
      // Timeout
      throw new ApiException(
        'การเชื่อมต่อหมดเวลา กรุณาลองใหม่',
        0,
        'TIMEOUT'
      );
    }
  }

  // Unknown error
  throw new ApiException(
    'เกิดข้อผิดพลาดที่ไม่คาดคิด',
    0,
    'UNKNOWN_ERROR'
  );
}

function getDefaultMessage(status: number): string {
  const messages: { [key: number]: string } = {
    400: 'ข้อมูลไม่ถูกต้อง',
    401: 'กรุณาเข้าสู่ระบบ',
    403: 'ไม่มีสิทธิ์เข้าถึง',
    404: 'ไม่พบข้อมูล',
    409: 'ข้อมูลซ้ำ',
    422: 'ข้อมูลไม่ผ่านการตรวจสอบ',
    429: 'มีการส่งคำขอมากเกินไป กรุณารอสักครู่',
    500: 'เกิดข้อผิดพลาดบน server',
    502: 'Server ไม่พร้อมให้บริการ',
    503: 'Service ไม่พร้อมให้บริการ',
  };
  return messages[status] || 'เกิดข้อผิดพลาด';
}
```

---

## Retry Logic

```typescript
// src/services/retryAxios.ts
import axios, { AxiosInstance, AxiosError } from 'axios';
import axiosRetry from 'axios-retry';

// ติดตั้ง axios-retry
// npm install axios-retry

export function createRetryClient(): AxiosInstance {
  const client = axios.create({
    baseURL: 'https://api.example.com',
    timeout: 10000,
  });

  axiosRetry(client, {
    retries: 3,
    retryDelay: (retryCount) => {
      // Exponential backoff: 1s, 2s, 4s
      return Math.pow(2, retryCount - 1) * 1000;
    },
    retryCondition: (error: AxiosError) => {
      // Retry ถ้าเป็น network error หรือ 5xx
      return (
        axiosRetry.isNetworkOrIdempotentRequestError(error) ||
        (error.response?.status !== undefined && error.response.status >= 500)
      );
    },
    onRetry: (retryCount, error) => {
      console.log(`Retry attempt ${retryCount}: ${error.message}`);
    },
  });

  return client;
}
```

### Manual Retry Logic

```typescript
async function fetchWithRetry<T>(
  fetchFn: () => Promise<T>,
  maxRetries: number = 3,
  delay: number = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fetchFn();
    } catch (error) {
      const isLastAttempt = attempt === maxRetries;

      if (isLastAttempt) {
        throw error;
      }

      // Check ว่าควร retry ไหม
      if (axios.isAxiosError(error)) {
        const status = error.response?.status;
        // ไม่ retry 4xx errors (client errors)
        if (status && status >= 400 && status < 500) {
          throw error;
        }
      }

      // Wait ก่อน retry
      const waitTime = delay * Math.pow(2, attempt - 1);
      console.log(`Attempt ${attempt} failed. Retrying in ${waitTime}ms...`);
      await new Promise((resolve) => setTimeout(resolve, waitTime));
    }
  }

  throw new Error('Max retries exceeded');
}

// ใช้งาน
const posts = await fetchWithRetry(() => apiClient.get('/posts'));
```

---

## Workshop: API Service Layer

### สร้าง Full API Service

```typescript
// src/services/api/index.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import AsyncStorage from '@react-native-async-storage/async-storage';
import NetInfo from '@react-native-community/netinfo';

class ApiService {
  private client: AxiosInstance;
  private isRefreshing = false;
  private failedQueue: Array<{
    resolve: (value: any) => void;
    reject: (error: any) => void;
  }> = [];

  constructor() {
    this.client = axios.create({
      baseURL: 'https://api.example.com/v1',
      timeout: 15000,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    this.setupInterceptors();
  }

  private setupInterceptors() {
    // Request
    this.client.interceptors.request.use(
      async (config) => {
        // ตรวจสอบ network
        const networkState = await NetInfo.fetch();
        if (!networkState.isConnected) {
          throw new Error('NO_INTERNET');
        }

        // เพิ่ม token
        const token = await AsyncStorage.getItem('access_token');
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }

        // เพิ่ม timestamp
        config.headers['X-Request-Time'] = Date.now().toString();

        return config;
      },
      (error) => Promise.reject(error)
    );

    // Response
    this.client.interceptors.response.use(
      (response) => response.data,
      async (error) => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
          if (this.isRefreshing) {
            return new Promise((resolve, reject) => {
              this.failedQueue.push({ resolve, reject });
            })
              .then((token) => {
                originalRequest.headers.Authorization = `Bearer ${token}`;
                return this.client(originalRequest);
              })
              .catch((err) => Promise.reject(err));
          }

          originalRequest._retry = true;
          this.isRefreshing = true;

          try {
            const newToken = await this.refreshToken();
            this.processQueue(null, newToken);
            originalRequest.headers.Authorization = `Bearer ${newToken}`;
            return this.client(originalRequest);
          } catch (refreshError) {
            this.processQueue(refreshError, null);
            await this.logout();
            return Promise.reject(refreshError);
          } finally {
            this.isRefreshing = false;
          }
        }

        return Promise.reject(this.formatError(error));
      }
    );
  }

  private processQueue(error: any, token: string | null) {
    this.failedQueue.forEach(({ resolve, reject }) => {
      if (error) {
        reject(error);
      } else {
        resolve(token);
      }
    });
    this.failedQueue = [];
  }

  private async refreshToken(): Promise<string> {
    const refreshToken = await AsyncStorage.getItem('refresh_token');
    if (!refreshToken) throw new Error('No refresh token');

    const response = await axios.post('/auth/refresh', { refreshToken });
    const { accessToken } = response.data;

    await AsyncStorage.setItem('access_token', accessToken);
    return accessToken;
  }

  private async logout() {
    await AsyncStorage.multiRemove(['access_token', 'refresh_token']);
  }

  private formatError(error: any) {
    if (axios.isAxiosError(error)) {
      if (!error.response) {
        return new Error('NETWORK_ERROR');
      }
      const { status, data } = error.response;
      return {
        status,
        message: data?.message || 'เกิดข้อผิดพลาด',
        code: data?.code || String(status),
        data,
      };
    }
    return error;
  }

  // Generic HTTP methods
  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    return this.client.get(url, config);
  }

  async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    return this.client.post(url, data, config);
  }

  async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    return this.client.put(url, data, config);
  }

  async patch<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    return this.client.patch(url, data, config);
  }

  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    return this.client.delete(url, config);
  }

  // Upload file
  async uploadFile(
    url: string,
    file: {
      uri: string;
      name: string;
      type: string;
    },
    onProgress?: (progress: number) => void
  ): Promise<any> {
    const formData = new FormData();
    formData.append('file', file as any);

    return this.client.post(url, formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
      onUploadProgress: (progressEvent) => {
        if (progressEvent.total && onProgress) {
          const progress = Math.round(
            (progressEvent.loaded / progressEvent.total) * 100
          );
          onProgress(progress);
        }
      },
    });
  }
}

export const api = new ApiService();
```

### Domain-specific API modules

```typescript
// src/services/api/authApi.ts
import { api } from './index';

interface LoginRequest {
  email: string;
  password: string;
}

interface AuthResponse {
  user: User;
  accessToken: string;
  refreshToken: string;
}

export const authApi = {
  login: (data: LoginRequest) =>
    api.post<AuthResponse>('/auth/login', data),

  register: (data: {
    name: string;
    email: string;
    password: string;
  }) => api.post<AuthResponse>('/auth/register', data),

  logout: () => api.post('/auth/logout'),

  forgotPassword: (email: string) =>
    api.post('/auth/forgot-password', { email }),

  resetPassword: (token: string, password: string) =>
    api.post('/auth/reset-password', { token, password }),

  verifyEmail: (token: string) =>
    api.post('/auth/verify-email', { token }),
};

// src/services/api/userApi.ts
export const userApi = {
  getProfile: () => api.get<User>('/users/me'),

  updateProfile: (data: Partial<User>) =>
    api.patch<User>('/users/me', data),

  uploadAvatar: (
    file: { uri: string; name: string; type: string },
    onProgress?: (p: number) => void
  ) => api.uploadFile('/users/me/avatar', file, onProgress),

  deleteAccount: () => api.delete('/users/me'),
};

// src/services/api/postsApi.ts
export const postsApi = {
  getAll: (params?: { page?: number; limit?: number; category?: string }) =>
    api.get<{ posts: Post[]; total: number; page: number }>('/posts', {
      params,
    }),

  getById: (id: number) => api.get<Post>(`/posts/${id}`),

  create: (data: { title: string; body: string; categoryId: number }) =>
    api.post<Post>('/posts', data),

  update: (id: number, data: Partial<Post>) =>
    api.patch<Post>(`/posts/${id}`, data),

  delete: (id: number) => api.delete(`/posts/${id}`),

  like: (id: number) => api.post(`/posts/${id}/like`),
  unlike: (id: number) => api.delete(`/posts/${id}/like`),
};
```

### ใช้งานใน React Component

```typescript
// src/screens/ProfileScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  Alert,
  ActivityIndicator,
} from 'react-native';
import * as ImagePicker from 'expo-image-picker';
import { userApi } from '../services/api/userApi';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export const ProfileScreen: React.FC = () => {
  const queryClient = useQueryClient();
  const [uploadProgress, setUploadProgress] = useState(0);

  const { data: user, isLoading } = useQuery({
    queryKey: ['user', 'me'],
    queryFn: userApi.getProfile,
  });

  const uploadMutation = useMutation({
    mutationFn: (file: { uri: string; name: string; type: string }) =>
      userApi.uploadAvatar(file, setUploadProgress),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['user', 'me'] });
      setUploadProgress(0);
      Alert.alert('สำเร็จ', 'อัปเดตรูปโปรไฟล์แล้ว');
    },
    onError: (error: any) => {
      setUploadProgress(0);
      Alert.alert('ผิดพลาด', error.message);
    },
  });

  const pickImage = async () => {
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: true,
      aspect: [1, 1],
      quality: 0.8,
    });

    if (!result.canceled) {
      const asset = result.assets[0];
      const fileName = asset.uri.split('/').pop() || 'avatar.jpg';
      const fileType = asset.mimeType || 'image/jpeg';

      uploadMutation.mutate({
        uri: asset.uri,
        name: fileName,
        type: fileType,
      });
    }
  };

  if (isLoading) return <ActivityIndicator />;

  return (
    <View style={styles.container}>
      <TouchableOpacity onPress={pickImage} style={styles.avatarContainer}>
        <Image
          source={{ uri: user?.avatar || 'https://via.placeholder.com/100' }}
          style={styles.avatar}
        />
        {uploadMutation.isPending ? (
          <View style={styles.uploadOverlay}>
            <Text style={styles.progressText}>{uploadProgress}%</Text>
          </View>
        ) : (
          <View style={styles.editBadge}>
            <Text style={styles.editBadgeText}>✏️</Text>
          </View>
        )}
      </TouchableOpacity>

      <Text style={styles.name}>{user?.name}</Text>
      <Text style={styles.email}>{user?.email}</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', padding: 24 },
  avatarContainer: { position: 'relative', marginBottom: 16 },
  avatar: { width: 100, height: 100, borderRadius: 50 },
  uploadOverlay: {
    position: 'absolute',
    inset: 0,
    borderRadius: 50,
    backgroundColor: 'rgba(0,0,0,0.5)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  progressText: { color: '#fff', fontWeight: 'bold' },
  editBadge: {
    position: 'absolute',
    bottom: 0,
    right: 0,
    backgroundColor: '#6200EE',
    width: 28,
    height: 28,
    borderRadius: 14,
    alignItems: 'center',
    justifyContent: 'center',
    borderWidth: 2,
    borderColor: '#fff',
  },
  editBadgeText: { fontSize: 14 },
  name: { fontSize: 22, fontWeight: 'bold' },
  email: { fontSize: 14, color: '#666' },
});
```

---

## Request Cancellation

```typescript
import axios, { CancelTokenSource } from 'axios';
import { useEffect, useRef } from 'react';

// ใช้ AbortController (modern)
function useAbortableFetch() {
  const abortControllerRef = useRef<AbortController | null>(null);

  const fetchData = async (url: string) => {
    // ยกเลิก request ก่อนหน้า
    abortControllerRef.current?.abort();
    abortControllerRef.current = new AbortController();

    try {
      const response = await axios.get(url, {
        signal: abortControllerRef.current.signal,
      });
      return response.data;
    } catch (error) {
      if (axios.isCancel(error)) {
        console.log('Request cancelled');
      }
      throw error;
    }
  };

  useEffect(() => {
    return () => {
      abortControllerRef.current?.abort();
    };
  }, []);

  return { fetchData };
}

// ใน Component
function SearchScreen() {
  const { fetchData } = useAbortableFetch();
  const [results, setResults] = useState([]);

  const handleSearch = async (query: string) => {
    if (!query) return;
    try {
      const data = await fetchData(`/api/search?q=${query}`);
      setResults(data);
    } catch (error) {
      if (!axios.isCancel(error)) {
        console.error(error);
      }
    }
  };

  return (
    <TextInput
      onChangeText={handleSearch}
      placeholder="ค้นหา..."
    />
  );
}
```

---

## Tips and Best Practices

### 1. ใช้ Interceptors แทน try/catch ซ้ำ ๆ

```typescript
// ❌ ไม่ดี - try/catch ทุกที่
const handleLogin = async () => {
  try {
    const res = await apiClient.post('/auth/login', credentials);
    // ...
  } catch (error) {
    if (axios.isAxiosError(error)) {
      // handle error ซ้ำซ้อน
    }
  }
};

// ✅ ดีกว่า - จัดการใน interceptor ครั้งเดียว
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    const message = getErrorMessage(error);
    showToast(message); // global error handling
    return Promise.reject(error);
  }
);
```

### 2. Axios Instance แยกตาม API

```typescript
// Public API (ไม่ต้อง auth)
export const publicApi = axios.create({
  baseURL: 'https://api.example.com/public',
});

// Authenticated API
export const privateApi = axios.create({
  baseURL: 'https://api.example.com/private',
});
// เพิ่ม auth interceptor เฉพาะ privateApi
addAuthInterceptor(privateApi);

// Upload API
export const uploadApi = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 60000, // 60 วินาทีสำหรับ upload
});
```

### 3. Type-safe Responses

```typescript
// กำหนด generic type ให้ทุก API call
interface ApiResponse<T> {
  data: T;
  message: string;
  success: boolean;
}

const getPosts = async (): Promise<Post[]> => {
  const response = await apiClient.get<ApiResponse<Post[]>>('/posts');
  return response.data.data;
};
```

---

## สรุป

Axios เป็น HTTP client ที่ทรงพลัง:

1. **Axios Instance** - ตั้งค่า base URL, headers, timeout
2. **Request Interceptors** - เพิ่ม auth token ก่อน request
3. **Response Interceptors** - handle token refresh, error formatting
4. **Transform** - แปลง request/response data
5. **Error Handling** - จัดการ errors แบบ centralized
6. **Retry Logic** - ลองใหม่เมื่อล้มเหลว

ใน Part 036 จะเรียนเรื่อง **JWT Authentication** สำหรับระบบ login/logout
