# Part 016: Fetch API และการเรียก REST API

## สารบัญ
1. Fetch Basics: GET, POST, PUT, DELETE
2. Headers และ Authentication
3. Error Handling
4. Loading States
5. Workshop: Fetch data จาก JSONPlaceholder

---

## 1. Fetch Basics

Fetch API คือ built-in API ของ JavaScript สำหรับการทำ HTTP requests React Native รองรับ Fetch API โดยตรง

### GET Request

```javascript
// GET - ดึงข้อมูล
const fetchUsers = async () => {
  const response = await fetch('https://jsonplaceholder.typicode.com/users');
  const data = await response.json();
  return data;
};

// พร้อม error handling
const fetchPost = async (id) => {
  try {
    const response = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`);
    
    if (!response.ok) {
      throw new Error(`HTTP Error: ${response.status}`);
    }
    
    const post = await response.json();
    return post;
  } catch (error) {
    console.error('Fetch error:', error);
    throw error;
  }
};
```

### POST Request

```javascript
// POST - สร้างข้อมูลใหม่
const createPost = async (postData) => {
  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(postData),
    });

    if (!response.ok) {
      throw new Error(`HTTP Error: ${response.status}`);
    }

    const newPost = await response.json();
    return newPost;
  } catch (error) {
    console.error('Create error:', error);
    throw error;
  }
};

// ใช้งาน
const post = await createPost({
  title: 'ชื่อโพสต์ใหม่',
  body: 'เนื้อหาโพสต์',
  userId: 1,
});
```

### PUT Request - อัพเดทข้อมูลทั้งหมด

```javascript
const updatePost = async (id, data) => {
  try {
    const response = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data),
    });

    if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
    return await response.json();
  } catch (error) {
    throw error;
  }
};
```

### PATCH Request - อัพเดทข้อมูลบางส่วน

```javascript
const patchPost = async (id, partialData) => {
  try {
    const response = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(partialData),
    });

    if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
    return await response.json();
  } catch (error) {
    throw error;
  }
};

// ใช้งาน - อัพเดทเฉพาะ title
await patchPost(1, { title: 'ชื่อใหม่' });
```

### DELETE Request

```javascript
const deletePost = async (id) => {
  try {
    const response = await fetch(`https://jsonplaceholder.typicode.com/posts/${id}`, {
      method: 'DELETE',
    });

    if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
    return true;
  } catch (error) {
    throw error;
  }
};
```

---

## 2. Headers และ Authentication

### Common Headers

```javascript
// ตัวอย่าง headers ที่ใช้บ่อย
const headers = {
  'Content-Type': 'application/json',          // ประเภท request body
  'Accept': 'application/json',                // ประเภทที่ต้องการรับ
  'Authorization': 'Bearer YOUR_TOKEN',        // Authentication
  'X-API-Key': 'YOUR_API_KEY',                 // API Key
  'Accept-Language': 'th',                      // ภาษา
  'Cache-Control': 'no-cache',                 // Cache control
};

const response = await fetch(url, {
  method: 'GET',
  headers,
});
```

### Bearer Token Authentication

```javascript
import AsyncStorage from '@react-native-async-storage/async-storage';

// Helper function สำหรับ authenticated requests
const authenticatedFetch = async (url, options = {}) => {
  const token = await AsyncStorage.getItem('@auth_token');
  
  const config = {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      ...(token ? { 'Authorization': `Bearer ${token}` } : {}),
      ...options.headers,
    },
  };

  return fetch(url, config);
};

// ใช้งาน
const response = await authenticatedFetch('https://api.example.com/profile');
const profile = await response.json();
```

### API Service Class

```javascript
class ApiService {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    this.token = null;
  }

  setToken(token) {
    this.token = token;
  }

  clearToken() {
    this.token = null;
  }

  getHeaders(customHeaders = {}) {
    return {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      ...(this.token ? { 'Authorization': `Bearer ${this.token}` } : {}),
      ...customHeaders,
    };
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseUrl}${endpoint}`;
    
    const config = {
      headers: this.getHeaders(options.headers),
      ...options,
    };

    if (config.body && typeof config.body === 'object') {
      config.body = JSON.stringify(config.body);
    }

    try {
      const response = await fetch(url, config);
      
      if (response.status === 401) {
        // Token หมดอายุ
        this.clearToken();
        // redirect to login
        throw new Error('UNAUTHORIZED');
      }
      
      if (!response.ok) {
        const error = await response.json().catch(() => ({}));
        throw new Error(error.message || `HTTP Error: ${response.status}`);
      }

      // 204 No Content
      if (response.status === 204) return null;
      
      return response.json();
    } catch (error) {
      if (error.name === 'AbortError') {
        throw new Error('Request cancelled');
      }
      throw error;
    }
  }

  get(endpoint, params) {
    const queryString = params
      ? '?' + new URLSearchParams(params).toString()
      : '';
    return this.request(`${endpoint}${queryString}`);
  }

  post(endpoint, body) {
    return this.request(endpoint, { method: 'POST', body });
  }

  put(endpoint, body) {
    return this.request(endpoint, { method: 'PUT', body });
  }

  patch(endpoint, body) {
    return this.request(endpoint, { method: 'PATCH', body });
  }

  delete(endpoint) {
    return this.request(endpoint, { method: 'DELETE' });
  }
}

// Singleton instance
export const api = new ApiService('https://jsonplaceholder.typicode.com');

// ใช้งาน
const users = await api.get('/users');
const post = await api.post('/posts', { title: 'ใหม่', body: 'เนื้อหา', userId: 1 });
```

---

## 3. Error Handling

### ประเภทของ Error

```javascript
// 1. Network Error - ไม่มีอินเทอร์เน็ต
try {
  const response = await fetch('https://api.example.com/data');
} catch (error) {
  if (error.message === 'Network request failed') {
    Alert.alert('ข้อผิดพลาด', 'ไม่มีการเชื่อมต่ออินเทอร์เน็ต');
  }
}

// 2. HTTP Error Codes
const handleHttpError = (status) => {
  switch (status) {
    case 400: return 'ข้อมูลไม่ถูกต้อง';
    case 401: return 'กรุณาเข้าสู่ระบบ';
    case 403: return 'ไม่มีสิทธิ์เข้าถึง';
    case 404: return 'ไม่พบข้อมูล';
    case 422: return 'ข้อมูลไม่ผ่านการตรวจสอบ';
    case 429: return 'คำขอมากเกินไป กรุณาลองใหม่';
    case 500: return 'เกิดข้อผิดพลาดที่เซิร์ฟเวอร์';
    case 503: return 'บริการชั่วคราวไม่พร้อมใช้งาน';
    default: return `เกิดข้อผิดพลาด (${status})`;
  }
};
```

### Comprehensive Error Handler

```javascript
class ApiError extends Error {
  constructor(message, status, data) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.data = data;
  }
}

const safeFetch = async (url, options = {}) => {
  try {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 10000); // 10s timeout

    const response = await fetch(url, {
      ...options,
      signal: controller.signal,
    });
    
    clearTimeout(timeoutId);

    const contentType = response.headers.get('content-type');
    const isJson = contentType?.includes('application/json');
    
    if (!response.ok) {
      const errorData = isJson ? await response.json() : await response.text();
      throw new ApiError(
        errorData.message || handleHttpError(response.status),
        response.status,
        errorData
      );
    }

    if (response.status === 204) return null;
    
    return isJson ? response.json() : response.text();
    
  } catch (error) {
    if (error.name === 'AbortError') {
      throw new Error('คำขอใช้เวลานานเกินไป กรุณาลองใหม่');
    }
    if (error.message === 'Network request failed') {
      throw new Error('ไม่สามารถเชื่อมต่ออินเทอร์เน็ตได้');
    }
    throw error;
  }
};
```

---

## 4. Loading States

### Basic Loading State

```javascript
import React, { useState, useEffect } from 'react';
import { View, Text, ActivityIndicator, FlatList, StyleSheet } from 'react-native';

function UserListScreen() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [refreshing, setRefreshing] = useState(false);

  const fetchUsers = async (isRefreshing = false) => {
    if (!isRefreshing) setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/users');
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const data = await response.json();
      setUsers(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
      setRefreshing(false);
    }
  };

  useEffect(() => {
    fetchUsers();
  }, []);

  if (loading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
        <Text style={styles.loadingText}>กำลังโหลดข้อมูล...</Text>
      </View>
    );
  }

  if (error) {
    return (
      <View style={styles.center}>
        <Text style={styles.errorIcon}>⚠️</Text>
        <Text style={styles.errorText}>{error}</Text>
        <TouchableOpacity style={styles.retryBtn} onPress={() => fetchUsers()}>
          <Text style={styles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <FlatList
      data={users}
      keyExtractor={item => item.id.toString()}
      renderItem={({ item }) => (
        <View style={styles.userCard}>
          <Text style={styles.userName}>{item.name}</Text>
          <Text style={styles.userEmail}>{item.email}</Text>
        </View>
      )}
      refreshing={refreshing}
      onRefresh={() => {
        setRefreshing(true);
        fetchUsers(true);
      }}
    />
  );
}
```

### Skeleton Loading

```javascript
import { View, StyleSheet, Animated } from 'react-native';
import { useEffect, useRef } from 'react';

function SkeletonItem() {
  const animatedValue = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.loop(
      Animated.sequence([
        Animated.timing(animatedValue, {
          toValue: 1,
          duration: 1000,
          useNativeDriver: true,
        }),
        Animated.timing(animatedValue, {
          toValue: 0,
          duration: 1000,
          useNativeDriver: true,
        }),
      ])
    ).start();
  }, []);

  const opacity = animatedValue.interpolate({
    inputRange: [0, 1],
    outputRange: [0.3, 0.7],
  });

  return (
    <Animated.View style={[styles.skeleton, { opacity }]}>
      <View style={styles.skeletonAvatar} />
      <View style={styles.skeletonContent}>
        <View style={styles.skeletonLine} />
        <View style={[styles.skeletonLine, { width: '60%' }]} />
      </View>
    </Animated.View>
  );
}

function SkeletonLoader() {
  return (
    <View>
      {[1, 2, 3, 4, 5].map(i => <SkeletonItem key={i} />)}
    </View>
  );
}

const styles = StyleSheet.create({
  skeleton: {
    flexDirection: 'row',
    padding: 16,
    marginBottom: 8,
    backgroundColor: '#FFF',
  },
  skeletonAvatar: {
    width: 50,
    height: 50,
    borderRadius: 25,
    backgroundColor: '#E0E0E0',
  },
  skeletonContent: {
    flex: 1,
    marginLeft: 12,
    justifyContent: 'center',
  },
  skeletonLine: {
    height: 14,
    backgroundColor: '#E0E0E0',
    borderRadius: 7,
    marginBottom: 8,
    width: '90%',
  },
});
```

---

## Workshop: Fetch data จาก JSONPlaceholder

### โครงสร้างแอป

```
JSONApp/
├── App.js
├── services/
│   └── api.js
├── screens/
│   ├── PostsScreen.js
│   ├── PostDetailScreen.js
│   ├── UsersScreen.js
│   └── UserDetailScreen.js
└── components/
    ├── PostCard.js
    ├── LoadingView.js
    └── ErrorView.js
```

### API Service

```javascript
// services/api.js
const BASE_URL = 'https://jsonplaceholder.typicode.com';

const request = async (endpoint, options = {}) => {
  try {
    const response = await fetch(`${BASE_URL}${endpoint}`, {
      headers: { 'Content-Type': 'application/json' },
      ...options,
    });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    if (response.status === 204) return null;
    return response.json();
  } catch (error) {
    throw error;
  }
};

export const PostsAPI = {
  getAll: (params = {}) => {
    const query = new URLSearchParams(params).toString();
    return request(`/posts${query ? '?' + query : ''}`);
  },
  getById: (id) => request(`/posts/${id}`),
  getComments: (postId) => request(`/posts/${postId}/comments`),
  create: (data) => request('/posts', { method: 'POST', body: JSON.stringify(data) }),
  update: (id, data) => request(`/posts/${id}`, { method: 'PUT', body: JSON.stringify(data) }),
  delete: (id) => request(`/posts/${id}`, { method: 'DELETE' }),
};

export const UsersAPI = {
  getAll: () => request('/users'),
  getById: (id) => request(`/users/${id}`),
  getPosts: (userId) => request(`/posts?userId=${userId}`),
};
```

### PostsScreen

```javascript
// screens/PostsScreen.js
import React, { useState, useEffect, useCallback } from 'react';
import {
  View, Text, FlatList, StyleSheet,
  TouchableOpacity, RefreshControl, TextInput, Alert
} from 'react-native';
import { PostsAPI } from '../services/api';

function PostCard({ post, onPress, onDelete }) {
  return (
    <TouchableOpacity style={styles.card} onPress={() => onPress(post)}>
      <View style={styles.cardHeader}>
        <View style={styles.userBadge}>
          <Text style={styles.userBadgeText}>U{post.userId}</Text>
        </View>
        <Text style={styles.postId}>#{post.id}</Text>
      </View>
      <Text style={styles.title} numberOfLines={2}>{post.title}</Text>
      <Text style={styles.body} numberOfLines={2}>{post.body}</Text>
      <TouchableOpacity
        style={styles.deleteBtn}
        onPress={() => onDelete(post.id)}
      >
        <Text style={styles.deleteBtnText}>ลบ</Text>
      </TouchableOpacity>
    </TouchableOpacity>
  );
}

export default function PostsScreen({ navigation }) {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [refreshing, setRefreshing] = useState(false);
  const [error, setError] = useState(null);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loadingMore, setLoadingMore] = useState(false);

  const LIMIT = 10;

  const loadPosts = useCallback(async (pageNum = 1, isRefresh = false) => {
    if (pageNum === 1) {
      isRefresh ? setRefreshing(true) : setLoading(true);
    } else {
      setLoadingMore(true);
    }
    setError(null);

    try {
      const data = await PostsAPI.getAll({
        _page: pageNum,
        _limit: LIMIT,
      });

      if (pageNum === 1) {
        setPosts(data);
      } else {
        setPosts(prev => [...prev, ...data]);
      }

      setHasMore(data.length === LIMIT);
      setPage(pageNum);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
      setRefreshing(false);
      setLoadingMore(false);
    }
  }, []);

  useEffect(() => {
    loadPosts(1);
  }, [loadPosts]);

  const handleDelete = (postId) => {
    Alert.alert('ยืนยันการลบ', 'คุณต้องการลบโพสต์นี้หรือไม่?', [
      { text: 'ยกเลิก', style: 'cancel' },
      {
        text: 'ลบ',
        style: 'destructive',
        onPress: async () => {
          try {
            await PostsAPI.delete(postId);
            setPosts(prev => prev.filter(p => p.id !== postId));
          } catch (err) {
            Alert.alert('ข้อผิดพลาด', 'ไม่สามารถลบได้');
          }
        },
      },
    ]);
  };

  const handleLoadMore = () => {
    if (!loadingMore && hasMore) {
      loadPosts(page + 1);
    }
  };

  if (loading) {
    return (
      <View style={styles.center}>
        <Text>กำลังโหลด...</Text>
      </View>
    );
  }

  if (error) {
    return (
      <View style={styles.center}>
        <Text style={styles.errorText}>เกิดข้อผิดพลาด: {error}</Text>
        <TouchableOpacity onPress={() => loadPosts(1)}>
          <Text style={styles.retryText}>ลองใหม่</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <FlatList
      data={posts}
      keyExtractor={item => item.id.toString()}
      renderItem={({ item }) => (
        <PostCard
          post={item}
          onPress={(post) => navigation.navigate('PostDetail', { postId: post.id })}
          onDelete={handleDelete}
        />
      )}
      refreshControl={
        <RefreshControl
          refreshing={refreshing}
          onRefresh={() => loadPosts(1, true)}
          colors={['#6200EE']}
        />
      }
      onEndReached={handleLoadMore}
      onEndReachedThreshold={0.3}
      ListFooterComponent={() =>
        loadingMore ? <Text style={styles.loadingMore}>กำลังโหลดเพิ่ม...</Text> : null
      }
      contentContainerStyle={styles.list}
    />
  );
}

const styles = StyleSheet.create({
  center: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 20 },
  list: { padding: 12 },
  card: {
    backgroundColor: '#FFF',
    borderRadius: 12,
    padding: 14,
    marginBottom: 10,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 3,
    elevation: 2,
  },
  cardHeader: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 8 },
  userBadge: {
    backgroundColor: '#EDE7F6',
    paddingHorizontal: 8,
    paddingVertical: 2,
    borderRadius: 10,
  },
  userBadgeText: { fontSize: 12, color: '#6200EE', fontWeight: '600' },
  postId: { fontSize: 12, color: '#999' },
  title: { fontSize: 15, fontWeight: 'bold', color: '#212121', marginBottom: 4 },
  body: { fontSize: 13, color: '#666', lineHeight: 18 },
  deleteBtn: { alignSelf: 'flex-end', marginTop: 8 },
  deleteBtnText: { color: '#E53935', fontSize: 13 },
  errorText: { color: '#E53935', marginBottom: 12, textAlign: 'center' },
  retryText: {
    color: '#6200EE',
    fontSize: 15,
    padding: 10,
    borderWidth: 1,
    borderColor: '#6200EE',
    borderRadius: 8,
  },
  loadingMore: { textAlign: 'center', padding: 12, color: '#999' },
});
```

### PostDetailScreen

```javascript
// screens/PostDetailScreen.js
import React, { useState, useEffect } from 'react';
import {
  View, Text, StyleSheet, ScrollView,
  ActivityIndicator, TextInput, TouchableOpacity, Alert
} from 'react-native';
import { PostsAPI } from '../services/api';

export default function PostDetailScreen({ route, navigation }) {
  const { postId } = route.params;
  const [post, setPost] = useState(null);
  const [comments, setComments] = useState([]);
  const [loading, setLoading] = useState(true);
  const [isEditing, setIsEditing] = useState(false);
  const [editTitle, setEditTitle] = useState('');
  const [saving, setSaving] = useState(false);

  useEffect(() => {
    loadData();
  }, [postId]);

  const loadData = async () => {
    setLoading(true);
    try {
      const [postData, commentsData] = await Promise.all([
        PostsAPI.getById(postId),
        PostsAPI.getComments(postId),
      ]);
      setPost(postData);
      setComments(commentsData);
      setEditTitle(postData.title);
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', error.message);
    } finally {
      setLoading(false);
    }
  };

  const handleSaveEdit = async () => {
    setSaving(true);
    try {
      const updated = await PostsAPI.update(postId, {
        ...post,
        title: editTitle,
      });
      setPost(updated);
      setIsEditing(false);
    } catch (error) {
      Alert.alert('ข้อผิดพลาด', 'ไม่สามารถบันทึกได้');
    } finally {
      setSaving(false);
    }
  };

  if (loading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    );
  }

  if (!post) return null;

  return (
    <ScrollView style={styles.container}>
      {/* Post */}
      <View style={styles.postSection}>
        {isEditing ? (
          <View>
            <TextInput
              style={styles.editInput}
              value={editTitle}
              onChangeText={setEditTitle}
              multiline
            />
            <View style={styles.editActions}>
              <TouchableOpacity
                style={styles.cancelBtn}
                onPress={() => {
                  setIsEditing(false);
                  setEditTitle(post.title);
                }}
              >
                <Text style={styles.cancelBtnText}>ยกเลิก</Text>
              </TouchableOpacity>
              <TouchableOpacity
                style={styles.saveBtn}
                onPress={handleSaveEdit}
                disabled={saving}
              >
                <Text style={styles.saveBtnText}>
                  {saving ? 'กำลังบันทึก...' : 'บันทึก'}
                </Text>
              </TouchableOpacity>
            </View>
          </View>
        ) : (
          <View>
            <View style={styles.postHeader}>
              <Text style={styles.postTitle}>{post.title}</Text>
              <TouchableOpacity onPress={() => setIsEditing(true)}>
                <Text style={styles.editBtn}>แก้ไข</Text>
              </TouchableOpacity>
            </View>
            <Text style={styles.postBody}>{post.body}</Text>
          </View>
        )}
      </View>

      {/* Comments */}
      <View style={styles.commentsSection}>
        <Text style={styles.commentsTitle}>
          ความคิดเห็น ({comments.length})
        </Text>
        {comments.map(comment => (
          <View key={comment.id} style={styles.comment}>
            <Text style={styles.commentName}>{comment.name}</Text>
            <Text style={styles.commentEmail}>{comment.email}</Text>
            <Text style={styles.commentBody}>{comment.body}</Text>
          </View>
        ))}
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  postSection: {
    backgroundColor: '#FFF',
    padding: 16,
    marginBottom: 8,
  },
  postHeader: { flexDirection: 'row', justifyContent: 'space-between', marginBottom: 8 },
  postTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#212121',
    flex: 1,
    marginRight: 8,
  },
  postBody: { fontSize: 15, color: '#555', lineHeight: 22 },
  editBtn: { color: '#6200EE', fontSize: 14 },
  editInput: {
    borderWidth: 1,
    borderColor: '#6200EE',
    borderRadius: 8,
    padding: 10,
    fontSize: 16,
    marginBottom: 10,
  },
  editActions: { flexDirection: 'row', justifyContent: 'flex-end', gap: 10 },
  cancelBtn: { padding: 10 },
  cancelBtnText: { color: '#999' },
  saveBtn: {
    backgroundColor: '#6200EE',
    paddingHorizontal: 16,
    paddingVertical: 10,
    borderRadius: 8,
  },
  saveBtnText: { color: '#FFF', fontWeight: '600' },
  commentsSection: { backgroundColor: '#FFF', padding: 16 },
  commentsTitle: { fontSize: 16, fontWeight: 'bold', marginBottom: 12 },
  comment: {
    borderLeftWidth: 3,
    borderLeftColor: '#6200EE',
    paddingLeft: 12,
    marginBottom: 16,
  },
  commentName: { fontSize: 14, fontWeight: '600', color: '#212121' },
  commentEmail: { fontSize: 12, color: '#6200EE', marginBottom: 4 },
  commentBody: { fontSize: 13, color: '#555', lineHeight: 18 },
});
```

---

## Tips และ Best Practices

```
✅ DO:
- ใช้ AbortController สำหรับ cancel requests
- Handle ทุก HTTP status codes
- แสดง loading state เสมอ
- ใช้ Promise.all สำหรับ parallel requests
- สร้าง API service layer แยกออกมา

❌ DON'T:
- ไม่เรียก fetch ใน render function โดยตรง
- ไม่ลืม cancel request เมื่อ component unmount
- ไม่ ignore error handling
- ไม่ hardcode URL ใน component

🚀 สำหรับ Production:
- ใช้ axios หรือ react-query
- Implement retry logic
- Add request/response interceptors
- Cache responses
```

---

## สรุป

Fetch API เป็นพื้นฐานของการสื่อสารกับ backend:

1. **HTTP Methods** - GET, POST, PUT, PATCH, DELETE
2. **Headers & Auth** - Bearer token, API keys
3. **Error Handling** - HTTP errors, network errors
4. **Loading States** - loading, skeleton, refreshing, pagination
5. **Workshop** - Complete CRUD app กับ JSONPlaceholder
